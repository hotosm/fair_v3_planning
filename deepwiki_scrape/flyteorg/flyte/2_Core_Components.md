# Core Components

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [datacatalog/go.mod](datacatalog/go.mod)
- [datacatalog/go.sum](datacatalog/go.sum)
- [flyteadmin/go.mod](flyteadmin/go.mod)
- [flyteadmin/go.sum](flyteadmin/go.sum)
- [flytecopilot/go.mod](flytecopilot/go.mod)
- [flytecopilot/go.sum](flytecopilot/go.sum)
- [flyteidl/clients/go/assets/admin.swagger.json](flyteidl/clients/go/assets/admin.swagger.json)
- [flyteidl/gen/pb-es/flyteidl/core/workflow_pb.ts](flyteidl/gen/pb-es/flyteidl/core/workflow_pb.ts)
- [flyteidl/gen/pb-go/flyteidl/core/workflow.pb.go](flyteidl/gen/pb-go/flyteidl/core/workflow.pb.go)
- [flyteidl/gen/pb-go/flyteidl/service/admin.pb.go](flyteidl/gen/pb-go/flyteidl/service/admin.pb.go)
- [flyteidl/gen/pb-go/gateway/flyteidl/service/admin.swagger.json](flyteidl/gen/pb-go/gateway/flyteidl/service/admin.swagger.json)
- [flyteidl/gen/pb-go/gateway/flyteidl/service/agent.swagger.json](flyteidl/gen/pb-go/gateway/flyteidl/service/agent.swagger.json)
- [flyteidl/gen/pb-go/gateway/flyteidl/service/external_plugin_service.swagger.json](flyteidl/gen/pb-go/gateway/flyteidl/service/external_plugin_service.swagger.json)
- [flyteidl/gen/pb-js/flyteidl.d.ts](flyteidl/gen/pb-js/flyteidl.d.ts)
- [flyteidl/gen/pb-js/flyteidl.js](flyteidl/gen/pb-js/flyteidl.js)
- [flyteidl/gen/pb_python/flyteidl/core/workflow_pb2.py](flyteidl/gen/pb_python/flyteidl/core/workflow_pb2.py)
- [flyteidl/gen/pb_python/flyteidl/core/workflow_pb2.pyi](flyteidl/gen/pb_python/flyteidl/core/workflow_pb2.pyi)
- [flyteidl/gen/pb_python/flyteidl/service/admin_pb2.py](flyteidl/gen/pb_python/flyteidl/service/admin_pb2.py)
- [flyteidl/gen/pb_rust/flyteidl.core.rs](flyteidl/gen/pb_rust/flyteidl.core.rs)
- [flyteidl/go.mod](flyteidl/go.mod)
- [flyteidl/go.sum](flyteidl/go.sum)
- [flyteidl/protos/flyteidl/core/workflow.proto](flyteidl/protos/flyteidl/core/workflow.proto)
- [flyteidl/protos/flyteidl/service/admin.proto](flyteidl/protos/flyteidl/service/admin.proto)
- [flyteplugins/go.mod](flyteplugins/go.mod)
- [flyteplugins/go.sum](flyteplugins/go.sum)
- [flytepropeller/go.mod](flytepropeller/go.mod)
- [flytepropeller/go.sum](flytepropeller/go.sum)
- [go.mod](go.mod)
- [go.sum](go.sum)

</details>



This document provides a comprehensive overview of the core components that make up the Flyte platform. It explains the purpose and functionality of each key component, how they interact, and their role in the overall workflow execution lifecycle. For details on how these components are deployed, see [Deployment](#2).

## Overview of Flyte Architecture

Flyte's architecture is designed with separation of concerns and scalability in mind, divided into two main planes:

1. **Control Plane** - Manages workflow definitions, schedules, and metadata
2. **Data Plane** - Handles the actual execution of workflows and tasks

```mermaid
graph TD
    subgraph "Control Plane"
        FlyteAdmin["FlyteAdmin Service"]
        DataCatalog["DataCatalog Service"]
        FlyteConsole["FlyteConsole UI"]
        DB[(Metadata Database)]
    end
    
    subgraph "Data Plane"
        FlytePropeller["FlytePropeller"]
        FlytePlugins["FlytePlugins"]
        FlyteCoPilot["FlyteCoPilot Sidecar"]
    end
    
    subgraph "External Resources"
        K8s[(Kubernetes API)]
        ObjectStore[(Object Storage)]
        ExtServices[External Services]
    end
    
    subgraph "User Interfaces"
        CLI["flytectl CLI"]
        SDKs["Python/Java/Scala SDKs"]
    end
    
    CLI --> FlyteAdmin
    SDKs --> FlyteAdmin
    FlyteConsole --> FlyteAdmin
    
    FlyteAdmin --> DB
    FlyteAdmin --> ObjectStore
    FlyteAdmin --> FlytePropeller
    FlyteAdmin <--> DataCatalog
    
    DataCatalog --> DB
    DataCatalog --> ObjectStore
    
    FlytePropeller --> K8s
    FlytePropeller --> FlytePlugins
    FlytePropeller --> ObjectStore
    FlytePropeller --> FlyteAdmin
    
    FlytePlugins --> K8s
    FlytePlugins --> ExtServices
    
    FlyteCoPilot --> ObjectStore
```

Sources:
- [flytepropeller/go.mod]()
- [flyteadmin/go.mod]()
- [go.mod]()

## Core Components in Detail

### FlyteAdmin

FlyteAdmin is the centralized control plane service that handles all user-facing API requests and maintains workflow state. It's responsible for:

1. Registering workflow entities (tasks, workflows, launch plans)
2. Managing executions (creating, monitoring, terminating)
3. Storing and retrieving workflow metadata
4. Handling authentication and authorization
5. Managing workflow schedules

FlyteAdmin exposes a gRPC service defined by the FlyteIDL and also provides a REST interface through the gRPC Gateway.

```mermaid
graph TD
    subgraph "FlyteAdmin Service"
        AdminService["AdminService"]
        AuthHandler["Authentication/Authorization"]
        MetadataManager["Metadata Manager"]
        StorageManager["Storage Manager"]
        ExecutionManager["Execution Manager"]
    end
    
    subgraph "External Systems"
        DB[(Database)]
        ObjectStore[(Object Storage)]
        Propeller[FlytePropeller]
        DataCat[DataCatalog]
    end
    
    Client[Client Requests] --> AdminService
    AdminService --> AuthHandler
    AdminService --> MetadataManager
    AdminService --> StorageManager
    AdminService --> ExecutionManager
    
    MetadataManager --> DB
    StorageManager --> ObjectStore
    ExecutionManager --> Propeller
    ExecutionManager --> DataCat
```

Sources:
- [flyteadmin/go.mod]()
- [flyteadmin/go.sum]()

### DataCatalog

DataCatalog is a service that manages metadata about datasets produced and consumed by Flyte workflows. Key features include:

1. Dataset versioning
2. Artifact tracking
3. Data lineage
4. Cached execution results

DataCatalog helps optimize workflow execution by enabling the reuse of previously computed results when inputs haven't changed.

```mermaid
graph TD
    subgraph "DataCatalog Service"
        CatalogService["CatalogService"]
        ArtifactManager["Artifact Manager"]
        CacheManager["Cache Manager"]
        MetadataManager["Metadata Manager"]
    end
    
    subgraph "Storage"
        DB[(Database)]
        ObjectStore[(Object Storage)]
    end
    
    FlyteAdmin[FlyteAdmin] --> CatalogService
    FlytePropeller[FlytePropeller] --> CatalogService
    
    CatalogService --> ArtifactManager
    CatalogService --> CacheManager
    CatalogService --> MetadataManager
    
    ArtifactManager --> ObjectStore
    CacheManager --> DB
    MetadataManager --> DB
```

Sources:
- [datacatalog/go.mod]()
- [datacatalog/go.sum]()

### FlytePropeller

FlytePropeller is the workflow execution engine that runs in the data plane. It is implemented as a Kubernetes operator that processes workflow custom resources. Key responsibilities include:

1. Executing workflows according to their specification
2. Traversing workflow DAGs and scheduling tasks
3. Handling retries and error conditions
4. Managing workflow state and task execution
5. Coordinating with plugins for task execution

FlytePropeller operates on a custom Kubernetes resource called `FlyteWorkflow` which represents the compiled form of a workflow.

```mermaid
graph TD
    subgraph "FlytePropeller Components"
        Controller["Controller"]
        WorkflowStore["Workflow Store"]
        NodeExecutor["Node Executor"]
        TaskExecutor["Task Executor"]
        PluginRegistry["Plugin Registry"]
    end
    
    subgraph "Kubernetes"
        CRD["FlyteWorkflow CRD"]
        K8sAPI["Kubernetes API"]
    end
    
    subgraph "External Systems"
        ObjectStore[(Object Storage)]
        FlyteAdmin[FlyteAdmin]
    end
    
    Controller --> WorkflowStore
    Controller --> NodeExecutor
    NodeExecutor --> TaskExecutor
    TaskExecutor --> PluginRegistry
    
    WorkflowStore <--> CRD
    TaskExecutor <--> K8sAPI
    PluginRegistry <--> K8sAPI
    
    Controller --> ObjectStore
    Controller --> FlyteAdmin
```

Sources:
- [flytepropeller/go.mod]()
- [flytepropeller/go.sum]()

### FlytePlugins

FlytePlugins is an extensible system for executing different types of tasks. Plugins are categorized into:

1. **Kubernetes Plugins** - Execute tasks as Kubernetes resources
   - Pod Plugin (for container tasks)
   - Spark Plugin
   - PyTorch Plugin
   - TensorFlow Plugin
   - MPI Plugin
   - Ray Plugin

2. **WebAPI Plugins** - Interface with external services
   - BigQuery
   - Snowflake
   - Athena
   - Redshift
   - Other cloud services

Each plugin implements a specific interface to handle task execution for its supported task type.

```mermaid
graph TD
    subgraph "Plugin System"
        PluginRegistry["Plugin Registry"]
        
        subgraph "K8s Plugins"
            PodPlugin["Pod Plugin"]
            SparkPlugin["Spark Plugin"]
            MLPlugins["ML Framework Plugins"]
            RayPlugin["Ray Plugin"]
        end
        
        subgraph "WebAPI Plugins"
            BigQueryPlugin["BigQuery Plugin"]
            SnowflakePlugin["Snowflake Plugin"]
            AthenaPlugin["Athena Plugin"]
            RedshiftPlugin["Redshift Plugin"]
        end
    end
    
    subgraph "External Systems"
        K8sAPI["Kubernetes API"]
        CloudServices["Cloud Services"]
    end
    
    TaskExecutor[TaskExecutor] --> PluginRegistry
    
    PluginRegistry --> PodPlugin
    PluginRegistry --> SparkPlugin
    PluginRegistry --> MLPlugins
    PluginRegistry --> RayPlugin
    
    PluginRegistry --> BigQueryPlugin
    PluginRegistry --> SnowflakePlugin
    PluginRegistry --> AthenaPlugin
    PluginRegistry --> RedshiftPlugin
    
    PodPlugin --> K8sAPI
    SparkPlugin --> K8sAPI
    MLPlugins --> K8sAPI
    RayPlugin --> K8sAPI
    
    BigQueryPlugin --> CloudServices
    SnowflakePlugin --> CloudServices
    AthenaPlugin --> CloudServices
    RedshiftPlugin --> CloudServices
```

Sources:
- [flyteplugins/go.mod]()
- [flyteplugins/go.sum]()

### FlyteCoPilot

FlyteCoPilot is a sidecar container that runs alongside task pods to facilitate input/output handling. It provides:

1. Download of input data from object storage
2. Upload of output data to object storage
3. Checkpointing support
4. Metadata collection

This sidecar helps abstract storage operations from the task containers themselves.

```mermaid
graph TD
    subgraph "Kubernetes Pod"
        TaskContainer["Task Container"]
        CoPilotContainer["FlyteCoPilot Container"]
        
        subgraph "Shared Volume"
            InputsDir["Inputs Directory"]
            OutputsDir["Outputs Directory"]
        end
    end
    
    subgraph "External Resources"
        ObjectStore[(Object Storage)]
        FlyteAdmin[FlyteAdmin]
    end
    
    CoPilotContainer -->|"Download Inputs"| ObjectStore
    CoPilotContainer -->|"Write to"| InputsDir
    TaskContainer -->|"Read from"| InputsDir
    TaskContainer -->|"Write to"| OutputsDir
    CoPilotContainer -->|"Read from"| OutputsDir
    CoPilotContainer -->|"Upload Outputs"| ObjectStore
    CoPilotContainer -->|"Report Metadata"| FlyteAdmin
```

Sources:
- [flytecopilot/go.mod]()
- [flytecopilot/go.sum]()

### FlyteConsole

FlyteConsole is the web-based user interface for Flyte. It provides:

1. Workflow visualization and monitoring
2. Execution launching and management
3. Task logs and outputs inspection
4. Project and domain management

FlyteConsole communicates with FlyteAdmin via its REST API.

### flytectl (CLI)

flytectl is the command-line interface for interacting with Flyte. It enables:

1. Workflow registration
2. Execution creation and management
3. Configuration of projects, domains, and launch plans
4. System administration tasks

The CLI provides similar capabilities to the web UI but through a terminal interface.

### FlyteIDL

FlyteIDL (Interface Definition Language) defines the protocol buffers and gRPC service definitions for all Flyte services. It serves as the contract between different components and clients. Key elements include:

1. Core data types (tasks, workflows, literals)
2. Admin service definitions
3. DataCatalog service definitions
4. Plugin APIs

All inter-component communication in Flyte is defined through these interfaces.

```mermaid
graph TD
    subgraph "FlyteIDL"
        CoreProtos["Core Protos\n(tasks, workflows, literals)"]
        AdminProtos["Admin Service Protos"]
        CatalogProtos["DataCatalog Protos"]
        PluginProtos["Plugin API Protos"]
    end
    
    subgraph "Used By"
        FlyteAdmin["FlyteAdmin"]
        FlytePropeller["FlytePropeller"]
        DataCatalog["DataCatalog"]
        FlytePlugins["FlytePlugins"]
        SDKs["Client SDKs"]
        CLI["flytectl"]
        Console["FlyteConsole"]
    end
    
    CoreProtos --> FlyteAdmin
    CoreProtos --> FlytePropeller
    CoreProtos --> DataCatalog
    CoreProtos --> FlytePlugins
    CoreProtos --> SDKs
    
    AdminProtos --> FlyteAdmin
    AdminProtos --> CLI
    AdminProtos --> Console
    AdminProtos --> SDKs
    
    CatalogProtos --> DataCatalog
    CatalogProtos --> FlyteAdmin
    
    PluginProtos --> FlytePlugins
    PluginProtos --> FlytePropeller
```

Sources:
- [flyteidl/go.mod]()
- [flyteidl/go.sum]()
- [flyteidl/gen/pb-js/flyteidl.js]()
- [flyteidl/gen/pb-js/flyteidl.d.ts]()

## Workflow Execution Flow

To understand how these components work together, let's trace the flow of a workflow execution:

```mermaid
sequenceDiagram
    participant User as User
    participant Admin as FlyteAdmin
    participant Propeller as FlytePropeller
    participant Plugins as FlytePlugins
    participant CoPilot as FlyteCoPilot
    participant K8s as Kubernetes
    participant Storage as Object Storage
    participant Catalog as DataCatalog
    
    User->>Admin: Register workflow
    Admin->>Storage: Store workflow definition
    
    User->>Admin: Launch execution
    Admin->>Storage: Store execution inputs
    Admin->>Propeller: Create FlyteWorkflow CR
    
    Propeller->>Propeller: Traverse workflow graph
    
    loop For each node
        alt Task Node
            Propeller->>Plugins: Execute task
            Plugins->>K8s: Create pod/resource
            K8s->>CoPilot: Start sidecar
            K8s->>K8s: Start task container
            
            CoPilot->>Storage: Download inputs
            CoPilot->>K8s: Signal task to start
            K8s->>K8s: Task processes inputs
            K8s->>CoPilot: Task completes
            CoPilot->>Storage: Upload outputs
            CoPilot->>Propeller: Report completion
            
        else Subworkflow Node
            Propeller->>Propeller: Process as nested workflow
            
        else Dynamic Node
            Propeller->>Plugins: Execute parent task
            Plugins->>Propeller: Return dynamic workflow
            Propeller->>Propeller: Execute dynamic workflow
        end
        
        Propeller->>Catalog: Register outputs
        Propeller->>Admin: Update node execution
    end
    
    Propeller->>Admin: Update execution status
    Admin->>User: Execution complete
```

This diagram illustrates how a workflow moves through the system from registration to execution completion, showing the interaction between all core components.

Sources:
- [flytepropeller/go.mod]()
- [flyteadmin/go.mod]()
- [flyteplugins/go.mod]()
- [flyteidl/go.mod]()

## Summary

Flyte's architecture is composed of several key components that work together to provide a scalable, reliable workflow platform:

| Component | Role | Location |
|-----------|------|----------|
| FlyteAdmin | Control plane API service | Admin namespace |
| DataCatalog | Data artifact tracking service | Admin namespace |
| FlytePropeller | Workflow execution engine | Execution namespace |
| FlytePlugins | Task execution plugins | Used by FlytePropeller |
| FlyteCoPilot | Task I/O sidecar | Runs in task pods |
| FlyteConsole | Web UI | Admin namespace |
| flytectl | Command-line interface | User machine |
| FlyteIDL | Interface definitions | Used by all components |

These components are designed with clear separation of concerns, allowing for independent scaling and high reliability. The control plane components (FlyteAdmin, DataCatalog) manage metadata and coordination, while the data plane components (FlytePropeller, FlytePlugins, FlyteCoPilot) handle the actual execution of workflows.