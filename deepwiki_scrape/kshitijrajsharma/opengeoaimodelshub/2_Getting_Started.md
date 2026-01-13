# Getting Started

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examplemodel/README.md](examplemodel/README.md)
- [infra/Readme.md](infra/Readme.md)
- [infra/install-docker.sh](infra/install-docker.sh)

</details>



This page provides an overview of initial setup procedures for the OpenGeoAIModelHub repository. The repository contains two independent but complementary systems: an **Example Model System** for refugee camp detection and an **Infrastructure System** for production MLOps deployment. This guide introduces the setup requirements and workflows for both systems at a high level.

For detailed installation instructions, see [Prerequisites and Installation](#2.1). For step-by-step execution procedures, see [Quick Start Guide](#2.2). For comprehensive documentation of the example model, see [Example Model System](#3). For infrastructure deployment details, see [Infrastructure System](#4).

## Repository Structure

The repository is organized into two primary directories, each containing a complete system with its own dependencies, configuration, and deployment procedures:

```mermaid
graph TB
    ROOT["OpenGeoAIModelHub Repository"]
    
    ROOT --> EXAMPLE["examplemodel/<br/>Example Model System"]
    ROOT --> INFRA["infra/<br/>Infrastructure System"]
    ROOT --> ROOT_FILES["Root Configuration"]
    
    EXAMPLE --> EM_SRC["src/<br/>Source Code"]
    EXAMPLE --> EM_MLPROJ["MLproject<br/>Entry Points"]
    EXAMPLE --> EM_PYPROJ["pyproject.toml<br/>Dependencies"]
    EXAMPLE --> EM_README["README.md<br/>Model Docs"]
    
    EM_SRC --> TRAIN_PY["train.py<br/>Training Pipeline"]
    EM_SRC --> INFERENCE_PY["inference.py<br/>Inference System"]
    EM_SRC --> MODEL_PY["model.py<br/>U-Net Architecture"]
    EM_SRC --> ESRI_DIR["esri/<br/>DLPK Generation"]
    
    INFRA --> COMPOSE["docker-compose.yml<br/>Service Stack"]
    INFRA --> SETUP["setup.sh<br/>Deployment Script"]
    INFRA --> ENV_TMPL[".env.template<br/>Configuration"]
    INFRA --> MANAGE["manage.sh<br/>Operations"]
    INFRA --> INFRA_README["Readme.md<br/>Infra Docs"]
    INFRA --> INSTALL_DOCKER["install-docker.sh<br/>Docker Setup"]
    
    ROOT_FILES --> README["README.md<br/>Main Documentation"]
    ROOT_FILES --> GIT_DIR[".github/<br/>CI/CD Workflows"]
```

**Sources:** [infra/Readme.md:1-83](), [examplemodel/README.md:1-52]()

## System Architecture Overview

The two systems serve different but complementary purposes and can be used independently or together:

```mermaid
graph LR
    subgraph "Example Model System"
        EM_TRAIN["train.py<br/>Training Pipeline"]
        EM_INFER["inference.py<br/>Prediction System"]
        EM_MLPROJ["MLproject<br/>Orchestration"]
    end
    
    subgraph "Infrastructure System"
        INFRA_MLFLOW["MLflow Service<br/>Experiment Tracking"]
        INFRA_MINIO["MinIO Service<br/>Artifact Storage"]
        INFRA_PG["PostgreSQL Service<br/>Metadata Store"]
    end
    
    subgraph "Local Development"
        UV["uv Package Manager<br/>Dependency Install"]
        MLFLOW_LOCAL["mlflow ui<br/>Local Tracking"]
    end
    
    subgraph "External Data"
        OAM["OpenAerialMap<br/>Satellite Imagery"]
        OSM["OpenStreetMap<br/>Label Data"]
    end
    
    UV --> EM_TRAIN
    EM_MLPROJ --> EM_TRAIN
    EM_MLPROJ --> EM_INFER
    
    EM_TRAIN -.->|"can log to"| MLFLOW_LOCAL
    EM_TRAIN -.->|"or log to"| INFRA_MLFLOW
    
    INFRA_MLFLOW --> INFRA_PG
    INFRA_MLFLOW --> INFRA_MINIO
    
    EM_TRAIN --> OAM
    EM_TRAIN --> OSM
```

| System | Purpose | Primary Use Case | Deployment Target |
|--------|---------|-----------------|-------------------|
| Example Model | ML model development and training | Local experimentation, model development | Standalone training environment |
| Infrastructure | Production MLOps platform | Multi-user experiment tracking, model registry | Server/cloud deployment |

**Sources:** [infra/Readme.md:1-83](), [examplemodel/README.md:1-52]()

## Setup Workflows

There are two primary setup workflows depending on your objectives:

### Workflow Diagram: Setup Decision Tree

```mermaid
graph TD
    START["Clone Repository"]
    
    START --> CHOOSE{"What do you<br/>want to set up?"}
    
    CHOOSE -->|"Train model locally"| MODEL_PATH["Example Model Setup"]
    CHOOSE -->|"Deploy infrastructure"| INFRA_PATH["Infrastructure Setup"]
    CHOOSE -->|"Both"| BOTH["Sequential Setup"]
    
    MODEL_PATH --> UV_INSTALL["Install uv<br/>curl -LsSf astral.sh/uv/install.sh"]
    UV_INSTALL --> UV_SYNC["uv sync<br/>in examplemodel/"]
    UV_SYNC --> MLFLOW_UI["uv run mlflow ui<br/>Local tracking"]
    MLFLOW_UI --> PREPROCESS["uv run mlflow run . -e preprocess"]
    PREPROCESS --> TRAIN["uv run mlflow run . -e train"]
    
    INFRA_PATH --> DOCKER_CHECK{"Docker<br/>installed?"}
    DOCKER_CHECK -->|"No"| DOCKER_INSTALL["./infra/install-docker.sh"]
    DOCKER_CHECK -->|"Yes"| ENV_CONFIG
    DOCKER_INSTALL --> ENV_CONFIG["cp .env.template .env<br/>Configure domain/credentials"]
    ENV_CONFIG --> DNS_SETUP["Configure DNS A records"]
    DNS_SETUP --> RUN_SETUP["./infra/setup.sh"]
    
    BOTH --> MODEL_PATH
    BOTH --> INFRA_PATH
```

**Sources:** [infra/Readme.md:14-30](), [examplemodel/README.md:7-38](), [infra/install-docker.sh:1-50]()

### Example Model Setup Path

The example model system runs entirely on your local machine using `uv` for dependency management and can log experiments either locally or to a remote MLflow server.

**Key Steps:**
1. Install `uv` package manager
2. Run `uv sync` in `examplemodel/` directory to install dependencies from `pyproject.toml`
3. Execute MLflow entry points defined in `MLproject` file
4. Access results via local MLflow UI or remote infrastructure

**Key Files:**
- [examplemodel/pyproject.toml]() - Python dependencies
- [examplemodel/MLproject]() - Entry point definitions
- [examplemodel/src/train.py]() - Training pipeline
- [examplemodel/src/inference.py]() - Inference pipeline

**Sources:** [examplemodel/README.md:7-38]()

### Infrastructure Setup Path

The infrastructure system deploys a multi-service Docker Compose stack on a server with automatic SSL certificates, requiring domain name configuration and Docker installation.

**Key Steps:**
1. Install Docker and Docker Compose (via `install-docker.sh` if needed)
2. Configure `.env` file from `.env.template` with domain and credentials
3. Set up DNS A records for all service subdomains
4. Run `setup.sh` to deploy the stack
5. Access services via configured subdomains

**Key Files:**
- [infra/docker-compose.yml]() - Service definitions
- [infra/.env.template]() - Configuration template
- [infra/setup.sh]() - Automated deployment
- [infra/manage.sh]() - Operations management

**Sources:** [infra/Readme.md:14-30]()

## Prerequisites Summary

### For Example Model System

| Requirement | Purpose | Installation Method |
|------------|---------|-------------------|
| `uv` package manager | Python dependency management | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Python 3.10+ | Runtime environment | System package manager |
| Git | Repository cloning | System package manager |

### For Infrastructure System

| Requirement | Purpose | Installation Method |
|------------|---------|-------------------|
| Docker Engine | Container runtime | [infra/install-docker.sh]() or manual |
| Docker Compose | Multi-container orchestration | Included with Docker Engine |
| Domain name | SSL certificates and routing | DNS provider |
| Server with public IP | Service hosting | Cloud provider or on-premise |

**Detailed installation instructions are provided in [Prerequisites and Installation](#2.1).**

**Sources:** [examplemodel/README.md:7-12](), [infra/Readme.md:14-30](), [infra/install-docker.sh:1-50]()

## Configuration Overview

### Example Model Configuration

The example model uses environment variables for remote tracking server configuration:

```mermaid
graph LR
    ENV_VARS["Environment Variables"]
    PYPROJECT["pyproject.toml<br/>Dependencies"]
    MLPROJECT["MLproject<br/>Entry Points"]
    
    ENV_VARS --> TRAIN["train.py"]
    PYPROJECT --> UV["uv sync"]
    UV --> DEPS["Installed Dependencies"]
    DEPS --> TRAIN
    MLPROJECT --> TRAIN
    
    ENV_VARS -.->|"AWS_ACCESS_KEY_ID"| MINIO_AUTH["MinIO Authentication"]
    ENV_VARS -.->|"AWS_SECRET_ACCESS_KEY"| MINIO_AUTH
    ENV_VARS -.->|"MLFLOW_S3_ENDPOINT_URL"| MINIO_AUTH
    ENV_VARS -.->|"MLFLOW_TRACKING_URI"| MLFLOW_CONN["MLflow Connection"]
```

**Key Environment Variables:**
- `AWS_ACCESS_KEY_ID` - MinIO/S3 access key for artifact storage
- `AWS_SECRET_ACCESS_KEY` - MinIO/S3 secret key
- `MLFLOW_S3_ENDPOINT_URL` - MinIO endpoint URL (e.g., `http://minio.yourdomain.com:9000`)
- `MLFLOW_TRACKING_URI` - MLflow server URL (optional for remote logging)

**Sources:** [examplemodel/README.md:40-48]()

### Infrastructure Configuration

The infrastructure stack uses a single `.env` file to configure all services:

```mermaid
graph TB
    ENV_TEMPLATE[".env.template<br/>Configuration Template"]
    ENV_FILE[".env<br/>Actual Configuration"]
    
    ENV_TEMPLATE -->|"cp command"| ENV_FILE
    
    ENV_FILE --> DOMAIN["DOMAIN<br/>yourdomain.com"]
    ENV_FILE --> EMAIL["EMAIL<br/>letsencrypt@email.com"]
    ENV_FILE --> CREDS["Service Credentials<br/>Passwords, Keys"]
    
    DOMAIN --> TRAEFIK["traefik service<br/>Routing Rules"]
    EMAIL --> TRAEFIK
    CREDS --> MLFLOW["mlflow service"]
    CREDS --> MINIO["minio service"]
    CREDS --> POSTGRES["postgres service"]
    
    TRAEFIK --> SSL["Let's Encrypt<br/>SSL Certificates"]
```

**Required DNS Records:**
- `yourdomain.com` → Homepage dashboard
- `mlflow.yourdomain.com` → MLflow tracking server
- `minio.yourdomain.com` → MinIO console
- `minio-api.yourdomain.com` → MinIO S3 API
- `postgres.yourdomain.com` → PostgreSQL database
- `rustdesk.yourdomain.com` → RustDesk server
- `traefik.yourdomain.com` → Traefik dashboard

**Sources:** [infra/Readme.md:32-41]()

## Service Management

Once the infrastructure is deployed, services are managed through two mechanisms:

### Management Script

The `manage.sh` script provides operational commands:

```mermaid
graph LR
    MANAGE["./manage.sh"]
    
    MANAGE --> STATUS["status<br/>Check service health"]
    MANAGE --> LOGS["logs <service><br/>View service logs"]
    MANAGE --> RESTART["restart <service><br/>Restart specific service"]
    MANAGE --> UPDATE["update<br/>Pull latest images"]
    MANAGE --> BACKUP["backup<br/>Create data backup"]
```

**Sources:** [infra/Readme.md:54-60]()

### Systemd Integration

The `setup.sh` script optionally creates a systemd service for automatic startup:

| Command | Function |
|---------|----------|
| `sudo systemctl start tech-infra` | Start all services |
| `sudo systemctl stop tech-infra` | Stop all services |
| `sudo systemctl status tech-infra` | Check service status |
| `sudo systemctl enable tech-infra` | Enable auto-start on boot |

**Sources:** [infra/Readme.md:62-68]()

## Integration Patterns

### Local Development with Remote Infrastructure

The example model can be trained locally while logging to remote infrastructure:

```mermaid
sequenceDiagram
    participant DEV as "Local Machine"
    participant TRAIN as "train.py"
    participant MLFLOW as "mlflow.yourdomain.com"
    participant MINIO as "minio-api.yourdomain.com"
    participant PG as "postgres.yourdomain.com"
    
    DEV->>DEV: "export MLFLOW_TRACKING_URI=https://mlflow.yourdomain.com"
    DEV->>DEV: "export MLFLOW_S3_ENDPOINT_URL=http://minio-api.yourdomain.com:9000"
    DEV->>DEV: "uv run mlflow run . -e train"
    
    TRAIN->>MLFLOW: "mlflow.start_run()"
    MLFLOW->>PG: "Create run entry in metadata store"
    
    loop "Training Loop"
        TRAIN->>MLFLOW: "mlflow.log_metric()"
        MLFLOW->>PG: "Store metrics"
    end
    
    TRAIN->>MLFLOW: "mlflow.log_artifact()"
    MLFLOW->>MINIO: "Upload artifacts to S3 storage"
    
    DEV->>MLFLOW: "View experiments in browser"
```

**Sources:** [examplemodel/README.md:40-48](), [infra/Readme.md:5-12]()

## Next Steps

After understanding the repository structure and setup overview:

1. **Install Prerequisites** - Follow [Prerequisites and Installation](#2.1) for detailed installation instructions for both `uv` and Docker
2. **Execute Quick Start** - Follow [Quick Start Guide](#2.2) for step-by-step commands to run your first training or deploy infrastructure
3. **Explore the Model** - Read [Example Model System](#3) for comprehensive documentation of the ML pipeline
4. **Deploy Infrastructure** - Read [Infrastructure System](#4) for detailed service configuration and management
5. **Customize** - See [Development Guide](#7) for instructions on extending functionality and working with custom datasets

**Sources:** [infra/Readme.md:1-83](), [examplemodel/README.md:1-52]()