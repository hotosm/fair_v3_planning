# Build System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [Makefile](Makefile)

</details>



## Purpose and Scope

This document provides a technical overview of the Flyte build system, which handles compilation, dependency management, deployment preparation, and development workflow automation. The build system is primarily implemented through a set of Makefiles that coordinate various build tools across multiple languages and components. For information about the CI/CD pipeline that uses this build system, see [CI/CD Pipeline](#7.2).

## Build System Architecture

The Flyte build system is organized as a collection of Makefile targets that coordinate various build processes across the monorepo structure.

```mermaid
graph TD
    subgraph "Main Build System"
        MakeFile["Makefile"] --> |includes| EndToEndTests["boilerplate/flyte/end2end/Makefile"]
        MakeFile --> |includes| GolangTests["boilerplate/flyte/golang_test_targets/Makefile"]
        MakeFile --> BuildTargets["Build Targets"]
    end
    
    subgraph "Build Targets"
        Compilation["Compilation Targets:<br/>compile, linux_compile"]
        ArtifactGeneration["Artifact Generation:<br/>prepare_artifacts, release_automation"]
        DeploymentTools["Deployment Tools:<br/>helm, deploy_sandbox"]
        Documentation["Documentation:<br/>docs, dev-docs"]
        DependencyManagement["Dependency Management:<br/>go-tidy, conda-lock"]
        DevTools["Development Tools:<br/>setup_local_dev"]
    end
    
    BuildTargets --> Compilation
    BuildTargets --> ArtifactGeneration
    BuildTargets --> DeploymentTools
    BuildTargets --> Documentation
    BuildTargets --> DependencyManagement
    BuildTargets --> DevTools
    
    subgraph "Build Inputs"
        GoModules["Go Modules"]
        PythonDeps["Python Dependencies"]
        FlyteConsole["FlyteConsole Frontend"]
        HelmCharts["Helm Charts"]
    end
    
    subgraph "Build Outputs"
        FlyteBinary["Flyte Binary"]
        DockerImages["Docker Images"]
        K8sManifests["Kubernetes Manifests"]
        Docs["Documentation"]
    end
    
    GoModules --> Compilation
    PythonDeps --> DependencyManagement
    FlyteConsole --> Compilation
    HelmCharts --> DeploymentTools
    
    Compilation --> FlyteBinary
    ArtifactGeneration --> DockerImages
    DeploymentTools --> K8sManifests
    Documentation --> Docs
```

Sources: [Makefile:1-152]()

## Core Components

The Flyte build system integrates several key components:

1. **Makefile Targets**: Primary interface for developers to build, test, and deploy Flyte
2. **Versioning System**: Extracts version information from Git
3. **Dependency Management**: Handles Go and Python dependencies
4. **Documentation Generation**: Builds Sphinx documentation
5. **Containerization**: Creates Docker images for Flyte components
6. **Kubernetes Deployment**: Generates Helm charts and manifests

Sources: [Makefile:9-13](), [Makefile:50-62](), [Makefile:74-84]()

## Versioning System

Flyte uses Git tags and commit hashes to generate version information at build time. This information is injected into binaries through linker flags.

```mermaid
graph LR
    subgraph "Version Generation"
        GitTag["Git Tag<br/>(git describe)"] --> GitVersion["GIT_VERSION"]
        GitCommit["Git Commit Hash<br/>(git rev-parse)"] --> GitHash["GIT_HASH"]
        CurrentDate["Current Date"] --> Timestamp["TIMESTAMP"]
    end
    
    subgraph "Binary Compilation"
        GitVersion --> LinkerFlags["LD_FLAGS"]
        GitHash --> LinkerFlags
        Timestamp --> LinkerFlags
        LinkerFlags --> GoBuild["go build -ldflags=$(LD_FLAGS)"]
        GoBuild --> CompiledBinary["Flyte Binary<br/>with Version Info"]
    end
```

The version information is encoded in the binary through linker flags that set values in the `flytestdlib/version` package:

- `Version`: Git tag and commit distance (e.g., "v1.2.3-5-g12345")
- `Build`: Short Git commit hash
- `BuildTime`: Build timestamp in YYYY-MM-DD format

Sources: [Makefile:9-13](), [Makefile:21-23]()

## Build Targets

### Compilation Targets

| Target | Description | Command |
|--------|-------------|---------|
| `compile` | Builds the Flyte binary for host platform | `go build -tags console -ldflags=$(LD_FLAGS) ./cmd/` |
| `linux_compile` | Cross-compiles for Linux | `GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build ...` |
| `cmd/single/dist` | Fetches FlyteConsole frontend assets | `script/get_flyteconsole_dist.sh` |

Sources: [Makefile:16-28]()

### Dependency Management

| Target | Description |
|--------|-------------|
| `go-tidy` | Runs `go mod tidy` on all Go modules in the repository |
| `install-piptools` | Installs Python pip-tools for dependency management |
| `install-conda-lock` | Installs conda-lock for Python environment management |
| `conda-lock` | Generates lockfiles for Python environments |

Sources: [Makefile:50-62](), [Makefile:128-138]()

### Deployment and Release

| Target | Description |
|--------|-------------|
| `helm` | Generates Kubernetes manifests from Helm charts |
| `release_automation` | Prepares release artifacts |
| `deploy_sandbox` | Deploys to the Flyte sandbox environment |
| `helm_update` | Updates Helm chart dependencies |
| `helm_install` | Installs Flyte using Helm |
| `helm_upgrade` | Upgrades an existing Flyte installation |

Sources: [Makefile:34-48](), [Makefile:74-84]()

### Documentation

| Target | Description |
|--------|-------------|
| `docs` | Builds Sphinx documentation |
| `dev-docs` | Builds documentation in a Docker container for local development |

Sources: [Makefile:87-108]()

## Multi-Module Build Process

Flyte's monorepo contains multiple Go modules that must be built and maintained together. The build system provides targets to manage dependencies across these modules.

```mermaid
graph TD
    subgraph "Go Module Structure"
        FlyteCLI["flytectl"]
        FlytePropeller["flytepropeller"]
        FlyteAdmin["flyteadmin"]
        DataCatalog["datacatalog"]
        FlytePlugins["flyteplugins"]
        FlyteCopilot["flytecopilot"]
        FlyteIDL["flyteidl"]
        FlyteStdLib["flytestdlib"]
    end
    
    subgraph "Build Operations"
        GoTidy["go-tidy target"]
        Compile["compile target"]
    end
    
    GoTidy --> FlyteCLI
    GoTidy --> FlytePropeller
    GoTidy --> FlyteAdmin
    GoTidy --> DataCatalog
    GoTidy --> FlytePlugins
    GoTidy --> FlyteCopilot
    GoTidy --> FlyteIDL
    GoTidy --> FlyteStdLib
    
    Compile --> MainBinary["Main Flyte Binary"]
    
    FlyteStdLib --> |dependency| FlyteCLI
    FlyteStdLib --> |dependency| FlytePropeller
    FlyteStdLib --> |dependency| FlyteAdmin
    FlyteStdLib --> |dependency| DataCatalog
    FlyteStdLib --> |dependency| FlytePlugins
    FlyteStdLib --> |dependency| FlyteCopilot
    
    FlyteIDL --> |dependency| FlyteCLI
    FlyteIDL --> |dependency| FlytePropeller
    FlyteIDL --> |dependency| FlyteAdmin
    FlyteIDL --> |dependency| DataCatalog
    FlyteIDL --> |dependency| FlytePlugins
```

The `go-tidy` target ensures that the `go.mod` files in each module are properly maintained. This is important for the correct resolution of dependencies between the modules.

Sources: [Makefile:128-138]()

## Docker Image Build Process

Flyte components are packaged as Docker images for deployment. The build system includes targets for building these images.

```mermaid
graph TD
    subgraph "Docker Image Build"
        BuildNativeFlyte["build_native_flyte target"]
        DockerBuild["docker build"]
        FlyteConsole["FlyteConsole Frontend<br/>(from script/get_flyteconsole_dist.sh)"]
        GoBuild["Go Build Process"]
    end
    
    subgraph "Docker Build Context"
        Dockerfile["Dockerfile"]
        SourceCode["Source Code"]
        BuildOutputs["Build Outputs"]
    end
    
    subgraph "Output"
        FlyteImage["flyte-binary:native<br/>Docker Image"]
    end
    
    BuildNativeFlyte --> DockerBuild
    FlyteConsole --> DockerBuild
    SourceCode --> DockerBuild
    Dockerfile --> DockerBuild
    BuildOutputs --> DockerBuild
    
    DockerBuild --> FlyteImage
```

The `build_native_flyte` target builds a Docker image containing the Flyte binary for the host architecture. This is primarily used for local development. Official images are multi-architecture and built through the CI/CD pipeline.

Sources: [Makefile:119-126]()

## Development Workflow

The build system includes targets to facilitate local development:

```mermaid
graph TD
    subgraph "Local Development Setup"
        SetupLocalDev["setup_local_dev target"]
        K3dCluster["k3d Cluster"]
        FlyteDependencies["Flyte Dependencies"]
    end
    
    subgraph "Development Cycle"
        Compile["compile target"]
        DevDocs["dev-docs target"]
        LocalTesting["Local Testing"]
    end
    
    SetupLocalDev --> K3dCluster
    SetupLocalDev --> FlyteDependencies
    
    K3dCluster --> LocalTesting
    Compile --> LocalTesting
    DevDocs --> |Documentation Preview| LocalTesting
```

The `setup_local_dev` target sets up a local Kubernetes environment using k3d and installs the necessary dependencies for Flyte development.

Sources: [Makefile:115-117]()

## Documentation Build System

Flyte uses a sophisticated documentation build system that supports both local development and CI builds:

```mermaid
graph TD
    subgraph "Documentation Sources"
        SphinxSources["Sphinx Documentation<br/>Sources"]
        CondaEnvironment["conda Environment<br/>(monodocs-environment.yaml)"]
    end
    
    subgraph "Build Process"
        CondaLock["conda-lock target"] --> EnvironmentLock["monodocs-environment.lock.yaml"]
        DockerBuildCondaLock["Docker Build<br/>(docs/Dockerfile.conda-lock)"] --> CondaLock
        
        DevDocsImage["Docker Build<br/>(docs/Dockerfile.docs)"] --> DevDocs["dev-docs target"]
        EnvironmentLock --> DevDocsImage
        
        SphinxBuild["sphinx-build<br/>(make -C docs clean html)"] --> DocsOutput["HTML Documentation"]
    end
    
    SphinxSources --> SphinxBuild
    SphinxSources --> DevDocs
    CondaEnvironment --> CondaLock
    
    DevDocs --> |Local Preview| Developer["Developer"]
    DocsOutput --> |CI/CD Pipeline| PublishedDocs["Published Documentation"]
```

The documentation build system uses Conda environments to ensure reproducible builds across different platforms. Docker is used to provide a consistent build environment for local development.

Sources: [Makefile:87-108](), [Makefile:91-103]()

## Integration with CI/CD

The build system is designed to work with the CI/CD pipeline, providing targets that are used in CI workflows:

| Target | CI Usage |
|--------|----------|
| `docs` | Build documentation in CI |
| `linux_compile` | Build Linux binaries for releases |
| `release_automation` | Prepare release artifacts |
| `lint-helm-charts` | Validate Helm charts |
| `spellcheck` | Run code spellcheck |

The CI/CD pipeline uses these targets to validate changes, build artifacts, and publish releases.

Sources: [Makefile:87-90](), [Makefile:26-28](), [Makefile:39-44](), [Makefile:141-147]()

## Best Practices for Development

When working with the Flyte build system, consider these best practices:

1. Use `make compile` for local development and testing
2. Run `make go-tidy` before committing changes to ensure dependency files are up to date
3. Use `make dev-docs` to preview documentation changes locally
4. Set up a local development environment with `make setup_local_dev`
5. Use `make help` to see a list of available make targets and their descriptions

Sources: [Makefile:110-113]()