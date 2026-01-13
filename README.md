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

## Diagram

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
