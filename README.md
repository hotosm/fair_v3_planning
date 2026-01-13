# fAIr 3.0 Planning

- fAIr is currently tied to AWS and uses a Celery job queue.
- A new version will be migrated to Kubernetes inside AWS,
  with the workflow orchestration handled by a MLOps tool
  such as Flyte.
- Versioning of workflows will be possible, with ML lineage
  tracking, easily accessible models, and consistent metadata
  exposed by a STAC extension.
- In the end, fAIr 3.0 will empower end users to generate
  and iterate on models, creating a marketplace for open
  and fair geo-AI model creation.

See [current architecture](./deepwiki_scrape/hotosm/fAIr/1_Overview.md).
See [planned architecture](./planned_architecture.md).

## Overall Architecture Diagram

```mermaid
flowchart TD
    A0["fAIr AI-Assisted Mapping Application
"]
    A1["Flyte ML Workflow Orchestrator
"]
    A2["MLflow Experiment Tracking & Model Registry
"]
    A3["STAC-MLM Geo-AI Model Metadata Standard
"]
    A4["Kubernetes Cluster
"]
    A5["S3 Object Storage
"]
    A6["ONNX Model Format
"]
    A0 -- "Orchestrates training via" --> A1
    A1 -- "Deploys workloads on" --> A4
    A1 -- "Integrates with" --> A2
    A2 -- "Stores artifacts in" --> A5
    A2 -- "Generates metadata for" --> A3
    A3 -- "Describes model format" --> A6
    A6 -- "Enables client inference for" --> A0
```

## Flyte Integration Diagram

```mermaid
sequenceDiagram
    participant FB as fAIr Backend
    participant FA as Flyte Admin
    participant FP as Flyte Propeller
    participant K8s as Kubernetes Cluster
    participant S3_MLF as S3 / MLflow

    FB->>FA: Submit "Building Detection" Workflow
    Note over FA: Records workflow details in database.
    FA->>FP: Create FlyteWorkflow CRD (Kubernetes Request)
    FP->>K8s: Launch Pod for "Fetch Imagery" Task
    Note over K8s: Pod fetches imagery from S3.
    K8s->>FP: "Fetch Imagery" Task Completed
    FP->>K8s: Launch Pod for "Train AI Model" Task (on GPU node)
    Note over K8s: Pod trains AI, logs with MLflow, uses S3 for data.
    K8s->>FP: "Train AI Model" Task Completed
    FP->>K8s: Launch Pod for "Convert to ONNX" Task
    Note over K8s: Pod converts model.
    K8s->>FP: "Convert to ONNX" Task Completed
    FP->>FA: Report Workflow Success and Results
    Note over FA: Updates execution status.
    FA->>FB: Workflow Done! (with ONNX model path)
```
