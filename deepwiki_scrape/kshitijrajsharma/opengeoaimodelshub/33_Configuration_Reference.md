# Configuration Reference

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examplemodel/.env_sample](examplemodel/.env_sample)
- [examplemodel/pyproject.toml](examplemodel/pyproject.toml)
- [infra/.env.template](infra/.env.template)

</details>



This document provides a complete reference for all configuration files, environment variables, and settings used in the OpenGeoAIModelHub system. It covers both the infrastructure stack configuration and the example model configuration.

For information about setting up the infrastructure stack, see [Infrastructure Deployment](#6.1). For local development configuration, see [Local Development Setup](#7.1). For MLproject entry point parameters, see [MLproject API Reference](#8.1).

## Configuration File Overview

The repository uses three primary configuration files:

1. **`infra/.env.template`** - Infrastructure stack environment variables for Docker Compose services
2. **`examplemodel/pyproject.toml`** - Python project dependencies and metadata
3. **`examplemodel/.env_sample`** - Runtime environment variables for MLflow tracking and artifact storage

```mermaid
graph TB
    subgraph "Configuration Files"
        ENV_TEMPLATE["infra/.env.template<br/>Infrastructure Settings"]
        PYPROJECT["examplemodel/pyproject.toml<br/>Python Dependencies"]
        ENV_SAMPLE["examplemodel/.env_sample<br/>Runtime Environment"]
    end
    
    subgraph "Docker Compose Stack"
        TRAEFIK["traefik service"]
        MLFLOW_SVC["mlflow service"]
        MINIO["minio service"]
        POSTGRES["postgresql service"]
        RUSTDESK["rustdesk service"]
        HOMEPAGE["homepage service"]
    end
    
    subgraph "Training Pipeline"
        TRAIN_PY["train.py"]
        INFERENCE_PY["inference.py"]
        MODEL_PY["model.py"]
    end
    
    ENV_TEMPLATE -->|provides env vars| TRAEFIK
    ENV_TEMPLATE -->|provides env vars| MLFLOW_SVC
    ENV_TEMPLATE -->|provides env vars| MINIO
    ENV_TEMPLATE -->|provides env vars| POSTGRES
    ENV_TEMPLATE -->|provides env vars| RUSTDESK
    ENV_TEMPLATE -->|provides env vars| HOMEPAGE
    
    PYPROJECT -->|defines dependencies for| TRAIN_PY
    PYPROJECT -->|defines dependencies for| INFERENCE_PY
    PYPROJECT -->|defines dependencies for| MODEL_PY
    
    ENV_SAMPLE -->|configures tracking URI| TRAIN_PY
    ENV_SAMPLE -->|configures S3 backend| TRAIN_PY
    ENV_SAMPLE -->|configures tracking URI| INFERENCE_PY
```

**Configuration File Relationships**

Sources: [infra/.env.template:1-37](), [examplemodel/pyproject.toml:1-31](), [examplemodel/.env_sample:1-5]()

## Infrastructure Configuration (.env.template)

The `.env.template` file defines all environment variables required by the Docker Compose infrastructure stack. This file must be copied to `.env` and populated with actual values before deployment.

### Domain and SSL Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `DOMAIN` | Base domain for all services | Yes | `example.com` | - |
| `ACME_EMAIL` | Email for Let's Encrypt SSL certificate notifications | Yes | `admin@example.com` | - |

These variables configure Traefik's automatic SSL certificate management through Let's Encrypt ACME protocol. All services are exposed as subdomains of `DOMAIN`.

Sources: [infra/.env.template:1-3]()

### Traefik Reverse Proxy Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `TRAEFIK_DATA_DIR` | Host directory for Traefik persistent data (certificates, configuration) | Yes | `./volumes/traefik-data` | `./volumes/traefik-data` |
| `TRAEFIK_AUTH_USER` | Username for Traefik dashboard basic authentication | Yes | `admin` | `admin` |
| `TRAEFIK_AUTH_PASSWORD` | Plain text password (used for hash generation only) | Yes | `securepassword123` | - |
| `TRAEFIK_AUTH_PASSWORD_HASH` | bcrypt hash of password for HTTP basic auth | Yes | `$2y$05$...` | - |

The `TRAEFIK_AUTH_PASSWORD_HASH` must be a valid bcrypt hash. Generate using:
```bash
echo $(htpasswd -nbB admin "your-password") | sed -e s/\\$/\\$\\$/g
```

The Traefik dashboard is accessible at `traefik.${DOMAIN}` using these credentials.

Sources: [infra/.env.template:5-9]()

### Homepage Dashboard Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `HOMEPAGE_CONFIG` | Host directory containing homepage configuration files | Yes | `./homepage-config` | `./homepage-config` |
| `HOMEPAGE_ALLOWED_HOSTS` | Comma-separated list of allowed hostnames | Yes | `example.com,www.example.com` | - |

Homepage provides a unified dashboard for monitoring all infrastructure services. It accesses the Docker socket to display service status.

Sources: [infra/.env.template:11-13]()

### MLflow Tracking Server Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `MLFLOW_IMAGE` | Docker image for MLflow tracking server | Yes | `ghcr.io/kshitijrajsharma/opengeoaimodelshub/mlflow:latest` | See example |

This points to the custom MLflow Docker image built by the CI/CD pipeline. The image includes necessary dependencies and configuration for PostgreSQL + MinIO backend integration.

MLflow server is accessible at `mlflow.${DOMAIN}`.

Sources: [infra/.env.template:15-16]()

### MinIO Object Storage Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `AWS_ACCESS_KEY_ID` | MinIO access key (S3-compatible) | Yes | `minioadmin` | - |
| `AWS_SECRET_ACCESS_KEY` | MinIO secret key (S3-compatible) | Yes | `minioadmin123` | - |
| `MINIO_BUCKET_NAME` | Default bucket name for MLflow artifacts | Yes | `mlflow` | `mlflow` |
| `MINIO_DATA_DIR` | Host directory for MinIO persistent storage | Yes | `./volumes/minio` | `./volumes/minio` |

MinIO provides S3-compatible object storage for MLflow artifacts (model files, plots, datasets). The service exposes:
- **API**: `minio-api.${DOMAIN}` - S3-compatible API endpoint
- **Console**: `minio.${DOMAIN}` - Web-based management console

The bucket specified in `MINIO_BUCKET_NAME` must be created before MLflow can store artifacts. This is typically handled automatically by the setup script.

Sources: [infra/.env.template:18-22]()

### PostgreSQL + PostGIS Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `POSTGRES_USER` | PostgreSQL superuser username | Yes | `mlflow` | - |
| `POSTGRES_PASSWORD` | PostgreSQL superuser password | Yes | `secure_pg_password` | - |
| `POSTGRES_DB` | Default database name | Yes | `mlflow` | `mlflow` |
| `POSTGRES_DATA_DIR` | Host directory for PostgreSQL persistent storage | Yes | `./volumes/postgres` | `./volumes/postgres` |

PostgreSQL stores MLflow metadata (experiments, runs, parameters, metrics, tags). The database includes PostGIS extension for geospatial data support.

The database is accessible internally at `postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgresql:5432/${POSTGRES_DB}` and externally at `postgres.${DOMAIN}:5432`.

Sources: [infra/.env.template:24-28]()

### RustDesk Remote Desktop Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `RUSTDESK_DATA_DIR` | Host directory for RustDesk persistent storage | Yes | `./volumes/rustdesk` | `./volumes/rustdesk` |
| `RUSTDESK_KEY` | RustDesk server encryption key | Yes | `generated-key-string` | - |

RustDesk provides secure remote desktop access to the infrastructure server. The key should be generated as specified in RustDesk documentation.

RustDesk service is accessible at `rustdesk.${DOMAIN}`.

Sources: [infra/.env.template:30-32]()

### System Configuration

| Variable | Purpose | Required | Example | Default |
|----------|---------|----------|---------|---------|
| `PUID` | User ID for file permissions | Yes | `1000` | `1000` |
| `PGID` | Group ID for file permissions | Yes | `1000` | `1000` |
| `TZ` | Timezone for container logs and timestamps | Yes | `UTC` | `UTC` |

These variables ensure that files created by Docker containers have correct ownership on the host system. Set `PUID` and `PGID` to match the host user running Docker Compose.

Sources: [infra/.env.template:34-37]()

## Infrastructure Service Subdomain Mapping

The following diagram shows how environment variables in `.env.template` map to service endpoints:

```mermaid
graph LR
    subgraph "Environment Variables"
        DOMAIN_VAR["DOMAIN=example.com"]
    end
    
    subgraph "Service Endpoints"
        HOMEPAGE_URL["homepage<br/>yourdomain.com"]
        MLFLOW_URL["mlflow<br/>mlflow.yourdomain.com"]
        MINIO_API_URL["minio-api<br/>minio-api.yourdomain.com"]
        MINIO_CONSOLE_URL["minio-console<br/>minio.yourdomain.com"]
        POSTGRES_URL["postgres<br/>postgres.yourdomain.com:5432"]
        RUSTDESK_URL["rustdesk<br/>rustdesk.yourdomain.com"]
        TRAEFIK_URL["traefik-dashboard<br/>traefik.yourdomain.com"]
    end
    
    DOMAIN_VAR --> HOMEPAGE_URL
    DOMAIN_VAR --> MLFLOW_URL
    DOMAIN_VAR --> MINIO_API_URL
    DOMAIN_VAR --> MINIO_CONSOLE_URL
    DOMAIN_VAR --> POSTGRES_URL
    DOMAIN_VAR --> RUSTDESK_URL
    DOMAIN_VAR --> TRAEFIK_URL
```

**Service URL Resolution from Domain Variable**

Sources: [infra/.env.template:2]()

## Example Model Configuration (pyproject.toml)

The `pyproject.toml` file defines Python project metadata and dependencies for the example model system. It follows PEP 621 standard for Python project metadata.

### Project Metadata

| Field | Value | Purpose |
|-------|-------|---------|
| `name` | `examplemodel` | Python package name |
| `version` | `0.0.1` | Package version following semantic versioning |
| `description` | Enhanced refugee camp detection model with production-ready STAC-MLM metadata | Project description |
| `readme` | `README.md` | Path to README file |
| `requires-python` | `>=3.10` | Minimum Python version requirement |

Sources: [examplemodel/pyproject.toml:1-6]()

### Core Dependencies

The following table categorizes all dependencies defined in `[project.dependencies]`:

| Category | Packages | Purpose |
|----------|----------|---------|
| **ML Framework** | `torch>=2.7.1`, `torchvision>=0.22.1`, `pytorch-lightning>=2.5.2` | PyTorch deep learning framework and Lightning training abstraction |
| **MLOps** | `mlflow>=3.1.1`, `pynvml>=12.0.0`, `psutil>=7.0.0` | Experiment tracking, GPU monitoring, system resource monitoring |
| **Model Export** | `onnx>=1.18.0`, `onnxscript>=0.3.2` | ONNX model format export and scripting |
| **GeoAI** | `geomltoolkits>=0.3.9` | Geospatial ML utilities for data preparation |
| **Metadata** | `stac-model>=0.3.0`, `pystac>=1.8.0`, `jsonschema>=4.0.0` | STAC-MLM metadata generation and validation |
| **Infrastructure** | `boto3>=1.39.12`, `requests>=2.28.0`, `dotenv>=0.9.9` | AWS S3/MinIO client, HTTP requests, environment variable management |
| **Image Processing** | `opencv-python>=4.8.0` | Computer vision operations for inference |

Sources: [examplemodel/pyproject.toml:7-24]()

### Dependency Groups

The `[dependency-groups]` section defines optional dependency sets for specific workflows:

| Group | Packages | Purpose |
|-------|----------|---------|
| `validation` | `jsonschema>=4.0.0`, `requests>=2.28.0` | STAC metadata validation workflow |

Install dependency groups using:
```bash
uv sync --group validation
```

Sources: [examplemodel/pyproject.toml:26-30]()

### Dependency Resolution and Lock File

Dependencies are resolved and locked using `uv` package manager. The resolution is deterministic and recorded in `uv.lock` (not shown in provided files but referenced in documentation).

```mermaid
graph TB
    subgraph "Dependency Management"
        PYPROJECT["pyproject.toml<br/>[project.dependencies]"]
        UV_LOCK["uv.lock<br/>Locked Versions"]
        UV_SYNC["uv sync command"]
        VENV[".venv/<br/>Virtual Environment"]
    end
    
    subgraph "Training Pipeline"
        TRAIN["train.py<br/>Training Script"]
        INFERENCE["inference.py<br/>Inference Script"]
        MODEL["model.py<br/>Model Architecture"]
    end
    
    PYPROJECT -->|reads requirements| UV_SYNC
    UV_SYNC -->|generates/updates| UV_LOCK
    UV_SYNC -->|installs packages to| VENV
    
    VENV -->|provides imports for| TRAIN
    VENV -->|provides imports for| INFERENCE
    VENV -->|provides imports for| MODEL
```

**Dependency Installation Flow**

Sources: [examplemodel/pyproject.toml:1-31]()

## Runtime Configuration (.env_sample)

The `.env_sample` file provides a template for runtime environment variables required by the training and inference scripts when connecting to the infrastructure stack.

### MLflow Tracking Configuration

| Variable | Purpose | Required | Example |
|----------|---------|----------|---------|
| `MLFLOW_TRACKING_URI` | MLflow tracking server URL | Yes | `http://mlflow.krschap.tech` |

This URL points to the MLflow tracking server. It can be:
- **Local**: `http://localhost:5000` (default MLflow port)
- **Remote**: `http://mlflow.${DOMAIN}` (infrastructure stack)
- **HTTPS**: `https://mlflow.${DOMAIN}` (with SSL enabled)

The training script uses this to log experiments, metrics, parameters, and artifacts.

Sources: [examplemodel/.env_sample:4]()

### MinIO S3 Backend Configuration

| Variable | Purpose | Required | Example |
|----------|---------|----------|---------|
| `AWS_ACCESS_KEY_ID` | MinIO access key (must match infrastructure `.env`) | Yes | `key_key` |
| `AWS_SECRET_ACCESS_KEY` | MinIO secret key (must match infrastructure `.env`) | Yes | `secret_secret` |
| `MLFLOW_S3_ENDPOINT_URL` | MinIO API endpoint URL | Yes | `https://minio-api.krschap.tech` |

These variables configure the S3-compatible client to store MLflow artifacts in MinIO. The credentials must match the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set in the infrastructure `.env` file.

The `MLFLOW_S3_ENDPOINT_URL` should point to the MinIO API endpoint:
- **Local**: `http://localhost:9000` (default MinIO port)
- **Remote**: `https://minio-api.${DOMAIN}` (infrastructure stack)

Sources: [examplemodel/.env_sample:1-3]()

## Configuration Flow Through System

The following diagram shows how configuration values flow from files through the system to running processes:

```mermaid
flowchart TD
    subgraph "Configuration Sources"
        ENV_TEMPLATE[".env.template<br/>Infrastructure Template"]
        ENV_FILE[".env<br/>Actual Values"]
        ENV_SAMPLE[".env_sample<br/>Runtime Template"]
        ENV_RUNTIME["Runtime .env<br/>Actual Values"]
        PYPROJECT["pyproject.toml<br/>Dependencies"]
    end
    
    subgraph "Deployment Phase"
        SETUP_SH["setup.sh script"]
        DOCKER_COMPOSE["docker-compose.yml"]
        SERVICES["Running Services:<br/>traefik, mlflow, minio,<br/>postgres, rustdesk, homepage"]
    end
    
    subgraph "Development Phase"
        UV_SYNC["uv sync"]
        EXPORT_ENV["export environment vars"]
        ML_SCRIPTS["Training/Inference Scripts"]
    end
    
    subgraph "Runtime Connections"
        MLFLOW_CLIENT["MLflow Client<br/>in Training Script"]
        S3_CLIENT["Boto3 S3 Client<br/>in Training Script"]
        MLFLOW_SERVER["MLflow Server<br/>Running Service"]
        MINIO_SERVER["MinIO Server<br/>Running Service"]
    end
    
    ENV_TEMPLATE -->|"copy and modify"| ENV_FILE
    ENV_FILE -->|"read by"| SETUP_SH
    ENV_FILE -->|"read by"| DOCKER_COMPOSE
    DOCKER_COMPOSE -->|"starts with env vars"| SERVICES
    
    ENV_SAMPLE -->|"copy and modify"| ENV_RUNTIME
    ENV_RUNTIME -->|"export vars"| EXPORT_ENV
    EXPORT_ENV -->|"provides config to"| ML_SCRIPTS
    
    PYPROJECT -->|"defines deps for"| UV_SYNC
    UV_SYNC -->|"installs packages for"| ML_SCRIPTS
    
    ML_SCRIPTS -->|"creates"| MLFLOW_CLIENT
    ML_SCRIPTS -->|"creates"| S3_CLIENT
    
    MLFLOW_CLIENT -->|"connects to<br/>MLFLOW_TRACKING_URI"| MLFLOW_SERVER
    S3_CLIENT -->|"connects to<br/>MLFLOW_S3_ENDPOINT_URL"| MINIO_SERVER
    
    SERVICES -->|"includes"| MLFLOW_SERVER
    SERVICES -->|"includes"| MINIO_SERVER
```

**Configuration to Runtime Flow**

Sources: [infra/.env.template:1-37](), [examplemodel/.env_sample:1-5](), [examplemodel/pyproject.toml:1-31]()

## Environment Variable Priority

When both infrastructure and runtime environments use similar variable names, the following priority applies:

| Context | Variables Used | Scope |
|---------|---------------|-------|
| **Infrastructure Deployment** | `infra/.env` | Controls Docker Compose services |
| **Training Script Execution** | `examplemodel/.env` or exported shell variables | Controls training script behavior |

Variables with the same name (e.g., `AWS_ACCESS_KEY_ID`) must have matching values between `infra/.env` and `examplemodel/.env` for the training script to successfully connect to the infrastructure services.

```mermaid
graph TB
    subgraph "Variable Name Collision"
        INFRA_AWS_KEY["AWS_ACCESS_KEY_ID<br/>in infra/.env"]
        RUNTIME_AWS_KEY["AWS_ACCESS_KEY_ID<br/>in examplemodel/.env"]
    end
    
    subgraph "Service Usage"
        MINIO_SVC["minio service<br/>Uses infra/.env value"]
        TRAIN_SCRIPT["train.py<br/>Uses runtime .env value"]
    end
    
    subgraph "Connection Requirement"
        MATCH["Values MUST match<br/>for connection to succeed"]
    end
    
    INFRA_AWS_KEY -->|configures| MINIO_SVC
    RUNTIME_AWS_KEY -->|configures| TRAIN_SCRIPT
    
    TRAIN_SCRIPT -->|connects to| MINIO_SVC
    MINIO_SVC -.->|validates credentials| MATCH
    TRAIN_SCRIPT -.->|provides credentials| MATCH
```

**Environment Variable Coordination Requirement**

Sources: [infra/.env.template:19-20](), [examplemodel/.env_sample:1-2]()

## Configuration Validation Requirements

### Infrastructure Configuration

Before deploying the infrastructure stack, validate:

1. **Domain DNS Resolution**: Ensure `DOMAIN` resolves to server IP address
2. **Subdomain DNS**: Ensure wildcard DNS or individual subdomain records exist for all services
3. **Email Validity**: `ACME_EMAIL` must be a valid email for Let's Encrypt notifications
4. **Password Hashing**: `TRAEFIK_AUTH_PASSWORD_HASH` must be valid bcrypt hash
5. **Directory Paths**: All `*_DATA_DIR` variables must be valid, writable paths
6. **Credentials Strength**: Use strong, unique passwords for all `*_PASSWORD` variables

### Example Model Configuration

Before running training, validate:

1. **Python Version**: System Python matches `requires-python` constraint (>=3.10)
2. **Network Connectivity**: `MLFLOW_TRACKING_URI` and `MLFLOW_S3_ENDPOINT_URL` are reachable
3. **Credential Matching**: S3 credentials match infrastructure MinIO credentials
4. **Dependency Resolution**: Run `uv sync` successfully without conflicts

## Common Configuration Patterns

### Pattern 1: Local Development

For local development without infrastructure stack:

**pyproject.toml**: Use as-is
**.env_sample**: Set to local MLflow instance
```bash
export MLFLOW_TRACKING_URI=http://localhost:5000
# Omit S3 variables to use local file storage
```

### Pattern 2: Remote Infrastructure

For production deployment with remote infrastructure:

**infra/.env**: Set all variables with production values
**examplemodel/.env**: Match credentials with infrastructure
```bash
export MLFLOW_TRACKING_URI=https://mlflow.yourdomain.com
export AWS_ACCESS_KEY_ID=<matches-infra-value>
export AWS_SECRET_ACCESS_KEY=<matches-infra-value>
export MLFLOW_S3_ENDPOINT_URL=https://minio-api.yourdomain.com
```

### Pattern 3: Hybrid Development

For local training with remote artifact storage:

**examplemodel/.env**: Mix local tracking with remote storage
```bash
export MLFLOW_TRACKING_URI=http://localhost:5000
export AWS_ACCESS_KEY_ID=<remote-credentials>
export AWS_SECRET_ACCESS_KEY=<remote-credentials>
export MLFLOW_S3_ENDPOINT_URL=https://minio-api.yourdomain.com
```

## Configuration File Locations in Repository

```mermaid
graph TB
    subgraph "Repository Root"
        INFRA_DIR["infra/"]
        EXAMPLE_DIR["examplemodel/"]
    end
    
    subgraph "Infrastructure Config"
        ENV_TEMPLATE["infra/.env.template<br/>Template File"]
        ENV_FILE["infra/.env<br/>Actual Configuration<br/>(git-ignored)"]
    end
    
    subgraph "Example Model Config"
        PYPROJECT["examplemodel/pyproject.toml<br/>Committed"]
        ENV_SAMPLE["examplemodel/.env_sample<br/>Template File"]
        ENV_RUNTIME_FILE["examplemodel/.env<br/>Actual Configuration<br/>(git-ignored)"]
    end
    
    INFRA_DIR --> ENV_TEMPLATE
    INFRA_DIR --> ENV_FILE
    EXAMPLE_DIR --> PYPROJECT
    EXAMPLE_DIR --> ENV_SAMPLE
    EXAMPLE_DIR --> ENV_RUNTIME_FILE
    
    ENV_TEMPLATE -.->|"copy to create"| ENV_FILE
    ENV_SAMPLE -.->|"copy to create"| ENV_RUNTIME_FILE
```

**Configuration File Organization**

Sources: [infra/.env.template:1](), [examplemodel/pyproject.toml:1](), [examplemodel/.env_sample:1]()

## Security Considerations

### Sensitive Variables

The following variables contain sensitive information and should never be committed to version control:

| Variable | Location | Sensitivity | Git Status |
|----------|----------|-------------|------------|
| `TRAEFIK_AUTH_PASSWORD` | `infra/.env` | High | Ignored |
| `AWS_ACCESS_KEY_ID` | `infra/.env`, `examplemodel/.env` | High | Ignored |
| `AWS_SECRET_ACCESS_KEY` | `infra/.env`, `examplemodel/.env` | High | Ignored |
| `POSTGRES_PASSWORD` | `infra/.env` | High | Ignored |
| `RUSTDESK_KEY` | `infra/.env` | Medium | Ignored |

### Template Files

Only `.env.template` and `.env_sample` are committed to the repository. These files contain placeholder values (e.g., `replace-with-*`) that must be replaced with actual secrets during deployment.

### Credential Rotation

When rotating credentials:

1. Update `infra/.env` with new values
2. Update `examplemodel/.env` with matching credentials
3. Restart infrastructure services: `docker-compose down && docker-compose up -d`
4. Restart any running training scripts to pick up new credentials

Sources: [infra/.env.template:8-9](), [infra/.env.template:19-20](), [infra/.env.template:25-26]()