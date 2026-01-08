# Kubernetes Plugin Machinery

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [flyteplugins/go/tasks/config_load_test.go](flyteplugins/go/tasks/config_load_test.go)
- [flyteplugins/go/tasks/pluginmachinery/core/mocks/fake_k8s_cache.go](flyteplugins/go/tasks/pluginmachinery/core/mocks/fake_k8s_cache.go)
- [flyteplugins/go/tasks/pluginmachinery/core/mocks/fake_k8s_client.go](flyteplugins/go/tasks/pluginmachinery/core/mocks/fake_k8s_client.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/config/config.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/config/config.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/config/k8spluginconfig_flags.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/config/k8spluginconfig_flags.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/config/k8spluginconfig_flags_test.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/config/k8spluginconfig_flags_test.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper_test.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper_test.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/k8s_resource_adds.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/k8s_resource_adds.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/k8s_resource_adds_test.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/k8s_resource_adds_test.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper_test.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper_test.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/testdata/config.yaml](flyteplugins/go/tasks/pluginmachinery/flytek8s/testdata/config.yaml)
- [flyteplugins/go/tasks/plugins/array/k8s/integration_test.go](flyteplugins/go/tasks/plugins/array/k8s/integration_test.go)
- [flyteplugins/go/tasks/plugins/k8s/pod/container_test.go](flyteplugins/go/tasks/plugins/k8s/pod/container_test.go)
- [flyteplugins/go/tasks/plugins/k8s/pod/plugin.go](flyteplugins/go/tasks/plugins/k8s/pod/plugin.go)
- [flyteplugins/go/tasks/plugins/k8s/pod/sidecar_test.go](flyteplugins/go/tasks/plugins/k8s/pod/sidecar_test.go)
- [flyteplugins/go/tasks/plugins/k8s/spark/spark.go](flyteplugins/go/tasks/plugins/k8s/spark/spark.go)
- [flyteplugins/go/tasks/plugins/k8s/spark/spark_test.go](flyteplugins/go/tasks/plugins/k8s/spark/spark_test.go)
- [flyteplugins/go/tasks/testdata/config.yaml](flyteplugins/go/tasks/testdata/config.yaml)
- [flytepropeller/pkg/controller/nodes/task/backoff/controller.go](flytepropeller/pkg/controller/nodes/task/backoff/controller.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler.go](flytepropeller/pkg/controller/nodes/task/backoff/handler.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler_map.go](flytepropeller/pkg/controller/nodes/task/backoff/handler_map.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler_test.go](flytepropeller/pkg/controller/nodes/task/backoff/handler_test.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist.go](flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist.go)
- [flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist_test.go](flytepropeller/pkg/controller/nodes/task/backoff/safe_resourcelist_test.go)
- [flytepropeller/pkg/controller/nodes/task/config/config.go](flytepropeller/pkg/controller/nodes/task/config/config.go)
- [flytepropeller/pkg/controller/nodes/task/config/config_flags.go](flytepropeller/pkg/controller/nodes/task/config/config_flags.go)
- [flytepropeller/pkg/controller/nodes/task/config/config_flags_test.go](flytepropeller/pkg/controller/nodes/task/config/config_flags_test.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager_test.go](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager_test.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config.go](flytepropeller/pkg/controller/nodes/task/plugin_config.go)
- [flytepropeller/pkg/controller/nodes/task/plugin_config_test.go](flytepropeller/pkg/controller/nodes/task/plugin_config_test.go)

</details>



This page documents the Kubernetes Plugin Machinery in Flyte, which provides the framework for converting Flyte tasks to Kubernetes resources, managing their lifecycle, monitoring execution status, and handling resource customization. This is a core component that enables Flyte to execute tasks on Kubernetes clusters.

## Overview

The Kubernetes Plugin Machinery is responsible for translating logical Flyte task definitions into physical Kubernetes resources and managing their execution. It provides a standardized approach for creating, monitoring, and cleaning up these resources, with support for various resource types beyond basic Pods (such as Spark applications, MPI jobs, etc.).

```mermaid
graph TD
    subgraph "Flyte System"
        FlytePropeller["FlytePropeller<br>Workflow Executor"]
        PluginRegistry["Plugin Registry"]
        TaskExecutor["Task Executor"]
    end
    
    subgraph "K8s Plugin Machinery"
        PluginManager["PluginManager"]
        K8sPlugins["K8s Plugins<br>(Pod, Spark, etc.)"]
        ResourceHandler["Resource Handlers"]
        BackoffController["BackoffController"]
    end
    
    subgraph "Kubernetes"
        K8sAPI["Kubernetes API"]
        Pods["Pods"]
        CRDs["Custom Resources<br>(Spark, PyTorch, etc.)"]
    end
    
    FlytePropeller -->|"executes task"| TaskExecutor
    TaskExecutor -->|"selects plugin"| PluginRegistry
    PluginRegistry -->|"provides"| PluginManager
    PluginManager -->|"manages"| K8sPlugins
    K8sPlugins -->|"build resources"| ResourceHandler
    ResourceHandler -->|"create/monitor"| K8sAPI
    PluginManager -->|"handles retries"| BackoffController
    K8sAPI -->|"creates"| Pods
    K8sAPI -->|"creates"| CRDs
    K8sAPI -->|"status"| PluginManager
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:99-111](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:99-111)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:26-40](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:26-40)

## Core Components

### Plugin Manager

The `PluginManager` is the central component responsible for orchestrating the lifecycle of Kubernetes resources. It handles task execution, monitoring, and cleanup, providing a consistent interface between Flyte and Kubernetes.

```mermaid
classDiagram
    class PluginManager {
        +id string
        +plugin k8s.Plugin
        +resourceToWatch runtime.Object
        +kubeClient pluginsCore.KubeClient
        +metrics PluginMetrics
        +backOffController *backoff.Controller
        +resourceLevelMonitor *ResourceMonitorIndex
        +eventWatcher EventWatcher
        +Handle(ctx, tCtx) Transition
        +Abort(ctx, tCtx) error
        +Finalize(ctx, tCtx) error
        +GetProperties() PluginProperties
        +GetID() string
    }
```

Key methods:
- `Handle`: Main entry point for task execution
- `Abort`: Handles task abortion
- `Finalize`: Cleans up resources after execution
- `launchResource`: Creates the Kubernetes resource
- `checkResourcePhase`: Interprets resource state as task phase

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:99-111](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:99-111)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:132-142](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:132-142)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:332-407](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:332-407)

### Plugin Interface

All Kubernetes plugins must implement the `k8s.Plugin` interface, which defines how to create resources from tasks and interpret their execution status.

```mermaid
classDiagram
    class Plugin {
        +GetProperties() PluginProperties
        +BuildResource(ctx, taskCtx) client.Object
        +BuildIdentityResource(ctx, meta) client.Object
        +GetTaskPhase(ctx, pluginCtx, resource) PhaseInfo
    }
```

Key methods:
- `GetProperties`: Returns plugin configuration properties
- `BuildResource`: Creates a full Kubernetes resource from a task
- `BuildIdentityResource`: Creates a minimal resource for lookups
- `GetTaskPhase`: Determines task execution phase from resource state

Sources:
- [flyteplugins/go/tasks/plugins/k8s/pod/plugin.go:39-41](flyteplugins/go/tasks/plugins/k8s/pod/plugin.go:39-41)
- [flyteplugins/go/tasks/plugins/k8s/spark/spark.go:40-57](flyteplugins/go/tasks/plugins/k8s/spark/spark.go:40-57)

### Configuration System

The configuration system defined in `K8sPluginConfig` provides a rich set of options for customizing how resources are created and managed.

```mermaid
classDiagram
    class K8sPluginConfig {
        +InjectFinalizer bool
        +DefaultAnnotations map[string]string
        +DefaultLabels map[string]string
        +DefaultEnvVars map[string]string
        +DefaultCPURequest resource.Quantity
        +DefaultMemoryRequest resource.Quantity
        +DefaultTolerations []v1.Toleration
        +DefaultNodeSelector map[string]string
        +DefaultAffinity *v1.Affinity
        +SchedulerName string
        +InterruptibleTolerations []v1.Toleration
        +InterruptibleNodeSelector map[string]string
        +ResourceTolerations map[ResourceName][]Toleration
        +GpuResourceName v1.ResourceName
        +PodPendingTimeout config2.Duration
        +ImagePullBackoffGracePeriod config2.Duration
    }
```

Sources:
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/config/config.go:80-255](flyteplugins/go/tasks/pluginmachinery/flytek8s/config/config.go:80-255)

## Execution Workflow

The execution workflow shows how a Flyte task is transformed into a Kubernetes resource and monitored through completion.

```mermaid
sequenceDiagram
    participant Propeller as FlytePropeller
    participant Manager as PluginManager
    participant Plugin as Plugin
    participant K8s as Kubernetes API
    participant Resource as K8s Resource
    
    Propeller->>Manager: Handle(taskCtx)
    
    alt Not Started Yet
        Manager->>Plugin: BuildResource(taskCtx)
        Plugin-->>Manager: K8s Resource Spec
        Manager->>K8s: Create Resource
        
        alt Resource Creation Error
            K8s-->>Manager: Error (quota/config)
            Manager->>Manager: Handle BackOff
            Manager-->>Propeller: PhaseInfo (Queued/Failed)
        else Resource Created
            K8s-->>Manager: Resource Created
            Manager-->>Propeller: PhaseInfo (Queued)
        end
        
    else Already Started
        Manager->>Plugin: BuildIdentityResource(meta)
        Plugin-->>Manager: Identity Resource
        Manager->>K8s: Get Resource
        K8s-->>Manager: Current Resource State
        Manager->>Plugin: GetTaskPhase(resource)
        Plugin-->>Manager: PhaseInfo
        Manager-->>Propeller: Transition
    end
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:332-407](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:332-407)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:272-330](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:272-330)

## Resource Customization

One of the key features of the Kubernetes Plugin Machinery is its ability to customize Kubernetes resources with Flyte-specific configurations.

### Pod Helper Functions

The `flytek8s` package provides a rich set of functions for customizing Kubernetes pods:

```mermaid
graph TD
    TaskTemplate["TaskTemplate"] -->|"convert"| RawPod["Raw Pod Spec"]
    RawPod -->|"customize"| FlyteAnnotations["Add Flyte Annotations/Labels"]
    FlyteAnnotations -->|"add"| ResourceRequirements["Apply Resource Requirements"]
    ResourceRequirements -->|"configure"| NodeSelectors["Add Node Selectors"]
    NodeSelectors -->|"add"| Tolerations["Add Tolerations"]
    Tolerations -->|"add"| ExtendedResources["Handle Extended Resources (GPU)"]
    ExtendedResources -->|"merge with"| PodTemplate["Merge with Pod Template"]
    PodTemplate -->|"finalize"| FinalPod["Final Pod Spec"]
```

Key customization functions:
- `ToK8sPodSpec`: Main entry point for pod customization
- `BuildRawPod`: Creates basic pod spec from task template
- `ApplyFlytePodConfiguration`: Applies Flyte-specific configurations
- `UpdatePod`: Updates pod with default configurations
- `ApplyInterruptibleNodeAffinity`: Handles interruptible task scheduling
- `ApplyGPUNodeSelectors`: Configures GPU-specific node selection

Sources:
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:608-626](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:608-626)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:425-532](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:425-532)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:283-315](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:283-315)

### Resource Requirements

The machinery provides sophisticated handling of resource requirements:

```mermaid
flowchart TD
    Resources["Container Resources"] -->|"customize"| EnsureRequests["Ensure Required Resources"]
    EnsureRequests -->|"validate"| CheckLimits["Check Against Limits"]
    CheckLimits -->|"adjust if needed"| ValidResources["Valid Resource Requirements"]
    
    subgraph "Resource Processing"
        HandleCPU["Handle CPU Resources"]
        HandleMemory["Handle Memory Resources"]
        HandleGPU["Handle GPU Resources"]
        HandleStorage["Handle Storage Resources"]
    end
    
    ValidResources --> HandleCPU
    ValidResources --> HandleMemory
    ValidResources --> HandleGPU
    ValidResources --> HandleStorage
```

Sources:
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper.go:71-113](flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper.go:71-113)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper.go:150-190](flyteplugins/go/tasks/pluginmachinery/flytek8s/container_helper.go:150-190)

### Extended Resources: GPU Support

The system has special handling for GPU resources, including device selection and partition sizes:

```mermaid
flowchart TD
    Task["Task with GPU Request"] -->|"requires GPU"| ApplyGPU["ApplyGPUNodeSelectors"]
    ApplyGPU -->|"determine"| Device["Set GPU Device Type"]
    Device -->|"check for"| Partitioning["GPU Partitioning"]
    
    Partitioning -->|"unpartitioned"| UnpartitionedConfig["Configure for Unpartitioned GPU"]
    Partitioning -->|"partitioned"| PartitionedConfig["Configure for Specific Partition Size"]
    
    UnpartitionedConfig -->|"add"| NodeSelectors["Add NodeSelectorRequirements"]
    PartitionedConfig -->|"add"| NodeSelectors
    
    NodeSelectors -->|"add"| Tolerations["Add GPU Tolerations"]
```

Sources:
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:199-281](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:199-281)

### Pod Templates

The machinery supports pod templates for consistent pod configurations:

```mermaid
flowchart TD
    BasePodSpec["Base Pod Spec"] -->|"merge with"| PodTemplate["Pod Template"]
    PodTemplate -->|"apply"| DefaultContainer["Default Container Template"]
    PodTemplate -->|"apply"| PrimaryContainer["Primary Container Template"]
    PodTemplate -->|"apply"| NamedContainers["Named Container Templates"]
    
    DefaultContainer & PrimaryContainer & NamedContainers -->|"combine"| MergedPodSpec["Merged Pod Spec"]
```

Sources:
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:665-690](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:665-690)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:695-836](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go:695-836)

## BackOff Handling and Resource Management

The Kubernetes Plugin Machinery includes sophisticated backoff handling for resource quota management.

### BackOff Controller

The `BackOffController` manages backoff strategies for resource creation:

```mermaid
classDiagram
    class Controller {
        +Clock clock.Clock
        +backOffHandlerMap HandlerMap
        +GetOrCreateHandler(ctx, key, backOffBaseSecond, maxBackOffDuration) *ComputeResourceAwareBackOffHandler
    }
    
    class ComputeResourceAwareBackOffHandler {
        +SimpleBackOffBlocker *SimpleBackOffBlocker
        +ComputeResourceCeilings *ComputeResourceCeilings
        +IsActive() bool
        +reset()
        +Handle(ctx, operation, requestedResourceList) error
    }
    
    class SimpleBackOffBlocker {
        +Clock clock.Clock
        +BackOffBaseSecond int
        +MaxBackOffDuration time.Duration
        +BackOffExponent Uint32
        +NextEligibleTime AtomicTime
        +isBlocking(t) bool
        +getBlockExpirationTime() time
        +reset()
        +backOff(ctx) time.Duration
    }
    
    class ComputeResourceCeilings {
        +computeResourceCeilings *SyncResourceList
        +isEligible(requestedResourceList) bool
        +update(reqResource, reqQuantity)
        +updateAll(resources)
        +reset(resource)
        +resetAll()
    }
    
    Controller --> ComputeResourceAwareBackOffHandler
    ComputeResourceAwareBackOffHandler --> SimpleBackOffBlocker
    ComputeResourceAwareBackOffHandler --> ComputeResourceCeilings
```

The backoff system provides:
- Exponential backoff for transient errors
- Resource ceiling tracking to avoid futile retries
- Efficient management of cluster resources

Sources:
- [flytepropeller/pkg/controller/nodes/task/backoff/handler.go:26-193](flytepropeller/pkg/controller/nodes/task/backoff/handler.go:26-193)
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260)

### Resource Creation with BackOff

The resource creation process integrates with the backoff controller:

```mermaid
flowchart TD
    Start["Start Resource Creation"] -->|"attempt"| CreateResource["Create K8s Resource"]
    CreateResource -->|"success"| Success["Resource Created"]
    CreateResource -->|"error"| ErrorType{"Error Type"}
    
    ErrorType -->|"quota exceeded"| ResourceQuota["Resource Quota Error"]
    ErrorType -->|"other error"| OtherError["Handle Other Error"]
    
    ResourceQuota -->|"update"| ResourceCeiling["Update Resource Ceiling"]
    ResourceCeiling -->|"calculate"| BackOff["Calculate BackOff Duration"]
    BackOff -->|"retry later"| RetryLater["Retry When Eligible"]
    
    OtherError -->|"check"| Retryable{"Retryable?"}
    Retryable -->|"yes"| SystemRetry["System Retryable Failure"]
    Retryable -->|"no"| Failure["Task Failure"]
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go:195-260)
- [flytepropeller/pkg/controller/nodes/task/backoff/handler.go:128-193](flytepropeller/pkg/controller/nodes/task/backoff/handler.go:128-193)

## Plugin Examples

The Kubernetes Plugin Machinery supports various plugins for different resource types.

### Pod Plugin

The Pod Plugin is the most basic plugin, creating standard Kubernetes pods:

```mermaid
classDiagram
    class plugin {
        +BuildIdentityResource(ctx, meta) client.Object
        +BuildResource(ctx, taskCtx) client.Object
        +GetTaskPhase(ctx, pluginCtx, r) PhaseInfo
        +GetTaskPhaseWithLogs(ctx, pluginCtx, r, logPlugin, logSuffix, extraLogTemplateVars) PhaseInfo
    }
```

Sources:
- [flyteplugins/go/tasks/plugins/k8s/pod/plugin.go:39-41](flyteplugins/go/tasks/plugins/k8s/pod/plugin.go:39-41)
- [flyteplugins/go/tasks/plugins/k8s/pod/plugin.go:44-138](flyteplugins/go/tasks/plugins/k8s/pod/plugin.go:44-138)

### Spark Plugin

The Spark Plugin demonstrates a more complex plugin for SparkApplication custom resources:

```mermaid
classDiagram
    class sparkResourceHandler {
        +GetProperties() k8s.PluginProperties
        +BuildResource(ctx, taskCtx) client.Object
        +BuildIdentityResource(ctx, meta) client.Object
        +GetTaskPhase(ctx, pluginCtx, resource) PhaseInfo
    }
```

Key components:
- `createSparkApplication`: Creates SparkApplication custom resource
- `createDriverSpec`: Configures Spark driver
- `createExecutorSpec`: Configures Spark executors
- `getSparkConfig`: Manages Spark configuration

Sources:
- [flyteplugins/go/tasks/plugins/k8s/spark/spark.go:40-57](flyteplugins/go/tasks/plugins/k8s/spark/spark.go:40-57)
- [flyteplugins/go/tasks/plugins/k8s/spark/spark.go:59-89](flyteplugins/go/tasks/plugins/k8s/spark/spark.go:59-89)
- [flyteplugins/go/tasks/plugins/k8s/spark/spark.go:91-135](flyteplugins/go/tasks/plugins/k8s/spark/spark.go:91-135)

## Plugin Registration and Integration

The plugin system integrates with the Flyte workflow executor:

```mermaid
graph TD
    subgraph "Plugin Registration"
        PluginRegistry["Plugin Registry"]
        CorePlugins["Core Plugins"]
        K8sPlugins["K8s Plugins"]
        
        CorePlugins -->|"register"| PluginRegistry
        K8sPlugins -->|"register"| PluginRegistry
    end
    
    subgraph "Plugin Initialization"
        ConfigSystem["Config System"]
        PluginConfig["K8sPluginConfig"]
        BackOffController["BackoffController"]
        MonitorIndex["ResourceMonitorIndex"]
        
        ConfigSystem -->|"provides"| PluginConfig
        PluginRegistry -->|"create"| PluginManager["PluginManager"]
        BackOffController -->|"used by"| PluginManager
        MonitorIndex -->|"used by"| PluginManager
    end
```

Sources:
- [flytepropeller/pkg/controller/nodes/task/plugin_config.go:20-98](flytepropeller/pkg/controller/nodes/task/plugin_config.go:20-98)
- [flyteplugins/go/tasks/plugins/k8s/spark/spark.go:557-569](flyteplugins/go/tasks/plugins/k8s/spark/spark.go:557-569)

## Summary

The Kubernetes Plugin Machinery provides a powerful and flexible framework for executing Flyte tasks on Kubernetes. Key features include:

1. A plugin-based architecture that supports various resource types
2. Extensive resource customization capabilities
3. Sophisticated error handling and backoff mechanisms
4. Support for extended resources like GPUs
5. Templating for consistent resource configurations

This system enables Flyte to leverage the full power of Kubernetes while providing a consistent and reliable task execution environment.

Sources:
- [flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go](flytepropeller/pkg/controller/nodes/task/k8s/plugin_manager.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/pod_helper.go)
- [flyteplugins/go/tasks/pluginmachinery/flytek8s/config/config.go](flyteplugins/go/tasks/pluginmachinery/flytek8s/config/config.go)