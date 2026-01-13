# MLflow Tracking Server

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.dockerignore](.dockerignore)
- [.github/workflows/docker-publish.yml](.github/workflows/docker-publish.yml)
- [.github/workflows/mlflow-image-publish.yml](.github/workflows/mlflow-image-publish.yml)
- [dockerfile.mlflow](dockerfile.mlflow)
- [infra/docker-compose.yml](infra/docker-compose.yml)

</details>



## Purpose and Scope

This document details the MLflow Tracking Server service within the OpenGeoAIModelHub infrastructure stack. It covers the service configuration, custom Docker image, integration with backend stores (PostgreSQL and MinIO), and the automated image publishing pipeline.

For information about other infrastructure services, see [Service Architecture](#4.1). For details about storage services configuration, see [Storage Services (MinIO and PostgreSQL)](#4.6). For guidance on using MLflow with the example model training pipeline, see [Training Pipeline](#3.2).

## Service Overview

MLflow is the central experiment tracking and model registry service in the infrastructure stack. It implements the standard two-store architecture pattern: PostgreSQL stores metadata (experiments, runs, metrics, parameters), while MinIO stores large artifacts (models, datasets, plots). The service is exposed via Traefik at `mlflow.${DOMAIN}` with automatic SSL certificate provisioning.

```mermaid
graph TB
    subgraph "MLflow Service Container"
        MLFLOW_SERVER["mlflow server<br/>Port 5000"]
        MLFLOW_BINARY["/usr/local/bin/mlflow"]
    end
    
    subgraph "Backend Store"
        POSTGRES["PostgreSQL Database<br/>postgres:5432<br/>POSTGRES_DB"]
        PSYCOPG2["psycopg2-binary<br/>Python Driver"]
    end
    
    subgraph "Artifact Store"
        MINIO["MinIO S3 Storage<br/>minio:9000<br/>MINIO_BUCKET_NAME"]
        BOTO3["boto3<br/>S3 Client"]
    end
    
    subgraph "Access Layer"
        TRAEFIK["Traefik Reverse Proxy"]
        USERS["Users/Training Pipeline"]
    end
    
    USERS -->|"HTTPS mlflow.${DOMAIN}"| TRAEFIK
    TRAEFIK -->|"HTTP :5000"| MLFLOW_SERVER
    
    MLFLOW_SERVER -->|"backend-store-uri"| PSYCOPG2
    PSYCOPG2 -->|"postgresql+psycopg2://"| POSTGRES
    
    MLFLOW_SERVER -->|"default-artifact-root"| BOTO3
    BOTO3 -->|"s3://"| MINIO
    
    MLFLOW_BINARY -.->|"uses"| MLFLOW_SERVER
    
    POSTGRES -.->|"stores"| METADATA["Experiments<br/>Runs<br/>Metrics<br/>Parameters"]
    MINIO -.->|"stores"| ARTIFACTS["Models<br/>Plots<br/>Datasets<br/>DLPK files"]
```

**Diagram: MLflow Two-Store Architecture**

The service depends on both `postgres` and `minio` containers, ensuring they start before MLflow initializes.

Sources: [infra/docker-compose.yml:62-84]()

## Service Configuration

The MLflow service is defined in the Docker Compose stack with specific environment variables, entrypoint configuration, and service dependencies.

### Container Definition

```yaml
mlflow:
  image: ${MLFLOW_IMAGE:-ghcr.io/kshitijrajsharma/opengeoaimodelshub/mlflow:latest}
  container_name: mlflow
  restart: unless-stopped
```

The service uses a custom-built Docker image, defaulting to `ghcr.io/kshitijrajsharma/opengeoaimodelshub/mlflow:latest` if the `MLFLOW_IMAGE` environment variable is not set. The container automatically restarts unless explicitly stopped.

| Configuration | Value | Purpose |
|---------------|-------|---------|
| `image` | `${MLFLOW_IMAGE}` | Custom MLflow image with boto3 and psycopg2 |
| `container_name` | `mlflow` | Fixed container name for DNS resolution |
| `restart` | `unless-stopped` | Automatic restart policy |
| `depends_on` | `minio`, `postgres` | Start order dependencies |

Sources: [infra/docker-compose.yml:62-65](), [infra/docker-compose.yml:72-74]()

### Environment Variables

The service requires five environment variables for S3 and database connectivity:

| Variable | Purpose | Example |
|----------|---------|---------|
| `MLFLOW_S3_ENDPOINT_URL` | MinIO endpoint for artifact storage | `http://minio:9000` |
| `MLFLOW_S3_IGNORE_TLS` | Disable TLS for internal S3 communication | `true` |
| `AWS_ACCESS_KEY_ID` | MinIO access key (root user) | From `.env` file |
| `AWS_SECRET_ACCESS_KEY` | MinIO secret key (root password) | From `.env` file |

```mermaid
graph LR
    ENV_FILE[".env file"]
    
    subgraph "Environment Variables"
        AWS_KEY["AWS_ACCESS_KEY_ID"]
        AWS_SECRET["AWS_SECRET_ACCESS_KEY"]
        PG_USER["POSTGRES_USER"]
        PG_PASS["POSTGRES_PASSWORD"]
        PG_DB["POSTGRES_DB"]
        BUCKET["MINIO_BUCKET_NAME"]
    end
    
    subgraph "MLflow Container"
        MLFLOW_ENV["Environment:<br/>MLFLOW_S3_ENDPOINT_URL<br/>MLFLOW_S3_IGNORE_TLS<br/>AWS_ACCESS_KEY_ID<br/>AWS_SECRET_ACCESS_KEY"]
        MLFLOW_CMD["Entrypoint:<br/>--backend-store-uri<br/>--default-artifact-root<br/>--artifacts-destination"]
    end
    
    ENV_FILE -->|"provides"| AWS_KEY
    ENV_FILE -->|"provides"| AWS_SECRET
    ENV_FILE -->|"provides"| PG_USER
    ENV_FILE -->|"provides"| PG_PASS
    ENV_FILE -->|"provides"| PG_DB
    ENV_FILE -->|"provides"| BUCKET
    
    AWS_KEY -->|"injected"| MLFLOW_ENV
    AWS_SECRET -->|"injected"| MLFLOW_ENV
    PG_USER -->|"interpolated"| MLFLOW_CMD
    PG_PASS -->|"interpolated"| MLFLOW_CMD
    PG_DB -->|"interpolated"| MLFLOW_CMD
    BUCKET -->|"interpolated"| MLFLOW_CMD
```

**Diagram: Environment Variable Flow**

The `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` serve dual purposes: they authenticate to MinIO (where they map to `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`) and provide S3 client credentials to MLflow.

Sources: [infra/docker-compose.yml:66-71]()

### Entrypoint Command

The service overrides the default MLflow image entrypoint with a complete server command:

```bash
mlflow server \
  --backend-store-uri postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres/${POSTGRES_DB} \
  --default-artifact-root s3://${MINIO_BUCKET_NAME}/ \
  --artifacts-destination s3://${MINIO_BUCKET_NAME}/ \
  -h 0.0.0.0
```

| Parameter | Purpose | Value |
|-----------|---------|-------|
| `--backend-store-uri` | Database connection string for metadata | `postgresql+psycopg2://user:pass@postgres/db` |
| `--default-artifact-root` | Default S3 path for new experiments | `s3://bucket_name/` |
| `--artifacts-destination` | Artifact storage location | `s3://bucket_name/` |
| `-h` | Bind address | `0.0.0.0` (all interfaces) |

The `backend-store-uri` uses the `postgresql+psycopg2` dialect, requiring the `psycopg2-binary` package installed in the custom Docker image. The hostname `postgres` resolves via Docker's internal DNS to the `postgres` container on the `traefik-network`.

Sources: [infra/docker-compose.yml:71]()

## Custom Docker Image

The MLflow service uses a custom Docker image built on top of the official MLflow image, adding PostgreSQL and S3 support dependencies.

### Dockerfile Structure

```mermaid
graph TB
    BASE["Base Image<br/>ghcr.io/mlflow/mlflow:v3.1.1"]
    
    subgraph "Build Stage: base"
        APT["apt-get update<br/>Install system dependencies"]
        REQ["Create requirements.txt:<br/>boto3<br/>psycopg2-binary"]
        PIP["pip install -r requirements.txt"]
    end
    
    subgraph "Final Stage"
        COPY["Copy site-packages<br/>Copy /usr/local/bin"]
        WORKDIR["Set WORKDIR /app"]
        EXPOSE["EXPOSE 5000"]
    end
    
    BASE -->|"FROM"| APT
    APT --> REQ
    REQ --> PIP
    PIP --> COPY
    COPY --> WORKDIR
    WORKDIR --> EXPOSE
```

**Diagram: Dockerfile Multi-Stage Build**

The Dockerfile uses build arguments for version flexibility:

| Build Argument | Default | Purpose |
|----------------|---------|---------|
| `MLFLOW_VERSION` | `v3.1.1` | Base MLflow image version |
| `BOTO3_VERSION` | `""` (latest) | AWS SDK for S3 support |
| `PSYCOPG2_VERSION` | `""` (latest) | PostgreSQL driver |

The multi-stage build pattern (`base` → `final`) optimizes the final image size by copying only necessary Python packages and binaries.

Sources: [dockerfile.mlflow:1-30]()

### Python Dependencies

The custom image adds two critical dependencies not included in the base MLflow image:

1. **boto3**: AWS SDK for Python, enabling S3-compatible storage (MinIO) integration
2. **psycopg2-binary**: PostgreSQL database adapter for Python, required for the `postgresql+psycopg2://` backend-store-uri

```dockerfile
RUN echo "boto3${BOTO3_VERSION:+==}${BOTO3_VERSION}" > /tmp/requirements.txt && \
    echo "psycopg2-binary${PSYCOPG2_VERSION:+==}${PSYCOPG2_VERSION}" >> /tmp/requirements.txt

RUN pip install --no-cache-dir -r /tmp/requirements.txt && \
    rm /tmp/requirements.txt
```

The build process conditionally pins versions if build arguments are provided, otherwise installs latest versions.

Sources: [dockerfile.mlflow:16-20]()

## Backend Store Integration

MLflow uses PostgreSQL as the backend store for all experiment metadata. The connection is established via SQLAlchemy's PostgreSQL dialect with psycopg2 as the driver.

### Database Connection

The connection string format is:
```
postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres/${POSTGRES_DB}
```

| Component | Purpose |
|-----------|---------|
| `postgresql+psycopg2` | SQLAlchemy dialect + driver specification |
| `${POSTGRES_USER}` | Database username (from environment) |
| `${POSTGRES_PASSWORD}` | Database password (from environment) |
| `@postgres` | Hostname (Docker service name) |
| `/${POSTGRES_DB}` | Database name (from environment) |

The hostname `postgres` is resolved by Docker's embedded DNS server to the IP address of the `postgres` container on the `traefik-network`.

### Stored Metadata

The backend store persists the following entities:

| Entity Type | Content | Example |
|-------------|---------|---------|
| Experiments | Project-level organization | "refugee-camp-detection" |
| Runs | Individual training executions | Run UUID, timestamps |
| Metrics | Time-series measurements | Loss, accuracy, F1-score |
| Parameters | Hyperparameters | Learning rate, batch size |
| Tags | Metadata annotations | "production", "baseline" |

Sources: [infra/docker-compose.yml:71]()

## Artifact Store Integration

MLflow stores large binary artifacts in MinIO using the S3 protocol. The artifact storage is configured with two related parameters.

### S3 Configuration

```bash
--default-artifact-root s3://${MINIO_BUCKET_NAME}/
--artifacts-destination s3://${MINIO_BUCKET_NAME}/
```

Both parameters point to the same S3 bucket but serve different purposes:
- `default-artifact-root`: Default location for new experiments
- `artifacts-destination`: Explicit override for artifact storage

The S3 endpoint is configured via environment variables rather than command-line arguments:

```yaml
environment:
  MLFLOW_S3_ENDPOINT_URL: "http://minio:9000"
  MLFLOW_S3_IGNORE_TLS: "true"
```

| Variable | Value | Purpose |
|----------|-------|---------|
| `MLFLOW_S3_ENDPOINT_URL` | `http://minio:9000` | MinIO API endpoint |
| `MLFLOW_S3_IGNORE_TLS` | `true` | Disable TLS verification for internal traffic |

The `boto3` library reads these environment variables and configures the S3 client to connect to MinIO instead of AWS S3.

### Stored Artifacts

The artifact store persists the following file types:

| Artifact Type | File Extension | Example Use Case |
|---------------|----------------|------------------|
| PyTorch models | `.pth`, `.pt` | Model checkpoints |
| ONNX models | `.onnx` | Cross-platform inference |
| ESRI packages | `.dlpk` | ArcGIS deployment |
| STAC metadata | `.json` | Model metadata |
| Plots | `.png`, `.jpg` | Confusion matrices, visualizations |
| Datasets | `.csv`, `.parquet` | Training/validation data |

Sources: [infra/docker-compose.yml:67-71]()

## Network and Routing Configuration

The MLflow service is exposed through Traefik using Docker labels for automatic service discovery and routing configuration.

### Traefik Labels

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.mlflow.rule=Host(`mlflow.${DOMAIN}`)"
  - "traefik.http.routers.mlflow.entrypoints=websecure"
  - "traefik.http.routers.mlflow.tls.certresolver=letsencrypt"
  - "traefik.http.services.mlflow.loadbalancer.server.port=5000"
  - "traefik.http.services.mlflow.loadbalancer.server.url=http://mlflow:5000"
```

```mermaid
graph LR
    INTERNET["Internet"]
    
    subgraph "Traefik Edge Router"
        HTTP["HTTP :80"]
        HTTPS["HTTPS :443"]
        ROUTER["HTTP Router:<br/>mlflow"]
        MIDDLEWARE["TLS Termination<br/>Let's Encrypt"]
        LB["Load Balancer"]
    end
    
    subgraph "MLflow Container"
        MLFLOW["MLflow Server<br/>:5000"]
    end
    
    INTERNET -->|"HTTP"| HTTP
    INTERNET -->|"HTTPS"| HTTPS
    
    HTTP -->|"redirect to HTTPS"| HTTPS
    HTTPS -->|"Host: mlflow.${DOMAIN}"| ROUTER
    ROUTER --> MIDDLEWARE
    MIDDLEWARE --> LB
    LB -->|"http://mlflow:5000"| MLFLOW
```

**Diagram: Traefik Routing to MLflow Service**

| Label | Purpose |
|-------|---------|
| `traefik.enable=true` | Enable Traefik for this service |
| `traefik.http.routers.mlflow.rule=Host(...)` | Route based on hostname |
| `traefik.http.routers.mlflow.entrypoints=websecure` | Use HTTPS entrypoint (:443) |
| `traefik.http.routers.mlflow.tls.certresolver=letsencrypt` | Automatic SSL certificates |
| `traefik.http.services.mlflow.loadbalancer.server.port=5000` | Backend container port |

### Network Configuration

The service is connected to the `traefik-network` bridge network, enabling communication with other services:

```yaml
networks:
  - traefik-network
```

This network provides:
1. **DNS resolution**: Service names (e.g., `postgres`, `minio`) resolve to container IPs
2. **Isolation**: Services on this network can communicate; external containers cannot
3. **Traefik integration**: Enables Traefik to proxy requests to the service

Sources: [infra/docker-compose.yml:75-83]()

## Image Publishing Pipeline

The custom MLflow Docker image is automatically built and published via GitHub Actions when the Dockerfile or workflow is modified.

### Workflow Triggers

The workflow [.github/workflows/mlflow-image-publish.yml]() executes on:

```yaml
on:
  push:
    branches: [main, master]
    paths:
      - "dockerfile.mlflow"
      - ".github/workflows/build-mlflow.yml"
  pull_request:
    branches: [main, master]
    paths:
      - "dockerfile.mlflow"
  workflow_dispatch:
```

| Trigger | Condition | Action |
|---------|-----------|--------|
| `push` | Dockerfile or workflow changes on main/master | Build and push |
| `pull_request` | Dockerfile changes in PR | Build only (no push) |
| `workflow_dispatch` | Manual trigger | Build and push |

### Build Process

```mermaid
graph TB
    TRIGGER["GitHub Push Event"]
    
    subgraph "GitHub Actions Workflow"
        CHECKOUT["Checkout repository<br/>actions/checkout@v4"]
        BUILDX["Setup Docker Buildx<br/>docker/setup-buildx-action@v3"]
        LOGIN["Login to GHCR<br/>docker/login-action@v3"]
        META["Extract metadata<br/>docker/metadata-action@v5"]
        BUILD["Build and push<br/>docker/build-push-action@v5"]
    end
    
    subgraph "Build Configuration"
        CONTEXT["Context: ."]
        DOCKERFILE["File: dockerfile.mlflow"]
        PLATFORMS["Platforms:<br/>linux/amd64<br/>linux/arm64"]
        CACHE["Cache: GitHub Actions"]
    end
    
    subgraph "Output"
        GHCR["GitHub Container Registry<br/>ghcr.io/.../mlflow:latest"]
        TAGS["Tags:<br/>:latest<br/>:master<br/>:master-{sha}"]
    end
    
    TRIGGER --> CHECKOUT
    CHECKOUT --> BUILDX
    BUILDX --> LOGIN
    LOGIN --> META
    META --> BUILD
    
    CONTEXT -.->|"input"| BUILD
    DOCKERFILE -.->|"input"| BUILD
    PLATFORMS -.->|"input"| BUILD
    CACHE -.->|"input"| BUILD
    
    BUILD -->|"push"| GHCR
    BUILD -->|"generates"| TAGS
```

**Diagram: CI/CD Pipeline for MLflow Image**

### Multi-Platform Build

The workflow builds for both AMD64 and ARM64 architectures:

```yaml
platforms: linux/amd64,linux/arm64
```

This enables deployment on:
- **x86_64 servers**: Traditional cloud VMs, data center hardware
- **ARM64 servers**: AWS Graviton, Apple Silicon, Raspberry Pi clusters

The build cache is stored in GitHub Actions cache to speed up subsequent builds:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

### Tag Strategy

The workflow generates multiple tags for version management:

| Tag Format | Example | Purpose |
|------------|---------|---------|
| `type=ref,event=branch` | `master` | Branch-based tag |
| `type=sha,prefix={{branch}}-` | `master-a1b2c3d` | Commit-specific tag |
| `type=raw,value=latest` | `latest` | Latest stable version |

Sources: [.github/workflows/mlflow-image-publish.yml:1-81]()

## Service Dependencies

The MLflow service declares explicit dependencies on storage services:

```yaml
depends_on:
  - minio
  - postgres
```

This ensures:
1. PostgreSQL starts before MLflow (backend store availability)
2. MinIO starts before MLflow (artifact store availability)
3. Docker Compose orders the startup sequence correctly

However, `depends_on` only waits for containers to start, not for services to be ready. MLflow implements retry logic to handle cases where PostgreSQL or MinIO take additional time to initialize.

```mermaid
sequenceDiagram
    participant DC as docker-compose
    participant PG as postgres
    participant MC as minio
    participant MF as mlflow
    
    DC->>PG: Start postgres container
    DC->>MC: Start minio container
    PG->>PG: Initialize database
    MC->>MC: Initialize S3 storage
    
    DC->>MF: Start mlflow container
    MF->>PG: Connect to PostgreSQL
    alt Database not ready
        PG-->>MF: Connection refused
        MF->>MF: Retry connection
    end
    PG-->>MF: Connection established
    
    MF->>MC: Connect to MinIO S3
    MC-->>MF: S3 client initialized
    
    MF->>MF: Start MLflow server on :5000
```

**Diagram: Service Startup Sequence**

Sources: [infra/docker-compose.yml:72-74]()

## Access and Authentication

The MLflow UI and API are accessible at `https://mlflow.${DOMAIN}` after Traefik provisions an SSL certificate. The service does not implement authentication by default; access control relies on network-level security (firewall rules, VPN) or can be added via Traefik middleware.

### API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `https://mlflow.${DOMAIN}/` | MLflow UI dashboard |
| `https://mlflow.${DOMAIN}/api/2.0/mlflow/experiments/list` | List experiments |
| `https://mlflow.${DOMAIN}/api/2.0/mlflow/runs/search` | Search runs |
| `https://mlflow.${DOMAIN}/#/models` | Model registry UI |

### Client Configuration

Training scripts connect to MLflow using the tracking URI:

```python
import mlflow

mlflow.set_tracking_uri("https://mlflow.yourdomain.com")
```

The client automatically handles SSL verification and communicates with both the backend store (via MLflow API) and artifact store (direct S3 connection to MinIO).

Sources: [infra/docker-compose.yml:75-81]()