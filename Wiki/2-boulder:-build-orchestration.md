# Boulder: Build Orchestration

Relevant source files

* [source/boulder/cli/deletecache\_command.d](../source/boulder/cli/deletecache_command.d)
* [source/boulder/cli/package.d](../source/boulder/cli/package.d)
* [source/boulder/controller.d](../source/boulder/controller.d)
* [source/boulder/main.d](../source/boulder/main.d)
* [source/boulder/stages/clean\_root.d](../source/boulder/stages/clean_root.d)
* [source/boulder/stages/package.d](../source/boulder/stages/package.d)

Boulder is the top-level orchestration component in the boulder-d-legacy/ system that manages the complete package build lifecycle. It coordinates all phases of building a package from a `stone.yml` recipe file, including source fetching, dependency resolution, environment setup, and invocation of the mason build system. Boulder operates through a sequential pipeline of stages, each responsible for a specific aspect of the build preparation and execution.

For information about the actual compilation and package creation process, see [Mason: Package Builder](3-mason:-package-builder). For recipe file format details, see [Stone.yml Recipe Format](6-stone.yml-recipe-format). For the underlying configuration system, see [Configuration System](5-configuration-system).

## Controller: The Main Orchestrator

The `Controller` class serves as the primary entry point for all build operations. It implements the `StageContext` interface and coordinates the execution of build stages.

**Controller Responsibilities**

```mermaid
flowchart TD

Controller["Controller<br>(StageContext impl)"]
Parse["parseRecipe()<br>Loads stone.yml"]
Config["ProfileConfiguration<br>Loads build profile"]
UC["UpstreamCache<br>Source management"]
Fetcher["FetchController<br>Download orchestration"]
Stages["iterateStages()<br>Execute stage pipeline"]
BuildJob["BuildJob<br>Build context"]
Profile["Profile object<br>Repository collections"]
Success["ExitStatus.Success"]
Failure["ExitStatus.Failure"]
Arch["architecture<br>Target platform"]
Confine["confinement<br>Container usage"]
CompCache["compilerCache<br>ccache enable"]
OutDir["outputDirectory<br>Artifacts location"]

Controller --> Parse
Controller --> Config
Controller --> UC
Controller --> Fetcher
Controller --> Stages
Parse --> BuildJob
Config --> Profile
Stages --> Success
Stages --> Failure
Controller --> Arch
Controller --> Confine
Controller --> CompCache
Controller --> OutDir

subgraph subGraph0 ["Configuration Properties"]
    Arch
    Confine
    CompCache
    OutDir
end
```

Sources: [source/boulder/controller.d44-125](../source/boulder/controller.d#L44-L125) [source/boulder/stages/package.d58-121](../source/boulder/stages/package.d#L58-L121)

**Key Methods**

| Method | Purpose | Returns |
| --- | --- | --- |
| `parseRecipe(string)` | Parse stone.yml file into Spec | `BuildJob` |
| `build(string)` | Execute full build pipeline | `ExitStatus` |
| `chroot(string)` | Enter interactive build environment | `ExitStatus` |
| `iterateStages(Stage[])` | Execute sequence of stages | `StageReturn` |

The Controller manages critical resources including paths to `moss` and `moss-container` binaries, the upstream source cache, and the fetch controller for parallel downloads. It also tracks mount points to ensure proper cleanup on both successful and failed builds.

Sources: [source/boulder/controller.d198-313](../source/boulder/controller.d#L198-L313) [source/boulder/controller.d321-347](../source/boulder/controller.d#L321-L347) [source/boulder/controller.d355-418](../source/boulder/controller.d#L355-L418)

## Build Lifecycle Pipeline

Boulder executes builds through a fixed sequence of stages defined in `buildStages`. Each stage performs a specific operation and returns a status indicating success, failure, or that it was skipped.

**Sequential Stage Execution**

```mermaid
flowchart TD

Start["build called"]
Parse["parseRecipe()<br>line 227"]
Autodeps["Extract auto-dependencies<br>lines 237-304"]
Iter["iterateStages(buildStages)<br>line 307"]
S1["stageCleanRoot<br>Remove existing build dirs"]
S2["stageCreateRoot<br>Create directory structure"]
S3["stageFetchUpstreams<br>Download sources"]
S4["stageConfigureRoot<br>Add repositories"]
S5["stagePopulateRoot<br>Install dependencies"]
S6["stageShareUpstreams<br>Link sources into buildroot"]
S7["stageBuildPackage<br>Invoke mason"]
S8["stageSyncArtefacts<br>Copy outputs"]
Success["ExitStatus.Success"]
Fail["ExitStatus.Failure"]

Start --> Parse
Parse --> Autodeps
Autodeps --> Iter
Iter --> S1
S1 --> S2
S2 --> S3
S3 --> S4
S4 --> S5
S5 --> S6
S6 --> S7
S7 --> S8
S8 --> Success
S1 --> Fail
S2 --> Fail
S3 --> Fail
S4 --> Fail
S5 --> Fail
S6 --> Fail
S7 --> Fail
S8 --> Fail
S1 --> S2
S3 --> S4
```

Sources: [source/boulder/stages/package.d41-45](../source/boulder/stages/package.d#L41-L45) [source/boulder/controller.d225-313](../source/boulder/controller.d#L225-L313)

**Stage Definitions**

The `buildStages` array defines the complete pipeline:

```
static auto buildStages = [
    &stageCleanRoot,        // Clean previous artifacts
    &stageCreateRoot,       // Create build directory tree
    &stageFetchUpstreams,   // Download and validate sources
    &stageConfigureRoot,    // Configure moss repositories
    &stagePopulateRoot,     // Install build dependencies
    &stageShareUpstreams,   // Hardlink sources into buildroot
    &stageBuildPackage,     // Execute mason build
    &stageSyncArtefacts,    // Copy .stone packages to output
];
```

Each stage is represented by a `Stage` struct containing a name and a `StageFunction` functor. For details on individual stages, see [Build Stages](2.3-build-stages).

Sources: [source/boulder/stages/package.d41-52](../source/boulder/stages/package.d#L41-L52)

## Stage Execution System

**Stage Interface Definition**

```mermaid
classDiagram
    class StageContext {
        «interface»
        +job() : BuildJob
        +outputDirectory() : string
        +architecture() : string
        +confinement() : bool
        +compilerCache() : bool
        +mossBinary() : string
        +containerBinary() : string
        +upstreamCache() : UpstreamCache
        +fetcher() : FetchContext
        +profile() : Profile
        +addMount(Mount) : void
    }
    class Stage {
        +name: string
        +functor: StageFunction
    }
    class StageFunction {
        «typedef»
        StageReturn function(StageContext)
    }
    class StageReturn {
        «enumeration»
        Success
        Failure
        Skipped
    }
    class Controller {
        -_job: BuildJob
        -_upstreamCache: UpstreamCache
        -_fetcher: FetchController
        -profileObj: Profile
        -mountPoints: Mount[]
        +iterateStages(Stage[])
        +parseRecipe(string)
        +build(string)
        +chroot(string)
    }
    StageContext <|.. Controller : implements
    Stage --> StageFunction : contains
    StageFunction --> StageReturn : returns
    StageFunction --> StageContext : receives
```

Sources: [source/boulder/stages/package.d58-160](../source/boulder/stages/package.d#L58-L160) [source/boulder/controller.d44-512](../source/boulder/controller.d#L44-L512)

**Stage Execution Flow**

The `iterateStages()` method in Controller handles the sequential execution logic:

1. **Iteration**: Loop through stages array maintaining an index
2. **Invocation**: Call each stage's functor with the Controller as context
3. **Error Handling**: Catch exceptions and convert to `StageReturn.Failure`
4. **Flow Control**: Advance index on Success/Skipped, break loop on Failure
5. **Cleanup**: Unmount all tracked mount points in reverse order (scope exit)

The mount tracking system ensures that any filesystems mounted during stages (e.g., by populate\_root or build\_package) are properly unmounted even if a later stage fails.

Sources: [source/boulder/controller.d355-418](../source/boulder/controller.d#L355-L418) [source/boulder/controller.d420-426](../source/boulder/controller.d#L420-L426)

## Dependency Auto-Detection

Before executing stages, the Controller performs automatic dependency detection to ensure all required tools are available in the build environment.

**Auto-Dependency Sources**

```mermaid
flowchart TD

Macros["Macro expansion<br>lines 238-250"]
Scripts["Build scripts<br>setup/build/check/install"]
Upstreams["recipe.upstreams<br>lines 255-294"]
PlainFiles["Plain archives<br>.tar.xz, .zip, etc"]
GitRepos["Git repositories<br>line 296-299"]
ExtraDeps["_job.extraDeps<br>string array"]
UpstreamDeps["upstreamDeps<br>string array"]
Merge["Merge + Sort + Uniq<br>lines 302-304"]
FinalDeps["Final dependency list"]

Macros --> ExtraDeps
Scripts --> ExtraDeps
Upstreams --> UpstreamDeps
PlainFiles --> UpstreamDeps
GitRepos --> UpstreamDeps
ExtraDeps --> Merge
UpstreamDeps --> Merge
Merge --> FinalDeps

subgraph subGraph1 ["Source Analysis"]
    Upstreams
    PlainFiles
    GitRepos
end

subgraph subGraph0 ["Recipe Analysis"]
    Macros
    Scripts
end
```

**Archive Format Detection**

Boulder automatically adds extraction tool dependencies based on upstream file extensions:

| Extension | Required Dependencies |
| --- | --- |
| `.xz` | `binary(tar)`, `binary(xz)` |
| `.zst` | `binary(tar)`, `binary(zstd)` |
| `.bz2` | `binary(tar)`, `binary(bzip2)` |
| `.gz` | `binary(tar)`, `binary(gzip)` |
| `.zip` | `binary(unzip)` |
| `.rpm` | `binary(rpm2cpio)`, `binary(cpio)` |
| `.deb` | `binary(ar)` |

Git sources automatically add `binary(git)` to the dependency list.

Sources: [source/boulder/controller.d237-304](../source/boulder/controller.d#L237-L304)

## Profile and Configuration Integration

Boulder uses the `ProfileConfiguration` system to manage repository collections and build settings.

**Profile Loading Sequence**

```mermaid
sequenceDiagram
  participant Controller
  participant ProfileConfiguration
  participant Profile
  participant Collections
  participant FileSystem

  Controller->>ProfileConfiguration: new ProfileConfiguration()
  Controller->>ProfileConfiguration: load(configDir)
  ProfileConfiguration->>FileSystem: Read config files
  FileSystem-->>ProfileConfiguration: Configuration data
  Controller->>ProfileConfiguration: Find profile by ID
  ProfileConfiguration-->>Controller: Profile object
  Controller->>Profile: Access collections
  Profile-->>Controller: Collection[] array
  loop [For each collection]
    Controller->>FileSystem: Check file:// URIs exist
    FileSystem-->>Controller: Validation result
  end
  note over Controller: Profile stored in
```

The default profile is `"default-x86_64"` but can be overridden via the `-p` CLI flag. Profiles contain repository collections that specify where to fetch build dependencies. For more details on profile configuration, see [Build Profiles and Repository Configuration](5.5-build-profiles-and-repository-configuration).

Sources: [source/boulder/controller.d69-99](../source/boulder/controller.d#L69-L99) [source/boulder/cli/package.d41](../source/boulder/cli/package.d#L41-L41)

## Integration with moss Ecosystem

Boulder tightly integrates with the moss package management system through both binary invocation and library linkage.

**Binary Integration Points**

```mermaid
flowchart TD

Controller["Controller"]
Moss["moss binary<br>mossBinary property"]
MossOps["Operations:<br>- Add repositories<br>- Install packages<br>- Create indices"]
Container["moss-container binary<br>containerBinary property"]
ContainerOps["Operations:<br>- Create isolated build env<br>- Execute mason in container"]
ConfigStage["stageConfigureRoot<br>Adds repos via moss"]
PopulateStage["stagePopulateRoot<br>Installs deps via moss"]
BuildStage["stageBuildPackage<br>Runs moss-container"]
Confined["confinement<br>enabled?"]
DirectExec["Direct mason execution"]

Controller --> Moss
Controller --> Container
ConfigStage --> Moss
PopulateStage --> Moss
BuildStage --> Container
BuildStage --> Confined
Confined --> Container

subgraph subGraph3 ["Confinement Check"]
    Confined
    DirectExec
    Confined --> DirectExec
end

subgraph subGraph2 ["Build Stages"]
    ConfigStage
    PopulateStage
    BuildStage
end

subgraph subGraph1 ["moss-container Binary Usage"]
    Container
    ContainerOps
    Container --> ContainerOps
end

subgraph subGraph0 ["moss Binary Usage"]
    Moss
    MossOps
    Moss --> MossOps
end
```

The Controller locates moss binaries relative to its own installation path. For confined builds (the default), both binaries must be present. For unconfined builds (via `-u` flag), only the moss binary is required.

Sources: [source/boulder/controller.d62-119](../source/boulder/controller.d#L62-L119)

**Library Integration**

Boulder links against multiple libmoss libraries for core functionality:

* **moss.core**: Utility functions (ExitStatus, computeSHA256, mounts)
* **moss.fetcher**: Download orchestration (FetchController, Fetchable)
* **moss.format.source**: Recipe parsing (Spec, UpstreamType)
* **moss.config**: Profile and repository configuration

For details on how boulder uses libmoss for dependency analysis, see [Dependency Resolution and Analysis](7.2-dependency-resolution-and-analysis).

Sources: [source/boulder/controller.d24-38](../source/boulder/controller.d#L24-L38)

## Fetch System and Upstream Cache

The Controller manages source downloads through the `FetchController` and validates/caches them via `UpstreamCache`.

**Download and Validation Flow**

```mermaid
sequenceDiagram
  participant stageFetchUpstreams
  participant FetchController
  participant Controller
  participant UpstreamCache
  participant Disk

  stageFetchUpstreams->>FetchController: Enqueue Fetchable objects
  FetchController->>Disk: Download to staging
  loop [Hash matches]
    FetchController->>Controller: onFetchComplete(f, 200)
    Controller->>Controller: fetchableToUpstream(f)
    Controller->>Disk: computeSHA256(file)
    Disk-->>Controller: actualHash
    Controller->>UpstreamCache: promote(upstream)
    UpstreamCache->>Disk: Move to fetched/
    Controller->>Controller: onFetchFail(f, msg)
    Controller->>Controller: Set failFlag = true
    Controller->>UpstreamCache: promote(upstream)
    UpstreamCache->>Disk: Move to git/
    FetchController->>Controller: onFetchFail(f, msg)
    Controller->>Controller: Set failFlag = true
    Controller->>FetchController: clear()
  end
```

The `onFetchComplete` callback validates checksums for plain archives before promoting them from staging to the permanent cache. Git repositories are validated by verifying the requested ref exists.

Sources: [source/boulder/controller.d121-124](../source/boulder/controller.d#L121-L124) [source/boulder/controller.d438-494](../source/boulder/controller.d#L438-L494)

**Upstream Matching**

The `fetchableToUpstream()` helper method matches completed downloads to their upstream definitions in the recipe:

* **Plain upstreams**: Matched by comparing the expected SHA256 hash with the download's filename (which is the hash)
* **Git upstreams**: Matched by comparing the URI basename

This mapping is necessary because the fetch system uses generic `Fetchable` objects while the recipe defines typed `UpstreamDefinition` objects.

Sources: [source/boulder/controller.d481-494](../source/boulder/controller.d#L481-L494)

## Error Handling and Cleanup

Boulder implements comprehensive error handling to ensure proper cleanup regardless of build outcome.

**Error Propagation**

```mermaid
flowchart TD

StageExec["Stage execution"]
TryCatch["Try/Catch"]
LogError["error(exception.message)"]
CheckReturn["StageReturn?"]
SetFailure["result = Failure"]
LogSuccess["info('Stage success')"]
LogFailure["error('Stage failure')"]
LogSkipped["trace('Stage skipped')"]
CheckFlag["failFlag set?"]
ForceFailure["Override to Failure"]
UseResult["Use stage result"]
FlowControl["Result type"]
Break["break build_loop"]
Continue["++stageIndex"]
Cleanup["scope(exit) cleanup"]
NextStage["Next stage"]
UnmountAll["foreach_reverse mountPoints"]
Return["return result"]

StageExec --> TryCatch
TryCatch --> LogError
TryCatch --> CheckReturn
LogError --> SetFailure
CheckReturn --> LogSuccess
CheckReturn --> LogFailure
CheckReturn --> LogSkipped
SetFailure --> CheckFlag
LogSuccess --> CheckFlag
LogFailure --> CheckFlag
LogSkipped --> CheckFlag
CheckFlag --> ForceFailure
CheckFlag --> UseResult
ForceFailure --> FlowControl
UseResult --> FlowControl
FlowControl --> Break
FlowControl --> Continue
Break --> Cleanup
Continue --> NextStage
NextStage --> StageExec
Cleanup --> UnmountAll
UnmountAll --> Return
```

The `failFlag` is set by download failures and forces all subsequent stages to fail immediately, preventing execution with incomplete sources.

Sources: [source/boulder/controller.d372-402](../source/boulder/controller.d#L372-L402) [source/boulder/controller.d403-416](../source/boulder/controller.d#L403-L416)

**Mount Point Cleanup**

The Controller tracks all mount points created during build stages and unmounts them in reverse order using a `scope(exit)` block. This ensures cleanup occurs even when exceptions are thrown:

* Unmount flags: `UnmountFlags.Force | UnmountFlags.Detach`
* Order: Reverse order of mount creation (LIFO)
* Error handling: Log failures but continue unmounting remaining mounts

Sources: [source/boulder/controller.d403-416](../source/boulder/controller.d#L403-L416) [source/boulder/controller.d420-426](../source/boulder/controller.d#L420-L426)

## Chroot Command Support

In addition to building packages, Boulder supports entering an interactive chroot environment for debugging and development.

**Chroot vs Build Stages**

```mermaid
flowchart TD

B1["stageCleanRoot"]
B2["stageCreateRoot"]
B3["stageFetchUpstreams"]
B4["stageConfigureRoot"]
B5["stagePopulateRoot"]
B6["stageShareUpstreams"]
B7["stageBuildPackage"]
B8["stageSyncArtefacts"]
C1["stageCreateRoot"]
C2["stagePopulateRoot"]
C3["stageChrootPackage"]
Build["boulder build"]
Chroot["boulder chroot"]

Build --> B1
Chroot --> C1

subgraph subGraph1 ["Chroot Pipeline"]
    C1
    C2
    C3
    C1 --> C2
    C2 --> C3
end

subgraph subGraph0 ["Build Pipeline"]
    B1
    B2
    B3
    B4
    B5
    B6
    B7
    B8
    B1 --> B2
    B2 --> B3
    B3 --> B4
    B4 --> B5
    B5 --> B6
    B6 --> B7
    B7 --> B8
end
```

The `chroot()` method uses a reduced stage set defined in `chrootStages`. It reuses an existing build root if available, or creates a new one if needed. The method automatically adds useful interactive tools (`git`, `nano`, `vim`) to the dependency list.

Sources: [source/boulder/controller.d321-347](../source/boulder/controller.d#L321-L347) [source/boulder/stages/package.d50-52](../source/boulder/stages/package.d#L50-L52)

## CLI Command Mapping

**Boulder Subcommand Dispatch**

```mermaid
flowchart TD

Debug["-d, --debug<br>Enable debug logging"]
Profile["-p, --profile<br>Select build profile"]
ConfigDir["-C, --config-directory<br>Config root path"]
CLI["boulder CLI<br>source/boulder/cli"]
Build["build<br>BuildControlCommand"]
Chroot["chroot<br>ChrootCommand"]
New["new<br>NewCommand"]
Delete["delete-cache<br>DeleteCacheCommand"]
Version["version<br>VersionCommand"]
Help["help<br>HelpCommand"]
BuildMethod["Controller.build()"]
ChrootMethod["Controller.chroot()"]
DrafterIntegration["drafter integration"]
CacheManagement["Cache cleanup"]
IterBuildStages["buildStages pipeline"]
IterChrootStages["chrootStages pipeline"]

CLI --> Build
CLI --> Chroot
CLI --> New
CLI --> Delete
CLI --> Version
CLI --> Help
Build --> BuildMethod
Chroot --> ChrootMethod
New --> DrafterIntegration
Delete --> CacheManagement
BuildMethod --> IterBuildStages
ChrootMethod --> IterChrootStages

subgraph subGraph0 ["Global Options"]
    Debug
    Profile
    ConfigDir
end
```

All boulder commands inherit global options defined in the `BoulderCLI` struct. These options are processed before subcommand execution and affect Controller initialization.

Sources: [source/boulder/cli/package.d28-48](../source/boulder/cli/package.d#L28-L48) [source/boulder/main.d32-42](../source/boulder/main.d#L32-L42)