# Task Execution

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [flytepropeller/pkg/controller/nodes/task/backoff/controller.go](flytepropeller/pkg/controller/nodes/task/backoff/controller.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler.go](flytepropeller/pkg/controller/nodes/task/backoff/handler.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler_map.go](flytepropeller/pkg/controller/nodes/task/backoff/handler_map.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler_test.go](flytepropeller/pkg/controller/nodes/task/backoff/handler_test.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist.go](flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist_test.go](flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist_test.go)
- [flytepropeller/pkg/controller/nodes/task/config/config.go](flytepropeller/pkg/controller/nodes/task/config/config.go)
- [flytepropeller/pkg/controller/nodes/task/config/config_flags.go](flytepropeller/pkg/controller/nodes/task/config/config_flags.go)
- [flytepropeller/pkg/controller/nodes/task/config/config_flags_test.go](flytepropeller/pkg/controller/nodes/task/config/config_flags_test.go)
- [flytepropeller/pkg/controller/nodes/task/handler.go](flytepropeller/pkg/controller/nodes/task/handler.go)
- [flytepropeller/pkg/controller/nodes/task/handler_test.go](flytepropeller/pkg/controller/nodes/task/handler_test.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager_test.go](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager_test.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config.go](flytepropeller/pkg/controller/nodes/task/plugin_config.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config_test.go](flytepropeller/pkg/controller/nodes/task/plugin_config_test.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/noop_resourcemanager.go](flytepropeller/pkg/controller/nodes/task/resourcemanager/noop_resourcemanager.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/redis_resourcemanager.go](flytepropeller/pkg/controller/nodes/task/resourcemanager/redis_resourcemanager.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/redis_resourcemanager_test.go](flytepropeller/pkg/controller/nodes/task/resourcemanager/redis_resourcemanager_test.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/resourceconstraints.go](flytepropeller/pkg/controller/nodes/task/resourcemanager/resourceconstraints.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager.go](flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager_test.go](flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager_test.go)
- [flytepropeller/pkg/controller/nodes/task/taskexec_context.go](flytepropeller/pkg/controller/nodes/task/taskexec_context.go)
- [flytepropeller/pkg/controller/nodes/task/taskexec_context_test.go](flytepropeller/pkg/controller/nodes/task/taskexec_context_test.go)
- [flytepropeller/pkg/controller/nodes/task/transformer.go](flytepropeller/pkg/controller/nodes/task/transformer.go)
- [flytepropeller/pkg/controller/nodes/task/transformer_test.go](flytepropeller/pkg/controller/nodes/task/transformer_test.go)

</details>



This page provides a detailed overview of how tasks are executed in Flyte. It covers the task execution lifecycle, including the plugin system, resource management, and status handling. For information about node execution, see [FlytePropeller Execution Engine](#3.2). For information about array tasks specifically, see [Array Tasks](#3.4).

## Task Execution Overview

Task execution in Flyte is handled by the Task Handler, which delegates the actual execution to plugins. The Task Handler is responsible for managing the task execution lifecycle, including plugin resolution, resource management, state transitions, and event reporting.

```mermaid
graph TD
    subgraph "Task Execution Flow"
        NodeExec["NodeExecutionContext"] -->|"calls Handle()"| TaskHandler["Handler"]
        TaskHandler -->|"1. Resolves plugin"| Plugin["Plugin"]
        TaskHandler -->|"2. Creates context"| TaskExecCtx["TaskExecutionContext"]
        TaskHandler -->|"3. Invokes plugin"| Plugin
        Plugin -->|"4. Returns phase info"| TaskHandler
        TaskHandler -->|"5. Records events"| EventsRecorder["EventsRecorder"]
        TaskHandler -->|"6. Updates state"| NodeState["NodeStateWriter"]
        TaskHandler -->|"7. Returns transition"| NodeExec
    end
```

Sources: 
- [flytepropeller/pkg/controller/nodes/task/handler.go:642-797](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)
- [flytepropeller/pkg/controller/nodes/task/transformer.go:90-201](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/transformer.go)

## Task Execution Lifecycle

The task execution process follows a well-defined lifecycle that handles tasks from initialization to completion:

```mermaid
sequenceDiagram
    participant NC as NodeExecutionContext
    participant TH as Task Handler
    participant PR as Plugin Registry
    participant TEC as TaskExecutionContext
    participant P as Plugin
    participant RM as ResourceManager
    participant ER as EventsRecorder
    participant NSW as NodeStateWriter

    NC->>TH: Handle(ctx, nCtx)
    TH->>PR: ResolvePlugin(taskType, executionConfig)
    PR-->>TH: plugin
    TH->>TEC: newTaskExecutionContext(ctx, nCtx, plugin)
    TH->>TH: Get current NodeState
    
    alt First execution (PhaseUndefined)
        TH->>P: Handle(ctx, taskExecutionContext)
        P-->>TH: PhaseInfo
    else Ongoing execution
        TH->>P: Handle(ctx, taskExecutionContext)
        P-->>TH: PhaseInfo
    end
    
    TH->>TH: Process plugin transition
    TH->>ER: RecordTaskEvent(event)
    TH->>NSW: PutTaskNodeState(state)
    
    alt Task not terminal
        TH->>NC: IncrementParallelism()
    end
    
    TH-->>NC: Return transition
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/handler.go:642-797](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)
- [flytepropeller/pkg/controller/nodes/task/handler.go:376-457](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)

### Key Task Execution Phases

Tasks go through various phases during execution:

1. **PhaseUndefined**: Initial state
2. **PhaseNotReady**: Task dependencies not yet satisfied
3. **PhaseQueued**: Task submitted for execution
4. **PhaseInitializing**: Task execution is initializing
5. **PhaseWaitingForResources**: Task waiting for resources to be available
6. **PhaseRunning**: Task is actively running
7. **PhaseSuccess**: Task completed successfully
8. **PhasePermanentFailure**: Task failed (non-recoverable)
9. **PhaseRetryableFailure**: Task failed (recoverable)
10. **PhaseAborted**: Task was aborted

Sources:
- [flytepropeller/pkg/controller/nodes/task/transformer.go:31-56](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/transformer.go)

## Plugin Architecture

Task execution in Flyte is handled by plugins. The plugin system is extensible and supports different types of task execution.

```mermaid
graph TD
    subgraph "Task Handler"
        TH["Handler"]
        PR["PluginRegistry"]
        DefaultPlugin["DefaultPlugin"]
        PluginsForType["PluginsForType Map"]
    end
    
    subgraph "Task Plugins"
        CorePlugins["Core Plugins"]
        K8sPlugins["K8s Plugins"]
        WebAPIPlugins["WebAPI Plugins"]
    end
    
    subgraph "K8s Plugins"
        K8sPM["K8s PluginManager"]
        PodPlugin["Pod Plugin"]
        SparkPlugin["Spark Plugin"]
        PyTorchPlugin["PyTorch Plugin"]
    end
    
    subgraph "WebAPI Plugins"
        AgentPlugin["Agent Plugin"]
        BigQueryPlugin["BigQuery Plugin"]
        SnowflakePlugin["Snowflake Plugin"]
    end
    
    TH -->|"uses"| PR
    TH -->|"maintains"| DefaultPlugin
    TH -->|"maintains"| PluginsForType
    
    PR -->|"registers"| CorePlugins
    PR -->|"registers"| K8sPlugins
    PR -->|"registers"| WebAPIPlugins
    
    K8sPlugins -->|"manages"| K8sPM
    K8sPM -->|"creates"| PodPlugin
    K8sPM -->|"creates"| SparkPlugin
    K8sPM -->|"creates"| PyTorchPlugin
    
    WebAPIPlugins -->|"creates"| AgentPlugin
    WebAPIPlugins -->|"creates"| BigQueryPlugin
    WebAPIPlugins -->|"creates"| SnowflakePlugin
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/handler.go:228-424](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config.go:20-76](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/plugin_config.go)

### Plugin Resolution

When a task is executed, the Task Handler resolves the appropriate plugin based on the task type and execution configuration. Plugin resolution follows this logic:

1. Check for plugin overrides in the execution config for the task type
2. If no override is found, use the default plugin for the task type
3. If no default plugin for the task type exists, check if it's an agent-compatible task type
4. If all else fails, use the default plugin (if configured)

```mermaid
flowchart TD
    Start["Task Execution"] --> CheckOverrides["Check execution config for overrides"]
    CheckOverrides --> OverridesExist{"Overrides exist?"}
    OverridesExist -->|Yes| TryOverrides["Try each override plugin"]
    TryOverrides --> OverrideFound{"Override plugin\nfound?"}
    OverrideFound -->|Yes| UseOverride["Use override plugin"]
    OverrideFound -->|No| CheckMissingBehavior{"Missing behavior\nis FAIL?"}
    CheckMissingBehavior -->|Yes| FailResolution["Fail resolution"]
    CheckMissingBehavior -->|No| CheckDefaultForType["Check default plugin for task type"]
    
    OverridesExist -->|No| CheckDefaultForType
    CheckDefaultForType --> DefaultExists{"Default exists\nfor task type?"}
    DefaultExists -->|Yes| UseDefault["Use default plugin for task type"]
    DefaultExists -->|No| CheckWebAPIPlugins["Check if task type is handled by WebAPI plugins"]
    
    CheckWebAPIPlugins --> WebAPIHandles{"WebAPI handles\ntask type?"}
    WebAPIHandles -->|Yes| UseWebAPI["Use WebAPI plugin"]
    WebAPIHandles -->|No| CheckDefaultPlugin["Check if default plugin exists"]
    
    CheckDefaultPlugin --> DefaultPluginExists{"Default plugin\nexists?"}
    DefaultPluginExists -->|Yes| UseDefaultPlugin["Use default plugin"]
    DefaultPluginExists -->|No| FailWithError["Fail with error: no plugin for task type"]
    
    UseOverride --> End["Return resolved plugin"]
    UseDefault --> End
    UseWebAPI --> End
    UseDefaultPlugin --> End
    FailResolution --> End["Error: no plugin found"]
    FailWithError --> End
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/handler.go:381-423](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config.go:20-76](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/plugin_config.go)

## K8s Plugin Manager

The K8s Plugin Manager is a core component of Flyte's task execution system. It creates and manages Kubernetes resources for task execution.

```mermaid
graph TD
    subgraph "K8s Plugin Manager"
        PM["PluginManager"]
        RM["ResourceMonitor"]
        BOC["BackOffController"]
        EW["EventWatcher"]
    end
    
    subgraph "Kubernetes"
        Client["KubeClient"]
        Resources["K8s Resources"]
        Events["K8s Events"]
    end
    
    subgraph "Plugin Lifecycle Methods"
        Handle["Handle()"]
        LaunchResource["launchResource()"]
        CheckResourcePhase["checkResourcePhase()"]
        Abort["Abort()"]
        Finalize["Finalize()"]
    end
    
    PM -->|"uses"| Client
    PM -->|"monitors"| RM
    PM -->|"manages backoff"| BOC
    PM -->|"watches"| EW
    
    Handle -->|"calls"| LaunchResource
    Handle -->|"calls"| CheckResourcePhase
    
    LaunchResource -->|"creates"| Resources
    CheckResourcePhase -->|"checks"| Resources
    EW -->|"monitors"| Events
    Abort -->|"deletes/updates"| Resources
    Finalize -->|"removes finalizers"| Resources
    
    BOC -->|"controls"| LaunchResource
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:99-552](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:332-407](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)

### K8s Plugin Execution Flow

```mermaid
sequenceDiagram
    participant TC as TaskExecutionContext
    participant PM as PluginManager
    participant KC as KubeClient
    participant KA as Kubernetes API
    participant BOC as BackOffController
    
    TC->>PM: Handle(ctx, tCtx)
    
    alt First time (PhaseNotStarted)
        PM->>PM: Build resource (Pod, etc.)
        PM->>PM: Add object metadata
        
        alt Resource quota may be exceeded
            PM->>BOC: GetOrCreateHandler(resourceKey)
            BOC-->>PM: backOffHandler
            PM->>BOC: Handle(operation, resourceRequest)
            
            alt Resource request exceeds ceiling
                BOC-->>PM: Error: operation blocked by backoff
                PM-->>TC: PhaseWaitingForResources
            else Resource request within ceiling
                BOC->>KC: Create(resource)
                
                alt Creation successful
                    KC->>KA: Create resource
                    KA-->>KC: Resource created
                    KC-->>BOC: Success
                    BOC-->>PM: Success
                    PM-->>TC: PhaseQueued
                else Resource quota exceeded
                    KC->>KA: Create resource
                    KA-->>KC: Error: resource quota exceeded
                    KC-->>BOC: Error: resource quota exceeded
                    BOC-->>PM: Error: resource quota exceeded
                    PM-->>TC: PhaseWaitingForResources
                end
            end
            
        else Normal resource creation
            PM->>KC: Create(resource)
            
            alt Creation successful
                KC->>KA: Create resource
                KA-->>KC: Resource created
                KC-->>PM: Success
                PM-->>TC: PhaseQueued
            else Creation failed
                KC->>KA: Create resource
                KA-->>KC: Error
                KC-->>PM: Error
                PM-->>TC: Error based on failure type
            end
        end
        
    else Resource already started
        PM->>PM: Build identity resource
        PM->>KC: Get(resource)
        KC->>KA: Get resource
        KA-->>KC: Resource
        KC-->>PM: Resource
        
        PM->>PM: GetTaskPhase(resource)
        
        alt Resource successful
            PM->>TC: PhaseSuccess
        else Resource still running
            PM->>TC: PhaseRunning
        else Resource failed
            PM->>TC: PhaseRetryableFailure or PhasePermanentFailure
        end
    end
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:272-330](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:332-407](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)

## Resource Management

Resource management is a critical aspect of task execution in Flyte. It ensures that tasks are executed efficiently and within resource constraints.

```mermaid
graph TD
    subgraph "Resource Management Architecture"
        TH["Task Handler"]
        RM["ResourceManager"]
        RTRM["TaskResourceManager (Proxy)"]
        BCRM["BaseResourceManager"]
        BRM["Builder"]
        RR["ResourceRegistrar"]
    end
    
    subgraph "Resource Manager Implementations"
        NoopRM["NoopResourceManager"]
        RedisRM["RedisResourceManager"]
    end
    
    subgraph "Resource Constraints"
        PRC["ProjectScopeResourceConstraint"]
        NRC["NamespaceScopeResourceConstraint"]
    end
    
    TH -->|"creates"| RTRM
    RTRM -->|"wraps"| BCRM
    BRM -->|"builds"| BCRM
    RM -->|"provides"| RR
    
    BCRM -.->|"implemented by"| NoopRM
    BCRM -.->|"implemented by"| RedisRM
    
    RTRM -->|"composes"| PRC
    RTRM -->|"composes"| NRC
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager.go:13-152](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/noop_resourcemanager.go:1-49](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/resourcemanager/noop_resourcemanager.go)
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/redis_resourcemanager.go:13-174](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/resourcemanager/redis_resourcemanager.go)

### Resource Allocation Flow

```mermaid
sequenceDiagram
    participant P as Plugin
    participant RTRM as TaskResourceManager
    participant RM as ResourceManager
    
    P->>RTRM: AllocateResource(namespace, token, constraints)
    RTRM->>RTRM: ComposeResourceConstraint(constraints)
    RTRM->>RTRM: Prepend token prefix
    RTRM->>RM: AllocateResource(namespace, prefixedToken, composedConstraints)
    RM-->>RTRM: AllocationStatus
    RTRM->>RTRM: Record ResourcePoolInfo
    RTRM-->>P: AllocationStatus
    
    P->>RTRM: ReleaseResource(namespace, token)
    RTRM->>RTRM: Prepend token prefix
    RTRM->>RM: ReleaseResource(namespace, prefixedToken)
    RM-->>RTRM: Success/Error
    RTRM-->>P: Success/Error
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager.go:93-135](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/resourcemanager/resourcemanager.go)

## Backoff Mechanism

When resources are constrained, Flyte uses a backoff mechanism to avoid constant retries that would overwhelm the system.

```mermaid
graph TD
    subgraph "Backoff System Components"
        BOC["BackOffController"]
        CRABH["ComputeResourceAwareBackOffHandler"]
        SBB["SimpleBackOffBlocker"]
        CRC["ComputeResourceCeilings"]
    end
    
    subgraph "Flow Control"
        IsBlocking{"Is blocking?"}
        IsEligible{"Resource request\nbelow ceiling?"}
        Operation["Operation (e.g., create pod)"]
        IsResourceError{"Resource\nquota error?"}
        UpdateCeiling["Update resource ceiling"]
        BackOff["Back off (increase exponent)"]
        BlockOperation["Block operation"]
        ReturnSuccess["Return success"]
        ReturnError["Return error"]
    end
    
    BOC -->|"manages"| CRABH
    CRABH -->|"contains"| SBB
    CRABH -->|"contains"| CRC
    
    CRABH -->|"flow begins"| IsBlocking
    IsBlocking -->|"Yes"| IsEligible
    IsBlocking -->|"No"| Operation
    
    IsEligible -->|"Yes"| Operation
    IsEligible -->|"No"| BlockOperation
    
    Operation -->|"Result"| IsResourceError
    IsResourceError -->|"Yes"| UpdateCeiling
    IsResourceError -->|"No"| ReturnSuccess
    
    UpdateCeiling --> BackOff
    BackOff --> ReturnError
    BlockOperation --> ReturnError
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/backoff/handler.go:128-192](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/backoff/handler.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/controller.go:22-62](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/backoff/controller.go)

## Task Execution Context

The Task Execution Context provides all the information needed for a plugin to execute a task.

```mermaid
classDiagram
    class TaskExecutionContext {
        +TaskExecutionMetadata() TaskExecutionMetadata
        +InputReader() InputReader
        +OutputWriter() OutputWriter
        +TaskReader() TaskReader
        +PluginStateReader() PluginStateReader
        +PluginStateWriter() PluginStateWriter
        +ResourceManager() ResourceManager
        +SecretManager() SecretManager
        +DataStore() DataStore
        +Catalog() CatalogClient
        +EventsRecorder() EventsRecorder
        +TaskRefreshIndicator() SignalAsync
    }
    
    class TaskExecutionMetadata {
        +GetTaskExecutionID() TaskExecutionID
        +GetNamespace() string
        +GetLabels() map[string]string
        +GetAnnotations() map[string]string
        +GetOwnerReference() OwnerReference
        +GetOverrides() TaskOverrides
        +GetMaxAttempts() uint32
        +GetPlatformResources() ResourceRequirements
        +GetEnvironmentVariables() map[string]string
    }
    
    class TaskExecutionID {
        +GetID() TaskExecutionIdentifier
        +GetGeneratedName() string
        +GetUniqueNodeID() string
    }
    
    TaskExecutionContext "1" --> "1" TaskExecutionMetadata
    TaskExecutionMetadata "1" --> "1" TaskExecutionID
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/taskexec_context.go:28-106](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/taskexec_context.go)
- [flytepropeller/pkg/controller/nodes/task/taskexec_context.go:106-154](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/taskexec_context.go)
- [flytepropeller/pkg/controller/nodes/task/taskexec_context.go:244-316](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/taskexec_context.go)

## Task Execution Events

Task execution progress is tracked through events. These events are sent to FlyteAdmin and used to update the UI and track task status.

| Event Phase | Description | Next Possible Phases |
|-------------|-------------|----------------------|
| UNDEFINED | Initial state | QUEUED |
| QUEUED | Task submitted for execution | INITIALIZING, RUNNING, FAILED |
| INITIALIZING | Task execution is initializing | RUNNING, WAITING_FOR_RESOURCES, FAILED |
| WAITING_FOR_RESOURCES | Task waiting for resources | INITIALIZING, RUNNING, FAILED |
| RUNNING | Task is actively running | SUCCEEDED, FAILED |
| SUCCEEDED | Task completed successfully | (terminal) |
| FAILED | Task failed (permanent or retryable) | (terminal) |
| ABORTED | Task was aborted | (terminal) |

Sources:
- [flytepropeller/pkg/controller/nodes/task/transformer.go:31-56](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/transformer.go)
- [flytepropeller/pkg/controller/nodes/task/transformer.go:90-201](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/transformer.go)
- [flytepropeller/pkg/controller/nodes/task/handler.go:734-775](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)

## Error Handling and Recovery

Task execution can encounter various error scenarios that need to be handled appropriately.

### Output Validation Errors

When a task succeeds but fails to produce valid outputs, the handler determines if the error is recoverable:

```mermaid
flowchart TD
    Start["Validate Output"] --> TaskSuccess{"Task reports\nsuccess?"}
    TaskSuccess -->|No| EndNoError["Return no error"]
    TaskSuccess -->|Yes| OutputsDeclared{"Outputs\ndeclared?"}
    
    OutputsDeclared -->|No| EndNoError
    OutputsDeclared -->|Yes| ReaderExists{"Output reader\nexists?"}
    
    ReaderExists -->|No| ReturnRecoverable["Return recoverable error:\nOutputs not generated"]
    ReaderExists -->|Yes| CheckError{"Is output\nan error?"}
    
    CheckError -->|Yes| ReadError["Read error from output"]
    ReadError --> ReturnTaskError["Return task error"]
    
    CheckError -->|No| CheckOutputExists{"Output file\nexists?"}
    CheckOutputExists -->|No| ReturnRecoverable2["Return recoverable error:\nOutputs not found"]
    
    CheckOutputExists -->|Yes| IsFile{"Is direct file?"}
    IsFile -->|No| CommitOutput["Commit output to storage"]
    CommitOutput --> EndNoError
    
    IsFile -->|Yes| EndNoError
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/handler.go:798-884](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)

### Resource Quota Handling

When a task encounters resource quota limitations, Flyte uses a backoff mechanism to avoid overwhelming the system with retries:

```mermaid
flowchart TD
    Start["Task Resource Creation"] --> QuotaExceeded{"Resource quota\nexceeded?"}
    
    QuotaExceeded -->|No| Continue["Continue execution"]
    QuotaExceeded -->|Yes| EligibleRequest{"Request exceeds\nquota limits?"}
    
    EligibleRequest -->|Yes| PermanentFailure["Permanent failure:\nResourceRequestsExceedLimits"]
    EligibleRequest -->|No| BackOffActive{"BackOff\ncontroller active?"}
    
    BackOffActive -->|Yes| UpdateBackOff["Update backoff parameters"]
    BackOffActive -->|No| SimpleWait["Wait for resources"]
    
    UpdateBackOff --> WaitForResources["Transition to\nWaitingForResources phase"]
    SimpleWait --> WaitForResources
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:232-246](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler.go:128-192](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/backoff/handler.go)

## Task Abort and Finalization

Tasks may need to be aborted or finalized in certain scenarios, such as when a workflow is terminated or when resources need to be cleaned up.

### Abort Process

```mermaid
sequenceDiagram
    participant NC as NodeExecutionContext
    participant TH as Task Handler
    participant P as Plugin
    participant ER as EventsRecorder
    
    NC->>TH: Abort(ctx, nCtx, reason)
    
    alt Task in terminal phase (and not cleanup on failure)
        TH-->>NC: Return (nothing to do)
    else Task needs abortion
        TH->>TH: ResolvePlugin(taskType, executionConfig)
        TH->>TH: Create taskExecutionContext
        TH->>P: Abort(ctx, tCtx)
        
        TH->>ER: Record buffered events with Aborted phase
        TH->>ER: Record abort event with reason
        
        TH-->>NC: Return success or error
    end
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/handler.go:886-964](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/handler.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:410-457](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)

### K8s Resource Finalization

```mermaid
sequenceDiagram
    participant PM as PluginManager
    participant Client as KubeClient
    participant O as K8sObject
    
    PM->>PM: getResource(tCtx)
    PM->>Client: Get(resource)
    
    alt Resource exists
        Client-->>PM: Resource object
        PM->>PM: clearFinalizer(ctx, object)
        PM->>Client: Update(object without finalizer)
        
        alt DeleteResourceOnFinalize enabled
            PM->>Client: Delete(object)
        end
    else Resource doesn't exist
        Client-->>PM: NotFound error
        PM-->>PM: Continue (nothing to do)
    end
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:460-552](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)

## Configuration

Task execution behavior can be configured through various configuration options:

| Configuration | Description | Default |
|---------------|-------------|---------|
| `task-plugins.enabled-plugins` | List of plugins to enable | [] (all plugins enabled) |
| `task-plugins.default-for-task-types` | Maps task types to their default plugin handler | {} |
| `max-plugin-phase-versions` | Maximum number of plugin phase versions allowed | 100000 |
| `backoff.base-second` | Base duration for exponential backoff | 2 seconds |
| `backoff.max-duration` | Maximum backoff duration | 20 seconds |

Sources:
- [flytepropeller/pkg/controller/nodes/task/config/config.go:15-101](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/config/config.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config.go:20-76](https://github.com/flyteorg/flyte/blob/master/flytepropeller/pkg/controller/nodes/task/plugin_config.go)