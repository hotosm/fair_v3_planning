# Requirements

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/design/diagrams/mlops-requirements.drawio.png](docs/design/diagrams/mlops-requirements.drawio.png)
- [docs/design/requirements.md](docs/design/requirements.md)

</details>



This page documents the functional requirements for the EOEPCA MLOps Building Block. These requirements define the capabilities that must be provided by the Model Trainer, Training Data Manager, and MLOps UI components. The requirements establish the behavioral specifications that the system architecture and implementation must satisfy.

For information about how these requirements translate into practical user workflows and scenarios, see [Use Cases](#2.2). For details on the specific components that implement these requirements, see [GitLab](#3.1), [SharingHub](#3.2), and [MLflow SharingHub](#3.3).

**Sources:** [docs/design/requirements.md:1-35]()

---

## Requirements Overview

The MLOps Building Block requirements are organized into three functional categories, each addressing a distinct aspect of the machine learning operations lifecycle:

| Category | Code Range | Component Mapping | Primary Responsibility |
|----------|------------|-------------------|------------------------|
| **Model Trainer** | BR086-BR094 | MLflow SharingHub | Experiment tracking, model registry, version management |
| **Training Data Manager** | BR095-BR096 | GitLab + DVC + SharingHub | Dataset versioning, metadata management, discovery |
| **MLOps UI** | BR097 | GitLab UI + SharingHub UI + MLflow UI | Web-based access to all platform capabilities |

The requirements are designed to support a complete MLOps workflow from dataset preparation through model training, versioning, and publication for reuse.

**Sources:** [docs/design/requirements.md:13-35]()

---

## Requirements to Component Mapping

The following diagram illustrates how the functional requirements map to the three core components of the MLOps Building Block:

```mermaid
graph TB
    subgraph "Functional Requirements"
        MT["Model Trainer Requirements<br/>BR086-BR094"]
        TDM["Training Data Manager Requirements<br/>BR095-BR096"]
        UI["MLOps UI Requirements<br/>BR097"]
    end
    
    subgraph "Implementation Components"
        MLF["MLflow SharingHub<br/>docker.io/eoepca/mlflow-sharinghub"]
        GL["GitLab<br/>gitlab.com/gitlab-org/gitlab"]
        DVC["DVC<br/>Data Version Control"]
        SH["SharingHub<br/>gitlab.com/eoepca/sharinghub"]
    end
    
    subgraph "Storage Backend"
        PG["PostgreSQL<br/>mlflow_backend_store"]
        S3["S3 Object Storage<br/>mlflow_artifacts"]
        GLPG["GitLab PostgreSQL<br/>gitlab_database"]
    end
    
    subgraph "API Layer"
        MLFAPI["MLflow Tracking API<br/>/api/2.0/mlflow/*"]
        STACAPI["STAC API<br/>/api/stac/v1/*"]
        GLAPI["GitLab REST API<br/>/api/v4/*"]
    end
    
    MT -->|"BR086-BR092"| MLF
    MT -->|"BR093"| MLF
    MT -->|"BR094"| SH
    
    TDM -->|"BR095"| DVC
    TDM -->|"BR095"| GL
    TDM -->|"BR096"| SH
    
    UI -->|"BR097-1"| MLF
    UI -->|"BR097-2"| GL
    UI -->|"BR097-2"| SH
    
    MLF --> MLFAPI
    MLF --> PG
    MLF --> S3
    
    GL --> GLAPI
    GL --> GLPG
    
    DVC --> GL
    DVC --> S3
    
    SH --> STACAPI
    SH --> GLAPI
```

**Sources:** [docs/design/requirements.md:1-35]()

---

## Model Trainer Requirements (BR086-BR094)

The Model Trainer requirements define capabilities for managing the complete machine learning training lifecycle, including experiment tracking, model versioning, and model registry functionality. These requirements are primarily satisfied by **MLflow SharingHub**.

### Detailed Requirements

| Requirement ID | Description | Implementation Component |
|----------------|-------------|--------------------------|
| **BR086** | Support management of ML training for popular frameworks (TensorFlow, PyTorch, Keras, etc.) | MLflow autologging, MLflow tracking API |
| **BR087** | Support interoperable model representations (ONNX) | MLflow model registry with ONNX format support |
| **BR088** | Support initiation of training runs and capture resultant metrics | MLflow tracking API, `mlflow.log_metric()`, `mlflow.log_param()` |
| **BR089** | Maintain history of runs with parameterization, datasets, and metrics | PostgreSQL backend store, MLflow experiments and runs database |
| **BR090** | Persist model assets in platform storage with workspace integration | S3 artifacts store, MLflow artifact logging |
| **BR091** | Support applying versions to generated models | MLflow model registry, `mlflow.register_model()` |
| **BR092** | Register versioned models in Resource Discovery for sharing | SharingHub STAC API integration, automatic STAC item generation |
| **BR093** | Provide Web UI for training run history management | MLflow UI at `/mlflow` endpoint |
| **BR094** | Support integration as service within Workspace BB | RESTful API, authentication via SharingHub |

**Sources:** [docs/design/requirements.md:13-24]()

---

### Model Trainer Implementation Architecture

The following diagram shows how MLflow SharingHub implements the Model Trainer requirements:

```mermaid
graph TB
    subgraph "User Code"
        TRAIN["Training Script<br/>train.py"]
        CLIENT["MLflow Client<br/>mlflow.start_run()"]
    end
    
    subgraph "MLflow SharingHub Container"
        FLASK["Flask Application<br/>app.py"]
        TRACKING["Tracking Service<br/>/api/2.0/mlflow/runs/*"]
        REGISTRY["Model Registry<br/>/api/2.0/mlflow/registered-models/*"]
        AUTOLOG["Auto-logging<br/>mlflow.autolog()"]
    end
    
    subgraph "Storage Layer"
        PGDB["PostgreSQL<br/>experiments, runs, metrics<br/>registered_models, model_versions"]
        S3STORE["S3 Bucket<br/>model artifacts<br/>ONNX files, pkl files"]
    end
    
    subgraph "Integration Layer"
        SHINT["SharingHub Integration<br/>permission_check()"]
        STACGEN["STAC Item Generator<br/>ml-model extension"]
    end
    
    TRAIN -->|"BR086: Log experiments"| CLIENT
    CLIENT -->|"BR088: POST /runs/create"| TRACKING
    AUTOLOG -->|"BR086: Framework metrics"| TRACKING
    
    TRACKING -->|"BR089: Store run history"| PGDB
    TRACKING -->|"BR090: Store artifacts"| S3STORE
    
    CLIENT -->|"BR091: POST /registered-models/create"| REGISTRY
    REGISTRY -->|"BR091: Version management"| PGDB
    REGISTRY -->|"BR087: Store ONNX models"| S3STORE
    
    REGISTRY -->|"BR092: Auto-publish"| STACGEN
    STACGEN -->|"BR092: Generate STAC item"| SHINT
    
    FLASK -->|"BR093: Serve UI"| MLUI["MLflow Web UI<br/>/mlflow"]
    FLASK -->|"BR094: Check permissions"| SHINT
```

**Sources:** [docs/design/requirements.md:13-24]()

---

## Training Data Manager Requirements (BR095-BR096)

The Training Data Manager requirements define capabilities for version-controlled dataset management with metadata standards compliance. These requirements are satisfied through the combination of **GitLab**, **DVC**, and **SharingHub**.

### Detailed Requirements

| Requirement ID | Description | Implementation Component |
|----------------|-------------|--------------------------|
| **BR095** | Support version-controlled management of training datasets with STAC metadata | GitLab repository versioning, DVC data versioning, SharingHub STAC catalog generation |
| **BR096** | Register versioned ML training data in Resource Discovery for sharing | SharingHub STAC API, GitLab topics filtering (`sharinghub:dataset`) |

**Sources:** [docs/design/requirements.md:25-29]()

---

### Training Data Manager Implementation Architecture

The following diagram illustrates how GitLab, DVC, and SharingHub work together to satisfy Training Data Manager requirements:

```mermaid
graph TB
    subgraph "Dataset Project"
        REPO["GitLab Repository<br/>sen1floods11-dataset"]
        DVCFILE[".dvc Files<br/>data.dvc, train.csv.dvc"]
        METADATA["Metadata Files<br/>README.md, stac_item.json"]
        TOPIC["GitLab Topic<br/>sharinghub:dataset"]
    end
    
    subgraph "Version Control"
        GITLOG["Git Commit History<br/>git log, git tag"]
        DVCTRACK["DVC Tracking<br/>dvc add, dvc push"]
    end
    
    subgraph "Data Storage"
        S3DATA["S3 Bucket<br/>training-data/<br/>*.tif, *.csv, *.npy"]
        DVCCACHE["DVC Cache<br/>.dvc/cache/"]
    end
    
    subgraph "SharingHub Discovery"
        CRAWLER["GitLab Metadata Extractor<br/>get_gitlab_projects()"]
        STACCAT["STAC Catalog<br/>/collections/dataset"]
        STACITEM["STAC Item<br/>eo extension, assets"]
    end
    
    REPO -->|"BR095: Version control"| GITLOG
    REPO -->|"BR095: Add topic"| TOPIC
    REPO -->|"BR095: Commit metadata"| METADATA
    
    DVCFILE -->|"BR095: Track data files"| DVCTRACK
    DVCTRACK -->|"BR095: Push data"| S3DATA
    DVCTRACK -->|"BR095: Local cache"| DVCCACHE
    
    CRAWLER -->|"BR096: Filter by topic"| TOPIC
    CRAWLER -->|"BR096: Extract metadata"| METADATA
    CRAWLER -->|"BR096: Generate catalog"| STACCAT
    
    STACCAT -->|"BR096: Create items"| STACITEM
    STACITEM -->|"BR096: Link to"| S3DATA
    
    GITLOG -.->|"BR095: Version tags"| STACITEM
```

**Sources:** [docs/design/requirements.md:25-29]()

---

## MLOps UI Requirements (BR097)

The MLOps UI requirements define the web-based user interface capabilities for accessing training run history and training data management. These requirements are satisfied through the combination of **MLflow UI**, **GitLab UI**, and **SharingHub Web UI**.

### Detailed Requirements

| Requirement ID | Description | Implementation Component |
|----------------|-------------|--------------------------|
| **BR097-1** | Access training run history for model run management and performance assessment | MLflow UI at `/mlflow` endpoint, experiments dashboard, run comparison views |
| **BR097-2** | Access training data for management, versioning, and metadata editing | GitLab project UI, DVC file browser, SharingHub catalog browser |

**Sources:** [docs/design/requirements.md:30-35]()

---

### MLOps UI Implementation Architecture

The following diagram shows the three user interfaces and their responsibilities:

```mermaid
graph TB
    subgraph "User Access"
        USER["Data Scientist / ML Developer"]
        BROWSER["Web Browser<br/>HTTPS"]
    end
    
    subgraph "MLflow UI"
        MLFUI["MLflow Web Interface<br/>sharinghub.domain/mlflow"]
        EXPVIEW["Experiments View<br/>/experiments/<exp_id>"]
        RUNVIEW["Run Details View<br/>/runs/<run_id>"]
        REGVIEW["Model Registry View<br/>/models/<model_name>"]
        COMPARE["Run Comparison<br/>/compare-runs"]
    end
    
    subgraph "GitLab UI"
        GLUI["GitLab Web Interface<br/>gitlab.domain"]
        PROJVIEW["Project View<br/>/<namespace>/<project>"]
        FILEVIEW["File Browser<br/>/-/tree/main"]
        COMMIT["Commit History<br/>/-/commits/main"]
        TAGS["Tags / Releases<br/>/-/tags"]
    end
    
    subgraph "SharingHub UI"
        SHUI["SharingHub Web Interface<br/>sharinghub.domain"]
        CATVIEW["Catalog Browser<br/>/browse"]
        ITEMVIEW["Item Detail View<br/>/item/<item_id>"]
        SEARCH["Search & Filter<br/>/search"]
        DOWNLOAD["Download Assets<br/>/download"]
    end
    
    USER --> BROWSER
    
    BROWSER -->|"BR097-1: View experiments"| MLFUI
    MLFUI --> EXPVIEW
    MLFUI --> RUNVIEW
    MLFUI --> REGVIEW
    MLFUI --> COMPARE
    
    BROWSER -->|"BR097-2: Manage datasets"| GLUI
    GLUI --> PROJVIEW
    GLUI --> FILEVIEW
    GLUI --> COMMIT
    GLUI --> TAGS
    
    BROWSER -->|"BR097-1 & BR097-2: Discover resources"| SHUI
    SHUI --> CATVIEW
    SHUI --> ITEMVIEW
    SHUI --> SEARCH
    SHUI --> DOWNLOAD
    
    EXPVIEW -.->|"Links to GitLab project"| PROJVIEW
    REGVIEW -.->|"Links to STAC item"| ITEMVIEW
    ITEMVIEW -.->|"Links to GitLab project"| PROJVIEW
```

**Sources:** [docs/design/requirements.md:30-35]()

---

## Requirements Traceability Matrix

The following table provides a complete traceability matrix showing which components and code entities satisfy each requirement:

| Requirement | Component | Key Implementation Details | Configuration Files |
|-------------|-----------|----------------------------|---------------------|
| BR086 | MLflow SharingHub | MLflow autologging, framework integrations (TensorFlow, PyTorch, Keras) | N/A (framework-agnostic) |
| BR087 | MLflow SharingHub | MLflow ONNX model flavor, model serialization | N/A (MLflow built-in) |
| BR088 | MLflow SharingHub | MLflow Tracking API: `/api/2.0/mlflow/runs/create`, `/api/2.0/mlflow/runs/log-metric` | N/A (MLflow API) |
| BR089 | MLflow SharingHub + PostgreSQL | Experiments, runs, metrics, params tables in PostgreSQL backend | `postgresql://mlflow_user:password@postgres:5432/mlflow` |
| BR090 | MLflow SharingHub + S3 | S3 artifacts store, MLflow artifact logging API | `s3://mlflow-artifacts/` |
| BR091 | MLflow SharingHub | MLflow Model Registry: `/api/2.0/mlflow/registered-models/create`, model versioning | N/A (MLflow API) |
| BR092 | MLflow SharingHub + SharingHub | Auto-publish to STAC catalog on model registration, ml-model extension | SharingHub STAC catalog generation |
| BR093 | MLflow SharingHub | Flask application serving MLflow UI at `/mlflow` endpoint | MLflow UI templates |
| BR094 | MLflow SharingHub + SharingHub | RESTful API, permission checking via SharingHub integration | Authentication headers, OIDC integration |
| BR095 | GitLab + DVC + SharingHub | Git version control, DVC data tracking, STAC metadata generation | `.dvc` files, `dvc.yaml`, GitLab topics |
| BR096 | SharingHub | STAC catalog with dataset collection, GitLab topic filtering (`sharinghub:dataset`) | `categories.yaml`, `tags.yaml` |
| BR097-1 | MLflow UI | Web interface for experiments, runs, models at `/mlflow` | MLflow UI Flask routes |
| BR097-2 | GitLab UI + SharingHub UI | GitLab project UI, file browser, SharingHub catalog browser | GitLab Rails UI, SharingHub Flask templates |

**Sources:** [docs/design/requirements.md:1-35]()

---

## Integration Requirements

The requirements are designed to work together as an integrated system. The following diagram illustrates the critical integration points between components:

```mermaid
graph LR
    subgraph "BR086-BR091: Model Training"
        TRAIN["Model Training<br/>Python Script"]
        MLFTRACK["MLflow Tracking<br/>Log Metrics"]
        MLFREG["MLflow Registry<br/>Register Model"]
    end
    
    subgraph "BR092: Discovery Integration"
        AUTOPUB["Auto-publish<br/>on Registration"]
        STACGEN["STAC Item<br/>ml-model Extension"]
    end
    
    subgraph "BR095-BR096: Data Management"
        DVCADD["DVC Add<br/>Track Dataset"]
        GITCOMMIT["Git Commit<br/>Metadata"]
        STACCAT["STAC Catalog<br/>Dataset Collection"]
    end
    
    subgraph "BR093-BR097: UI Access"
        MLFUI["MLflow UI<br/>View Runs"]
        GLUI["GitLab UI<br/>Manage Data"]
        SHUI["SharingHub UI<br/>Browse Catalog"]
    end
    
    TRAIN -->|"Experiment tracking"| MLFTRACK
    MLFTRACK -->|"Model versioning"| MLFREG
    MLFREG -->|"BR092: Trigger"| AUTOPUB
    AUTOPUB -->|"Generate"| STACGEN
    
    DVCADD -->|"Version data"| GITCOMMIT
    GITCOMMIT -->|"BR096: Extract metadata"| STACCAT
    
    STACGEN -->|"Appears in"| SHUI
    STACCAT -->|"Appears in"| SHUI
    
    MLFTRACK -.->|"BR093: View in"| MLFUI
    GITCOMMIT -.->|"BR097-2: View in"| GLUI
    STACGEN -.->|"Links to"| GLUI
    STACCAT -.->|"Links to"| GLUI
```

**Sources:** [docs/design/requirements.md:1-35]()

---

## Summary

The MLOps Building Block requirements establish a comprehensive framework for machine learning operations covering:

- **Model training lifecycle management** (BR086-BR094) through MLflow SharingHub
- **Dataset version control and discovery** (BR095-BR096) through GitLab, DVC, and SharingHub integration
- **Web-based user interfaces** (BR097) providing unified access to all platform capabilities

These requirements are satisfied through the coordinated interaction of three core components: **GitLab** for code and data version control, **SharingHub** for discovery and metadata management via STAC API, and **MLflow SharingHub** for experiment tracking and model registry. The architecture ensures that models and datasets are consistently versioned, properly metadata-tagged, and discoverable through standardized APIs.

For practical examples of these requirements in action, see [Use Cases](#2.2). For deployment instructions, see [Deployment Guide](#5).

**Sources:** [docs/design/requirements.md:1-35]()