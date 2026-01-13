# Architecture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/design/architecture.md](docs/design/architecture.md)
- [docs/design/diagrams/mlops-archi.drawio.png](docs/design/diagrams/mlops-archi.drawio.png)
- [docs/design/diagrams/mlops-overview.drawio.png](docs/design/diagrams/mlops-overview.drawio.png)
- [docs/index.md](docs/index.md)

</details>



## Purpose and Scope

This document describes the overall system architecture of the EOEPCA MLOps Building Block, including its core components, their relationships, and deployment patterns. It covers the high-level design principles, component integration patterns, data flow, and security architecture.

For specific deployment instructions, see [Deployment Guide](#5). For detailed configuration options, see [Configuration Reference](#6). For API usage details, see [API Reference](#7).

Sources: [docs/design/architecture.md:1-76](), [docs/index.md:1-45]()

## System Overview

The MLOps Building Block provides a complete platform for training, managing, and discovering machine learning models within the EOEPCA ecosystem. The architecture is built on three core components that work together to provide an integrated MLOps solution:

1. **GitLab** - Provides version control, project management, and CI/CD capabilities
2. **SharingHub** - Enables discovery and collaboration through a STAC API catalog
3. **MLflow SharingHub** - Manages experiment tracking and model registry

These components are deployed on Kubernetes and share common infrastructure services including S3 object storage, PostgreSQL databases, and centralized authentication through Keycloak.

```mermaid
graph TB
    subgraph "User Personas"
        MLDev["ML Developer"]
        DataSci["Data Scientist"]
        MLUser["ML User / Operator"]
    end
    
    subgraph "Core Components"
        GL["GitLab<br/>Code & CI/CD"]
        SH["SharingHub<br/>Discovery Platform"]
        MLF["MLflow SharingHub<br/>Experiment Tracking"]
    end
    
    subgraph "Infrastructure Services"
        S3["S3 Object Storage<br/>Artifacts & Data"]
        PG["PostgreSQL<br/>Metadata Store"]
        KC["Keycloak<br/>Identity Provider"]
    end
    
    subgraph "APIs & Interfaces"
        STACAPI["STAC API"]
        MLFAPI["MLflow Tracking API"]
        WebUI["Web Interface"]
    end
    
    MLDev -->|"1. Browse & Train"| WebUI
    DataSci -->|"2. Publish Datasets"| WebUI
    MLUser -->|"3. Download Models"| STACAPI
    
    WebUI --> SH
    WebUI --> MLF
    WebUI --> GL
    
    SH -->|"Exposes"| STACAPI
    SH -->|"Extracts metadata from"| GL
    
    MLF -->|"Provides"| MLFAPI
    MLF -->|"Checks permissions via"| SH
    MLF -->|"Stores metadata"| PG
    MLF -->|"Stores artifacts"| S3
    
    GL -->|"Authenticates via OIDC"| KC
    GL -->|"Stores backups"| S3
    
    SH -->|"Authenticates via OAuth"| GL
```

**Diagram: High-Level System Architecture**

Sources: [docs/design/architecture.md:6-29](), [docs/index.md:16-38]()

## Core Components

### GitLab

GitLab serves as the foundation of the MLOps platform, providing version control, project organization, and CI/CD capabilities. Projects in GitLab are tagged with specific topics (e.g., `sharinghub:aimodel`, `sharinghub:dataset`) to categorize them for discovery through SharingHub.

**Key Responsibilities:**
- Source code version control using Git
- Project metadata storage (topics, tags, descriptions)
- CI/CD pipeline execution
- Large File Storage (LFS) for model artifacts
- Authentication gateway via OIDC integration with Keycloak

**Integration Points:**
- SharingHub extracts project metadata via GitLab API
- MLflow can be integrated directly into GitLab projects
- Backup storage in S3
- User authentication delegated to Keycloak

```mermaid
graph LR
    subgraph "GitLab Components"
        GWS["gitlab-webservice<br/>pods"]
        GSK["gitlab-sidekiq<br/>background jobs"]
        GKAS["gitlab-kas<br/>agent server"]
        GPG["gitlab-postgresql<br/>database"]
        GRED["gitlab-redis<br/>cache"]
    end
    
    subgraph "External Storage"
        S3["S3 Buckets<br/>gitlab-backups<br/>gitlab-lfs"]
    end
    
    subgraph "Authentication"
        KC["Keycloak<br/>OIDC Provider"]
    end
    
    USER["Users"] -->|"HTTPS"| GWS
    GWS --> GPG
    GWS --> GRED
    GWS -->|"OAuth/OIDC"| KC
    GSK --> GPG
    GSK --> S3
    GWS -->|"LFS Objects"| S3
    GKAS -->|"Agent connections"| GWS
```

**Diagram: GitLab Component Architecture**

Sources: [docs/design/architecture.md:52-76]()

### SharingHub

SharingHub is a lightweight discovery and collaboration platform deployed on top of GitLab. It dynamically extracts metadata from GitLab projects and exposes them through a standardized STAC API, making AI models and datasets discoverable.

**Key Responsibilities:**
- Dynamic STAC catalog generation from GitLab projects
- STAC API implementation for standardized discovery
- OAuth-based authentication against GitLab
- Permission checking for private resources
- Metadata extraction and STAC item creation
- Integration with S3 for asset storage

**STAC Catalog Structure:**

The SharingHub generates a STAC catalog with the following structure:
- **Root Catalog**: Single root with ID `gitlab-cs`
- **Collections**: Mapped from GitLab topics (e.g., `sharinghub:aimodel` → AI Model Collection)
- **Items**: One STAC item per GitLab project, with appropriate STAC extensions

```mermaid
graph TB
    subgraph "STAC Catalog Hierarchy"
        ROOT["STAC Root Catalog<br/>id: gitlab-cs"]
        
        subgraph "Collections (mapped from GitLab topics)"
            COLL_AI["AI Model Collection<br/>gitlab_topic: sharinghub:aimodel"]
            COLL_DS["Dataset Collection<br/>gitlab_topic: sharinghub:dataset"]
            COLL_PROC["Processor Collection<br/>gitlab_topic: sharinghub:processor"]
        end
        
        subgraph "STAC Items (one per project)"
            ITEM1["flood-model<br/>ml-model extension<br/>ONNX assets"]
            ITEM2["sen1floods11-dataset<br/>eo extension<br/>GeoTIFF assets"]
            ITEM3["wine-quality-model<br/>ml-model extension<br/>CSV + ONNX"]
        end
        
        ROOT --> COLL_AI
        ROOT --> COLL_DS
        ROOT --> COLL_PROC
        
        COLL_AI -->|"topic filter"| ITEM1
        COLL_AI -->|"topic filter"| ITEM3
        COLL_DS -->|"topic filter"| ITEM2
    end
    
    subgraph "GitLab Projects"
        GP1["Project: flood-model<br/>Topic: sharinghub:aimodel"]
        GP2["Project: sen1floods11<br/>Topic: sharinghub:dataset"]
        GP3["Project: wine-quality<br/>Topic: sharinghub:aimodel"]
    end
    
    GP1 -.metadata extraction.-> ITEM1
    GP2 -.metadata extraction.-> ITEM2
    GP3 -.metadata extraction.-> ITEM3
```

**Diagram: STAC Catalog Structure and GitLab Mapping**

Sources: [docs/design/architecture.md:52-76](), [docs/index.md:22-28]()

### MLflow SharingHub

MLflow SharingHub is a customized MLflow deployment that integrates with SharingHub for permission management and with S3 for artifact storage. It provides experiment tracking and a model registry.

**Key Responsibilities:**
- Experiment tracking API for logging metrics, parameters, and artifacts
- Model registry for versioning and staging models
- Permission checking delegated to SharingHub
- PostgreSQL backend for metadata storage
- S3 artifact store for model files and training artifacts
- Integration with SharingHub for automatic STAC item creation

**Architecture:**

```mermaid
graph TB
    subgraph "MLflow SharingHub Components"
        MLFSERVER["mlflow-sharinghub<br/>tracking server"]
        MLFPG["mlflow-postgresql<br/>backend store"]
    end
    
    subgraph "Storage"
        S3["S3 Bucket<br/>mlflow-artifacts<br/>models, metrics, logs"]
    end
    
    subgraph "Integration"
        SH["SharingHub<br/>permission checking"]
    end
    
    CLIENT["MLflow Client<br/>Python SDK"] -->|"MLflow Tracking API"| MLFSERVER
    
    MLFSERVER -->|"Check permissions"| SH
    MLFSERVER -->|"Store metadata<br/>experiments, runs, metrics"| MLFPG
    MLFSERVER -->|"Store artifacts<br/>models, plots, logs"| S3
    
    MLFSERVER -->|"Auto-link registered models"| SH
    SH -.->|"Creates STAC items"| STACITEM["STAC Item with<br/>ml-model extension"]
```

**Diagram: MLflow SharingHub Integration**

Sources: [docs/design/architecture.md:52-76]()

## Data Flow and Integration Patterns

### End-to-End Model Training Workflow

The following diagram illustrates the complete lifecycle of training and publishing a machine learning model:

```mermaid
sequenceDiagram
    participant Dev as ML Developer
    participant GL as GitLab
    participant SH as SharingHub
    participant MLF as MLflow SharingHub
    participant S3 as S3 Storage
    
    Note over Dev,S3: 1. Project Setup
    Dev->>GL: Create project with topic sharinghub:aimodel
    GL-->>SH: Webhook notification (project created)
    SH->>GL: Extract metadata via API
    SH-->>Dev: Project appears in STAC catalog
    
    Note over Dev,S3: 2. Model Training
    Dev->>SH: Browse datasets via STAC API
    Dev->>MLF: Initialize MLflow tracking (mlflow.set_tracking_uri)
    Dev->>MLF: Log parameters (mlflow.log_param)
    Dev->>MLF: Log metrics (mlflow.log_metric)
    MLF->>S3: Store training artifacts
    MLF->>SH: Check project permissions
    
    Note over Dev,S3: 3. Model Registration
    Dev->>MLF: Register model (mlflow.register_model)
    MLF->>S3: Store model artifacts (ONNX file)
    MLF->>SH: Create/update STAC item with ml-model extension
    SH-->>Dev: Model discoverable in catalog
```

**Diagram: Model Training and Registration Flow**

Sources: [docs/design/architecture.md:10-28]()

### STAC Item Creation and Metadata Extraction

SharingHub dynamically generates STAC items from GitLab projects using the following process:

```mermaid
flowchart TD
    START["GitLab Project<br/>with sharinghub topic"] --> WEBHOOK["GitLab Webhook<br/>project update event"]
    
    WEBHOOK --> EXTRACT["SharingHub extracts metadata<br/>- Project name/description<br/>- Topics and tags<br/>- README content<br/>- LFS objects"]
    
    EXTRACT --> MAP["Map to STAC Item<br/>- id: project path<br/>- title: project name<br/>- description: README<br/>- assets: LFS files"]
    
    MAP --> EXT{"Determine STAC Extensions"}
    
    EXT -->|"Topic: aimodel"| MLEXT["Add ml-model extension<br/>- ml-model:architecture<br/>- ml-model:training-processor-type<br/>- ml-model:training-os<br/>- ml-model:memory_size"]
    
    EXT -->|"Topic: dataset"| EOEXT["Add eo extension<br/>- eo:bands<br/>- eo:gsd"]
    
    MLEXT --> ASSETS["Generate asset links<br/>- model files (ONNX)<br/>- metadata files<br/>- documentation"]
    
    EOEXT --> ASSETS
    
    ASSETS --> PUBLISH["Publish STAC Item<br/>Available via STAC API"]
    
    PUBLISH --> MLF_CHECK{"MLflow model<br/>registered?"}
    
    MLF_CHECK -->|"Yes"| LINK["Auto-link MLflow model<br/>Add mlflow:run_id property"]
    MLF_CHECK -->|"No"| END["STAC Item available"]
    LINK --> END
```

**Diagram: STAC Item Creation Process**

Sources: [docs/design/architecture.md:52-76]()

## Deployment Architecture

The MLOps Building Block is deployed on Kubernetes using ArgoCD for GitOps-style continuous deployment. All components are namespaced for isolation and share common infrastructure services.

### Kubernetes Architecture

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "argocd namespace"
            ARGO["ArgoCD Applications<br/>- gitlab-app.yaml<br/>- sharinghub-app.yaml<br/>- mlflow-sharinghub-app.yaml"]
        end
        
        subgraph "gitlab namespace"
            GLPODS["GitLab Pods<br/>- webservice<br/>- sidekiq<br/>- kas"]
            GLPG["gitlab-postgresql<br/>StatefulSet"]
            GLREDIS["gitlab-redis<br/>StatefulSet"]
            GLINGRESS["Ingress<br/>gitlab.domain"]
        end
        
        subgraph "sharinghub namespace"
            SHPOD["sharinghub<br/>Deployment"]
            SHINGRESS["Ingress<br/>sharinghub.domain"]
        end
        
        subgraph "mlflow namespace"
            MLFPOD["mlflow-sharinghub<br/>Deployment"]
            MLFPG["mlflow-postgresql<br/>StatefulSet"]
            MLFINGRESS["Ingress<br/>sharinghub.domain/mlflow"]
        end
        
        subgraph "cert-manager namespace"
            CERT["cert-manager<br/>TLS automation"]
        end
        
        subgraph "Shared Infrastructure"
            NGINX["NGINX Ingress Controller"]
        end
    end
    
    subgraph "External Services"
        S3EXT["S3 Storage Provider<br/>- gitlab-lfs<br/>- gitlab-backups<br/>- mlflow-artifacts"]
        KCEXT["Keycloak<br/>OIDC Provider"]
        HELM["Helm Repositories<br/>- charts.gitlab.io<br/>- csgroup-oss"]
    end
    
    ARGO -->|"Syncs from Git"| GLPODS
    ARGO -->|"Syncs from Git"| SHPOD
    ARGO -->|"Syncs from Git"| MLFPOD
    ARGO -.->|"Pulls charts"| HELM
    
    GLINGRESS --> NGINX
    SHINGRESS --> NGINX
    MLFINGRESS --> NGINX
    
    NGINX -->|"Requests certs"| CERT
    
    GLPODS --> GLPG
    GLPODS --> GLREDIS
    GLPODS --> S3EXT
    GLPODS -->|"OIDC auth"| KCEXT
    
    SHPOD --> GLPODS
    SHPOD --> S3EXT
    
    MLFPOD --> MLFPG
    MLFPOD --> S3EXT
    MLFPOD -->|"Permission check"| SHPOD
```

**Diagram: Kubernetes Deployment Architecture**

Sources: [docs/design/architecture.md:52-76]()

### Namespace and Resource Organization

Each component is deployed in its own Kubernetes namespace to provide isolation:

| Namespace | Components | Purpose |
|-----------|-----------|---------|
| `argocd` | ArgoCD Applications | GitOps deployment management |
| `gitlab` | GitLab webservice, sidekiq, kas, postgresql, redis | Version control and CI/CD |
| `sharinghub` | SharingHub application | Discovery and STAC API |
| `mlflow` | MLflow server, postgresql | Experiment tracking |
| `cert-manager` | cert-manager controllers | TLS certificate automation |
| Shared | NGINX Ingress Controller | Traffic routing |

**Persistent Storage:**
- PostgreSQL databases use persistent volumes for data durability
- GitLab and MLflow artifacts stored in external S3 buckets
- Configuration managed via Kubernetes Secrets and ConfigMaps

Sources: [docs/design/architecture.md:52-76]()

## Security and Authentication Architecture

The security architecture is built on centralized authentication through Keycloak with OAuth/OIDC flows, and permission delegation between components.

### Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant NGINX as NGINX Ingress
    participant GL as GitLab
    participant SH as SharingHub
    participant MLF as MLflow SharingHub
    participant KC as Keycloak
    
    Note over User,KC: 1. User Access to GitLab
    User->>Browser: Access gitlab.domain
    Browser->>NGINX: HTTPS Request
    NGINX->>GL: Forward request
    GL-->>Browser: Redirect to Keycloak
    Browser->>KC: OAuth/OIDC login
    KC-->>Browser: Return token
    Browser->>GL: Request with token
    GL->>KC: Validate token
    KC-->>GL: Token valid
    GL-->>Browser: Grant access
    
    Note over User,KC: 2. SharingHub API Access
    User->>SH: Access STAC API
    SH->>GL: OAuth token exchange
    GL->>KC: Validate via OIDC
    KC-->>GL: User identity
    GL-->>SH: User info + permissions
    SH-->>User: Filtered catalog (based on permissions)
    
    Note over User,KC: 3. MLflow Tracking
    User->>MLF: Log experiment via MLflow API
    MLF->>SH: Check project permissions
    SH->>GL: Verify project access
    GL-->>SH: Access granted/denied
    SH-->>MLF: Permission result
    MLF-->>User: Allow/deny operation
```

**Diagram: Authentication and Authorization Flow**

### Secrets Management

All sensitive credentials are stored in Kubernetes Secrets:

| Secret Name | Namespace | Purpose |
|-------------|-----------|---------|
| `gitlab-oidc` | `gitlab` | Keycloak client credentials for GitLab |
| `sharinghub-oidc` | `sharinghub` | OAuth credentials for SharingHub |
| `sharinghub-oidc-default-token` | `sharinghub` | Optional token for public read access |
| `mlflow-sharinghub` | `mlflow` | Secret key for MLflow server |
| `*-tls` | Various | TLS certificates from cert-manager |

**TLS Security:**
- All external traffic encrypted via HTTPS
- Certificates automatically provisioned by cert-manager
- Let's Encrypt used as Certificate Authority
- Certificates stored as Kubernetes TLS Secrets

Sources: [docs/design/architecture.md:52-76]()

### Permission Model

The permission model is hierarchical and delegated:

```mermaid
flowchart TD
    KC["Keycloak<br/>Identity Provider"] -->|"OIDC authentication"| GL["GitLab<br/>Source of Truth for Permissions"]
    
    GL -->|"Project membership<br/>public/private visibility"| PERMS["GitLab Project Permissions<br/>- Owner<br/>- Maintainer<br/>- Developer<br/>- Reporter<br/>- Guest"]
    
    PERMS -->|"OAuth token exchange"| SH["SharingHub<br/>Enforces permissions on STAC API"]
    
    SH -->|"Filtered catalog<br/>based on user access"| STACAPI["STAC API Response<br/>Only accessible projects"]
    
    MLF["MLflow SharingHub"] -->|"Permission check API"| SH
    
    SH -->|"Validates against GitLab"| GL
    
    USER["User"] -.->|"1. Authenticates"| KC
    USER -.->|"2. Accesses"| STACAPI
    USER -.->|"3. Logs experiments"| MLF
```

**Diagram: Permission Delegation Model**

**Permission Rules:**
- GitLab project permissions define who can access models and datasets
- SharingHub filters STAC catalog based on user's GitLab project access
- MLflow delegates all permission checks to SharingHub
- Public projects visible to anonymous users (if `sharinghub-oidc-default-token` configured)
- Private projects only visible to authenticated users with access

Sources: [docs/design/architecture.md:52-76]()

## Design Principles

The MLOps Building Block architecture follows these key design principles:

1. **Separation of Concerns**: Each component has a well-defined responsibility
   - GitLab: Version control and project management
   - SharingHub: Discovery and metadata management
   - MLflow: Experiment tracking and model registry

2. **Standards-Based Integration**: Uses established standards for interoperability
   - STAC for spatial data discovery
   - ONNX for framework-agnostic model representation
   - OAuth/OIDC for authentication

3. **GitOps Deployment**: Infrastructure as code with ArgoCD
   - Declarative configuration in Git repositories
   - Automated synchronization and drift detection
   - Version-controlled deployment history

4. **Cloud-Native Architecture**: Designed for Kubernetes environments
   - Containerized components
   - Horizontal scalability
   - Health checks and self-healing

5. **Security by Default**: Centralized authentication and fine-grained permissions
   - OIDC integration with Keycloak
   - Project-level access control via GitLab
   - TLS encryption for all external traffic

Sources: [docs/design/architecture.md:1-76](), [docs/index.md:16-45]()