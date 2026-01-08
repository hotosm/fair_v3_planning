# Array Tasks

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [flytepropeller/pkg/apis/flyteworkflow/v1alpha1/array.go](flytepropeller/pkg/apis/flyteworkflow/v1alpha1/array.go)
- [flytepropeller/pkg/apis/flyteworkflow/v1alpha1/mocks/ExecutableArrayNode.go](flytepropeller/pkg/apis/flyteworkflow/v1alpha1/mocks/ExecutableArrayNode.go)
- [flytepropeller/pkg/compiler/transformers/k8s/node.go](flytepropeller/pkg/compiler/transformers/k8s/node.go)
- [flytepropeller/pkg/compiler/transformers/k8s/node_test.go](flytepropeller/pkg/compiler/transformers/k8s/node_test.go)
- [flytepropeller/pkg/controller/config/config.go](flytepropeller/pkg/controller/config/config.go)
- [flytepropeller/pkg/controller/config/config_flags.go](flytepropeller/pkg/controller/config/config_flags.go)
- [flytepropeller/pkg/controller/config/config_flags_test.go](flytepropeller/pkg/controller/config/config_flags_test.go)
- [flytepropeller/pkg/controller/nodes/array/handler.go](flytepropeller/pkg/controller/nodes/array/handler.go)
- [flytepropeller/pkg/controller/nodes/array/handler_test.go](flytepropeller/pkg/controller/nodes/array/handler_test.go)
- [flytepropeller/pkg/controller/nodes/array/node_execution_context.go](flytepropeller/pkg/controller/nodes/array/node_execution_context.go)
- [flytepropeller/pkg/controller/nodes/array/node_execution_context_test.go](flytepropeller/pkg/controller/nodes/array/node_execution_context_test.go)

</details>



This document explains how array nodes are processed for parallel execution of tasks within Flyte. Array tasks allow you to execute the same task over a collection of inputs in parallel, providing a simple and efficient way to process data at scale. For details about individual task execution, see [Task Execution](#3.3).

## Overview

Array tasks enable a form of map-reduce pattern where the same operation is performed on each item in an input collection. Flyte manages the fan-out execution, monitoring, and collection of results with configurable parallelism, retry behavior, and success criteria.

```mermaid
graph TD
    subgraph "Array Task Execution"
        InputCollection["Input Collection"] --> SplitInputs["Split Inputs"]
        SplitInputs --> |"Item 1"| Task1["SubNode Task 1"]
        SplitInputs --> |"Item 2"| Task2["SubNode Task 2"]
        SplitInputs --> |"Item 3"| Task3["SubNode Task 3"]
        SplitInputs --> |"Item n"| TaskN["SubNode Task n"]
        
        Task1 --> |"Result 1"| GatherOutputs["Gather Outputs"]
        Task2 --> |"Result 2"| GatherOutputs
        Task3 --> |"Result 3"| GatherOutputs
        TaskN --> |"Result n"| GatherOutputs
        
        GatherOutputs --> OutputCollection["Output Collection"]
    end
```

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:54-764](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)

## Array Node Structure

An array node is defined by an `ArrayNodeSpec` that contains:

- **SubNodeSpec**: The specification of the task to be executed for each item in the input collection
- **Parallelism**: Optional limit on the number of concurrent executions
- **MinSuccesses**: Optional minimum number of successful executions required for the array to be considered successful
- **MinSuccessRatio**: Optional minimum ratio of successful executions (alternative to MinSuccesses)
- **BoundInputs**: Optional list of inputs that should not be split (passed directly to each task)

```mermaid
classDiagram
    class ArrayNodeSpec {
        +NodeSpec SubNodeSpec
        +uint32* Parallelism
        +uint32* MinSuccesses
        +float32* MinSuccessRatio
        +string[] BoundInputs
        +GetSubNodeSpec() *NodeSpec
        +GetParallelism() *uint32
        +GetMinSuccesses() *uint32
        +GetMinSuccessRatio() *float32
        +GetBoundInputs() []string
    }
```

Sources: [flytepropeller/pkg/apis/flyteworkflow/v1alpha1/array.go:1-29](https://github.com/flyteorg/flyte/flytepropeller/pkg/apis/flyteworkflow/v1alpha1/array.go), [flytepropeller/pkg/compiler/transformers/k8s/node.go:178-208](https://github.com/flyteorg/flyte/flytepropeller/pkg/compiler/transformers/k8s/node.go)

## Input Handling

When an array task executes, the inputs are processed as follows:

1. For each input that is a collection and not in the `BoundInputs` list, items are distributed to individual subtasks
2. Each subtask receives a single element from each collection input
3. Non-collection inputs and bound inputs are passed directly to each subtask

If all inputs are collections of the same length, each subtask receives the corresponding item from each collection. For example, with collections `[a, b, c]` and `[x, y, z]`, the first subtask receives `a` and `x`, the second receives `b` and `y`, and so on.

```mermaid
graph TD
    subgraph "Input Processing"
        Input["Input LiteralMap"] --> |"Collection inputs"| Distribute["Distribute items to subtasks"]
        Input --> |"Scalar inputs"| CopyToAll["Copy to all subtasks"]
        Input --> |"Bound collection inputs"| CopyToAll
        
        Distribute --> SubTask1["SubTask 1 inputs"]
        Distribute --> SubTask2["SubTask 2 inputs"]
        Distribute --> SubTaskN["SubTask n inputs"]
        
        CopyToAll --> SubTask1
        CopyToAll --> SubTask2
        CopyToAll --> SubTaskN
    end
```

Sources: [flytepropeller/pkg/controller/nodes/array/node_execution_context.go:15-50](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/node_execution_context.go), [flytepropeller/pkg/controller/nodes/array/handler.go:783-786](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)

## Parallelism Control

Array tasks offer flexible parallelism control through several mechanisms:

### Parallelism Behaviors

These behaviors are controlled by the `DefaultParallelismBehavior` configuration:

- **Hybrid** (`ParallelismBehaviorHybrid`): Uses exact parallelism specified by the array node
- **Unlimited** (`ParallelismBehaviorUnlimited`): Maximum possible parallelism for nil/0 values
- **Workflow** (`ParallelismBehaviorWorkflow`): Uses workflow-level parallelism for nil/0 values

### Configuring Parallelism

Parallelism is determined through several factors:

1. Node-specific parallelism (if specified in the `ArrayNode` definition)
2. Workflow-level parallelism limit
3. System-level configuration settings

The actual parallelism is calculated based on the behavior mode and the available parallelism in the workflow.

```mermaid
flowchart TD
    Start["Determine parallelism"] --> HasNodeParallelism{"Has node\nparallelism?"}
    
    HasNodeParallelism -->|"Yes"| UseNodeParallelism["Use node parallelism"]
    HasNodeParallelism -->|"No"| CheckBehavior{"Check default\nbehavior"}
    
    CheckBehavior -->|"Hybrid"| UseWorkflowParallelism["Use workflow parallelism"]
    CheckBehavior -->|"Unlimited"| UseUnlimited["Use maximum available"]
    CheckBehavior -->|"Workflow"| UseWorkflowParallelism
    
    UseNodeParallelism --> EnforceLimit["Enforce workflow maximum"]
    UseWorkflowParallelism --> EnforceLimit
    UseUnlimited --> EnforceLimit
    
    EnforceLimit --> End["Execute with chosen parallelism"]
```

Sources: [flytepropeller/pkg/controller/config/config.go:330-347](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/config/config.go), [flytepropeller/pkg/controller/nodes/array/handler.go:302-359](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)

## Success Criteria

Array tasks provide two ways to define success criteria:

1. **MinSuccesses**: Absolute number of subtasks that must succeed
2. **MinSuccessRatio**: Fraction of subtasks that must succeed

If no success criteria are specified, all subtasks must succeed for the array task to be considered successful.

The system calculates whether successful completion is still possible:

```mermaid
graph TD
    Start["Evaluate array status"] --> Calculate["Calculate counts:\n- Success count\n- Failed count\n- Running count"]
    
    Calculate --> CheckMinSuccesses{"Can reach\nminimum successes?"}
    
    CheckMinSuccesses -->|"No"| MarkFailing["Mark ArrayNode as failing"]
    CheckMinSuccesses -->|"Yes"| CheckIfComplete{"All tasks complete?"}
    
    CheckIfComplete -->|"No"| MarkExecuting["Continue executing"]
    CheckIfComplete -->|"Yes"| CheckEnough{"Enough successes?"}
    
    CheckEnough -->|"Yes"| MarkSucceeding["Mark ArrayNode as succeeding"]
    CheckEnough -->|"No"| MarkFailing
```

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:437-472](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)

## Execution Lifecycle

The array task goes through several phases during execution:

1. **ArrayNodePhaseNone**: Initial state
2. **ArrayNodePhaseExecuting**: Tasks are being executed
3. **ArrayNodePhaseFailing**: Failure conditions have been met
4. **ArrayNodePhaseSucceeding**: Success conditions have been met

### State Management

The array handler maintains state for each subtask using compact bit arrays for efficiency:

- **SubNodePhases**: Current phase of each subnode
- **SubNodeTaskPhases**: Task execution phase of each subnode
- **SubNodeRetryAttempts**: Number of retry attempts for each subnode
- **SubNodeSystemFailures**: Number of system failures for each subnode
- **SubNodeDeltaTimestamps**: Time difference from parent node start

```mermaid
stateDiagram-v2
    [*] --> ArrayNodePhaseNone
    
    ArrayNodePhaseNone --> ArrayNodePhaseExecuting: Initialize state
    
    ArrayNodePhaseExecuting --> ArrayNodePhaseExecuting: Process subtasks
    ArrayNodePhaseExecuting --> ArrayNodePhaseFailing: Not enough successes possible
    ArrayNodePhaseExecuting --> ArrayNodePhaseSucceeding: Min successes reached & all tasks done
    
    ArrayNodePhaseFailing --> [*]: Report failure
    
    ArrayNodePhaseSucceeding --> [*]: Gather outputs & report success
```

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:198-645](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)

## Output Aggregation

When an array task completes successfully, it gathers outputs from all subtasks and combines them into collections:

1. For each output variable from the subtasks, a collection is created
2. Each collection contains the corresponding output from each successful subtask
3. For failed subtasks (when using MinSuccesses or MinSuccessRatio), nil values are used
4. The output collections maintain the same order as the input collections

The combined outputs are then stored in the `outputs.pb` file in the node's output directory.

```mermaid
graph TD
    subgraph "Output Aggregation"
        SubTask1["SubTask 1 Output"] --> |"value1"| OutputVar1["Collection for Output 1"]
        SubTask2["SubTask 2 Output"] --> |"value2"| OutputVar1
        SubTask3["SubTask 3 Output"] --> |"value3"| OutputVar1
        
        SubTask1 --> |"result1"| OutputVar2["Collection for Output 2"]
        SubTask2 --> |"result2"| OutputVar2
        SubTask3 --> |"result3"| OutputVar2
        
        OutputVar1 --> OutputLiteralMap["Combined Output LiteralMap"]
        OutputVar2 --> OutputLiteralMap
        
        OutputLiteralMap --> Store["Store in output.pb"]
    end
```

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:496-626](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)

## Configuration

Array tasks can be configured through several options in the Flyte configuration:

| Configuration Option | Default | Description |
|----------------------|---------|-------------|
| `EventVersion` | 0 | Controls the event format for array nodes (0=legacy, 1=new) |
| `DefaultParallelismBehavior` | `ParallelismBehaviorUnlimited` | Default behavior for handling parallelism |
| `UseMapPluginLogs` | false | Use map plugin logs for subnode log links |
| `MaxTaskPhaseVersionAttempts` | 3 | Maximum retries for handling task phase version conflicts |

Sources: [flytepropeller/pkg/controller/config/config.go:124-129](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/config/config.go)

## Implementation Architecture

The array task execution is handled by the `arrayNodeHandler` which implements the `interfaces.NodeHandler` interface:

```mermaid
classDiagram
    class NodeHandler {
        <<interface>>
        +Handle(context.Context, NodeExecutionContext) (Transition, error)
        +Abort(context.Context, NodeExecutionContext, string) error
        +Finalize(context.Context, NodeExecutionContext) error
        +FinalizeRequired() bool
        +Setup(context.Context, SetupContext) error
    }
    
    class arrayNodeHandler {
        -eventConfig *config.EventConfig
        -literalOffloadingConfig config.LiteralOffloadingConfig
        -gatherOutputsRequestChannel chan *gatherOutputsRequest
        -metrics metrics
        -nodeExecutionRequestChannel chan *nodeExecutionRequest
        -nodeExecutor interfaces.Node
        -pluginStateBytesNotStarted []byte
        -pluginStateBytesStarted []byte
        +Handle(context.Context, NodeExecutionContext) (Transition, error)
        +Abort(context.Context, NodeExecutionContext, string) error
        +Finalize(context.Context, NodeExecutionContext) error
        +FinalizeRequired() bool
        +Setup(context.Context, SetupContext) error
    }
    
    NodeHandler <|-- arrayNodeHandler
```

The handler uses worker pools to process subtask executions and gather outputs in parallel:

```mermaid
graph TD
    subgraph "Worker Architecture"
        ArrayHandler["arrayNodeHandler"] --> |"create"| Workers["Worker Pool\n(NodeExecutionWorkerCount)"]
        
        Workers --> |"process"| NodeExecChannel["nodeExecutionRequestChannel"]
        Workers --> |"process"| GatherOutputsChannel["gatherOutputsRequestChannel"]
        
        NodeExecChannel --> |"send requests"| NodeExec["Node Execution Requests"]
        GatherOutputsChannel --> |"send requests"| OutputGather["Output Gathering Requests"]
        
        NodeExec --> |"results"| ResponseChannel["Response Channels"]
        OutputGather --> |"results"| ResponseChannel
    end
```

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:54-764](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go), [flytepropeller/pkg/controller/config/config.go:181-183](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/config/config.go)

## Special Handling

### Literal Offloading

For large output collections, the array handler supports offloading literals to external storage when they exceed a configured size threshold. This prevents overloading the Kubernetes data store with large outputs:

```mermaid
flowchart TD
    Start["Process output collection"] --> CheckSize{"Is output size > threshold?"}
    
    CheckSize -->|"Yes"| Offload["Offload to blob storage"]
    CheckSize -->|"No"| StoreInline["Store inline"]
    
    Offload --> UpdateLiteral["Update literal with reference"]
    StoreInline --> UpdateLiteral
    
    UpdateLiteral --> Save["Save to outputs.pb"]
```

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:608-618](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go), [flytepropeller/pkg/controller/config/config.go:186-193](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/config/config.go)

## Error Handling

Array tasks provide robust error handling mechanisms:

1. **Retries**: Individual subtasks can be retried based on retry configuration
2. **Partial Success**: Array tasks can succeed even with some failed subtasks (using MinSuccesses or MinSuccessRatio)
3. **Error Aggregation**: Error messages from failed subtasks are aggregated for easy diagnosis

When an array task fails, the aggregated error message includes information about which subtasks failed and why.

Sources: [flytepropeller/pkg/controller/nodes/array/handler.go:362-426](https://github.com/flyteorg/flyte/flytepropeller/pkg/controller/nodes/array/handler.go)