# Storage Services (MinIO and PostgreSQL)

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [infra/Readme.md](infra/Readme.md)
- [infra/docker-compose.yml](infra/docker-compose.yml)

</details>



## Purpose and Scope

This document details the two storage services that form the data layer of the OpenGeoAIModelHub infrastructure stack: MinIO for S3-compatible object storage and PostgreSQL with PostGIS extensions for relational data storage. These services provide persistent storage for MLflow experiments, model artifacts, and metadata. 

For information about how MLflow integrates with these services, see [MLflow Tracking Server](#4.5). For overall infrastructure architecture, see [Service Architecture](#4.1).

## Storage Architecture Overview

The infrastructure implements a standard two-store pattern where PostgreSQL handles structured metadata and MinIO handles large binary artifacts. Both services persist data to Docker volumes and are accessible through Traefik reverse proxy with automatic SSL termination.

```mermaid
graph TB
    subgraph "Storage Layer"
        minio["minio container"]
        postgres["postgres container"]
    end
    
    subgraph "Docker Volumes"
        minio_vol["./volumes/minio<br/>/data/minio"]
        postgres_vol["./volumes/postgres<br/>/var/lib/postgresql/data"]
    end
    
    subgraph "Network Access via Traefik"
        minio_api["minio-api.DOMAIN:443<br/>Port 9000"]
        minio_console["minio.DOMAIN:443<br/>Port 9001"]
        postgres_tcp["postgres.DOMAIN:5432<br/>TCP with TLS"]
    end
    
    subgraph "Client Services"
        mlflow["mlflow container"]
        external["External Clients<br/>psql, DBeaver, s3cmd"]
    end
    
    mlflow -->|"backend-store-uri<br/>postgresql+psycopg2://"| postgres
    mlflow -->|"default-artifact-root<br/>s3://bucket/"| minio
    
    external -->|"S3 API"| minio_api
    external -->|"Web Console"| minio_console
    external -->|"PostgreSQL Protocol"| postgres_tcp
    
    minio_api -->|"routes to"| minio
    minio_console -->|"routes to"| minio
    postgres_tcp -->|"routes to"| postgres
    
    minio -.->|"persists to"| minio_vol
    postgres -.->|"persists to"| postgres_vol
```

**Sources:** [infra/docker-compose.yml:62-136](), [infra/Readme.md:5-12]()

## MinIO Object Storage Service

MinIO provides S3-compatible object storage for MLflow artifacts including model files, plots, datasets, and ONNX/DLPK packages. The service exposes two interfaces: an S3-compatible API endpoint and a web-based management console.

### Container Configuration

The `minio` service is defined in [infra/docker-compose.yml:86-113]() with the following configuration:

| Configuration | Value | Description |
|--------------|-------|-------------|
| **Image** | `minio/minio:RELEASE.2025-04-22T22-12-26Z` | Specific MinIO release version |
| **Container Name** | `minio` | Docker container identifier |
| **Restart Policy** | `unless-stopped` | Automatic restart on failure |
| **Command** | `minio server /data/minio --console-address ":9001"` | Server startup with console on port 9001 |
| **Volume Mount** | `./volumes/minio:/data/minio` | Persistent storage location |

### Authentication and Access

MinIO uses root credentials configured via environment variables:

| Environment Variable | Purpose | Configuration Source |
|---------------------|---------|---------------------|
| `MINIO_ROOT_USER` | Root access username | `${AWS_ACCESS_KEY_ID}` from `.env` |
| `MINIO_ROOT_PASSWORD` | Root access password | `${AWS_SECRET_ACCESS_KEY}` from `.env` |
| `MINIO_IDENTITY_TYPE` | Authentication mode | `internal` (local user database) |

**Sources:** [infra/docker-compose.yml:90-93]()

### Endpoint Access via Traefik

MinIO exposes two distinct endpoints through Traefik routing:

```mermaid
graph LR
    subgraph "External Access"
        api_client["S3 API Clients<br/>boto3, s3cmd, aws-cli"]
        web_browser["Web Browsers"]
    end
    
    subgraph "Traefik Routing"
        api_router["minio-api router<br/>Host: minio-api.DOMAIN"]
        console_router["minio-console router<br/>Host: minio.DOMAIN"]
    end
    
    subgraph "MinIO Container"
        api_port["API Server<br/>Port 9000"]
        console_port["Console UI<br/>Port 9001"]
    end
    
    api_client -->|"HTTPS"| api_router
    web_browser -->|"HTTPS"| console_router
    
    api_router -->|"minio-api service"| api_port
    console_router -->|"minio-console service"| console_port
```

**Routing Configuration:**
- **API Endpoint:** `minio-api.${DOMAIN}` → `http://minio:9000` [infra/docker-compose.yml:99-104]()
- **Console Endpoint:** `minio.${DOMAIN}` → `http://minio:9001` [infra/docker-compose.yml:105-110]()
- **TLS Certificate:** Let's Encrypt via `letsencrypt` resolver [infra/docker-compose.yml:101,107]()

**Sources:** [infra/docker-compose.yml:97-111](), [infra/Readme.md:37-38]()

### Integration with MLflow

MLflow connects to MinIO using S3-compatible API credentials. The connection is established through environment variables in the `mlflow` service:

```
MLFLOW_S3_ENDPOINT_URL: "http://minio:9000"
MLFLOW_S3_IGNORE_TLS: "true"
AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
```

The MLflow server is configured to use MinIO as its artifact store:
- **Artifact Root:** `s3://${MINIO_BUCKET_NAME}/` [infra/docker-compose.yml:71]()
- **Artifacts Destination:** `s3://${MINIO_BUCKET_NAME}/` [infra/docker-compose.yml:71]()
- **Internal Access:** Direct container-to-container communication via `http://minio:9000` [infra/docker-compose.yml:67]()

**Sources:** [infra/docker-compose.yml:66-71]()

### Data Persistence

MinIO stores all object data in a Docker volume mounted at `/data/minio` inside the container. The host directory defaults to `./volumes/minio` but can be customized via the `MINIO_DATA_DIR` environment variable [infra/docker-compose.yml:95]().

**Volume Structure:**
```
volumes/
└── minio/
    └── [bucket-name]/
        ├── [run-id]/
        │   ├── artifacts/
        │   ├── models/
        │   └── outputs/
        └── .minio.sys/
```

## PostgreSQL Database Service

PostgreSQL serves as the backend store for MLflow, storing all experiment metadata, run parameters, metrics, and tags. The service uses the PostGIS-enabled image to support geospatial data operations.

### Container Configuration

The `postgres` service is defined in [infra/docker-compose.yml:115-136]() with the following configuration:

| Configuration | Value | Description |
|--------------|-------|-------------|
| **Image** | `postgis/postgis:16-3.4-alpine` | PostgreSQL 16 with PostGIS 3.4 |
| **Container Name** | `postgres` | Docker container identifier |
| **Restart Policy** | `unless-stopped` | Automatic restart on failure |
| **Volume Mount** | `./volumes/postgres:/var/lib/postgresql/data` | Persistent storage location |

**Sources:** [infra/docker-compose.yml:116-125]()

### Database Configuration

PostgreSQL is configured with multiple extensions to support geospatial and MLflow operations:

| Environment Variable | Value | Purpose |
|---------------------|-------|---------|
| `POSTGRES_USER` | From `${POSTGRES_USER}` | Database superuser username |
| `POSTGRES_PASSWORD` | From `${POSTGRES_PASSWORD}` | Database superuser password |
| `POSTGRES_DB` | From `${POSTGRES_DB}` | Initial database name (typically `mlflow`) |
| `POSTGRES_MULTIPLE_EXTENSIONS` | `postgis,hstore,postgis_topology,postgis_raster,pgrouting` | Enabled PostGIS extensions |

**Sources:** [infra/docker-compose.yml:119-123]()

### Network Access Configuration

PostgreSQL is exposed through Traefik using TCP routing with TLS termination:

```mermaid
graph TB
    subgraph "External Clients"
        psql["psql CLI"]
        dbeaver["DBeaver GUI"]
        mlflow_remote["MLflow Remote Clients"]
    end
    
    subgraph "Traefik TCP Router"
        postgres_router["postgres router<br/>HostSNI: postgres.DOMAIN<br/>Entrypoint: postgres (5432)"]
        tls_termination["TLS Termination<br/>certresolver: letsencrypt"]
    end
    
    subgraph "PostgreSQL Container"
        postgres_server["PostgreSQL Server<br/>Port 5432<br/>Database: mlflow"]
        extensions["Extensions:<br/>postgis, hstore,<br/>postgis_topology,<br/>postgis_raster,<br/>pgrouting"]
    end
    
    psql -->|"PostgreSQL Protocol<br/>TLS Required"| postgres_router
    dbeaver -->|"PostgreSQL Protocol<br/>TLS Required"| postgres_router
    mlflow_remote -->|"PostgreSQL Protocol<br/>TLS Required"| postgres_router
    
    postgres_router -->|"Decrypts"| tls_termination
    tls_termination -->|"postgres service"| postgres_server
    postgres_server -.->|"provides"| extensions
```

**Traefik TCP Routing Configuration:**
- **Rule:** `HostSNI(\`postgres.${DOMAIN}\`)` [infra/docker-compose.yml:128]()
- **Entrypoint:** `postgres` (port 5432) [infra/docker-compose.yml:129]()
- **Service Target:** Port 5432 on `postgres` container [infra/docker-compose.yml:131]()
- **TLS:** Enabled with Let's Encrypt certificates [infra/docker-compose.yml:132-133]()

**Sources:** [infra/docker-compose.yml:126-133]()

### Connection Details for External Clients

The database can be accessed externally using standard PostgreSQL clients with the following parameters:

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Host** | `postgres.yourdomain.com` | DNS A record must point to server IP |
| **Port** | `5432` | Standard PostgreSQL port |
| **Database** | Value of `${POSTGRES_DB}` | Typically `mlflow` |
| **Username** | Value of `${POSTGRES_USER}` | From `.env` file |
| **Password** | Value of `${POSTGRES_PASSWORD}` | From `.env` file |
| **SSL Mode** | `require` | TLS is mandatory |

**Example connection using psql:**
```bash
psql "postgresql://username:password@postgres.yourdomain.com:5432/mlflow?sslmode=require"
```

**Example DBeaver configuration:**
- Connection Type: PostgreSQL
- Host: `postgres.yourdomain.com`
- Port: `5432`
- Database: `mlflow`
- Authentication: Database Native
- SSL: Use SSL (Require)

**Sources:** [infra/Readme.md:43-50]()

### Integration with MLflow Backend Store

MLflow uses PostgreSQL as its backend store for all metadata via SQLAlchemy connection URI:

```
postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres/${POSTGRES_DB}
```

This URI is passed to the MLflow server via the `--backend-store-uri` parameter [infra/docker-compose.yml:71]().

**MLflow Metadata Stored in PostgreSQL:**
- Experiments and experiment metadata
- Runs and run parameters
- Metrics time series data
- Tags and notes
- Model registry metadata
- Model versions and stage transitions

**Sources:** [infra/docker-compose.yml:71]()

### Data Persistence

PostgreSQL stores all database files in a Docker volume mounted at `/var/lib/postgresql/data` inside the container. The host directory defaults to `./volumes/postgres` but can be customized via the `PUID_DATA_DIR` environment variable [infra/docker-compose.yml:125]().

**Volume Structure:**
```
volumes/
└── postgres/
    ├── base/           # Database files
    ├── global/         # Cluster-wide tables
    ├── pg_wal/         # Write-ahead logs
    ├── pg_stat/        # Statistics files
    └── postgresql.conf # Configuration (if overridden)
```

## Storage Service Integration Flow

The following diagram shows how a typical MLflow training run interacts with both storage services:

```mermaid
sequenceDiagram
    participant Train as "Training Script"
    participant MLflow as "mlflow container<br/>Port 5000"
    participant PG as "postgres container<br/>Port 5432"
    participant MinIO as "minio container<br/>Port 9000"
    participant PGVol as "postgres volume<br/>./volumes/postgres"
    participant MinIOVol as "minio volume<br/>./volumes/minio"
    
    Train->>MLflow: "mlflow.start_run()"
    MLflow->>PG: "INSERT INTO runs (...)"
    PG->>PGVol: "Write to disk"
    PG-->>MLflow: "Run ID"
    MLflow-->>Train: "Run context"
    
    loop "Training Iterations"
        Train->>MLflow: "mlflow.log_metric(key, value, step)"
        MLflow->>PG: "INSERT INTO metrics (...)"
        PG->>PGVol: "Write to disk"
    end
    
    Train->>MLflow: "mlflow.log_param(key, value)"
    MLflow->>PG: "INSERT INTO params (...)"
    PG->>PGVol: "Write to disk"
    
    Train->>MLflow: "mlflow.log_artifact(file)"
    MLflow->>MinIO: "S3 PUT s3://bucket/run-id/artifacts/file"
    MinIO->>MinIOVol: "Write to disk"
    MinIO-->>MLflow: "Success"
    
    Train->>MLflow: "mlflow.pytorch.log_model(model)"
    MLflow->>MinIO: "S3 PUT s3://bucket/run-id/artifacts/model/*"
    MinIO->>MinIOVol: "Write model files"
    MLflow->>PG: "UPDATE runs SET artifact_uri=..."
    PG->>PGVol: "Write to disk"
    
    Train->>MLflow: "mlflow.end_run()"
    MLflow->>PG: "UPDATE runs SET status='FINISHED'"
    PG->>PGVol: "Write to disk"
```

**Sources:** [infra/docker-compose.yml:62-136]()

## Network Configuration

Both storage services are attached to the `traefik-network` Docker network, enabling:
- Direct container-to-container communication (e.g., `mlflow` → `postgres`, `mlflow` → `minio`)
- External access through Traefik reverse proxy
- DNS-based service discovery within the Docker network

```mermaid
graph TB
    subgraph "traefik-network"
        traefik["traefik<br/>Reverse Proxy"]
        mlflow["mlflow<br/>Tracking Server"]
        minio["minio<br/>Object Storage"]
        postgres["postgres<br/>Database"]
    end
    
    mlflow -->|"Internal DNS<br/>postgres:5432"| postgres
    mlflow -->|"Internal DNS<br/>minio:9000"| minio
    
    traefik -->|"Proxies to"| mlflow
    traefik -->|"Proxies to"| minio
    traefik -->|"Proxies to"| postgres
    
    external["External Clients"] -->|"HTTPS/TCP+TLS"| traefik
```

**Network Driver:** `bridge` [infra/docker-compose.yml:178-180]()

**Sources:** [infra/docker-compose.yml:82-83,111-113,134-136,177-180]()

## Data Backup Considerations

Both storage services persist data to local Docker volumes, which should be included in backup strategies:

| Service | Volume Path | Critical Data | Backup Method |
|---------|-------------|---------------|---------------|
| **MinIO** | `./volumes/minio` | Model artifacts, plots, datasets, ONNX/DLPK files | `tar` archive or `rsync` of entire directory; MinIO CLI `mc mirror` |
| **PostgreSQL** | `./volumes/postgres` | Experiment metadata, metrics, parameters, model registry | `pg_dump` via PostgreSQL client; `tar` archive of volume directory |

**Example PostgreSQL backup command:**
```bash
pg_dump "postgresql://user:pass@postgres.yourdomain.com:5432/mlflow?sslmode=require" > mlflow_backup.sql
```

**Example MinIO backup using mc CLI:**
```bash
mc alias set myminio https://minio-api.yourdomain.com access_key secret_key
mc mirror myminio/bucket-name ./backup/minio-bucket
```

The infrastructure provides a management script for backup operations at [infra/manage.sh]() which can be invoked with `./manage.sh backup` [infra/Readme.md:59]().

**Sources:** [infra/Readme.md:54-60]()

## Volume Management

Both services use environment variables to customize volume mount locations:

| Environment Variable | Default Value | Purpose | Configuration Line |
|---------------------|---------------|---------|-------------------|
| `MINIO_DATA_DIR` | `./volumes/minio` | MinIO data directory | [infra/docker-compose.yml:95]() |
| `POSTGRES_DATA_DIR` | `./volumes/postgres` | PostgreSQL data directory | [infra/docker-compose.yml:125]() |

These variables allow for flexible storage configuration, such as mounting external block storage or NFS volumes for production deployments.

**Sources:** [infra/docker-compose.yml:95,125]()