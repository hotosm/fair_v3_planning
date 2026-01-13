# Service Architecture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [infra/Readme.md](infra/Readme.md)
- [infra/docker-compose.yml](infra/docker-compose.yml)

</details>



## Purpose and Scope

This document details the architecture of the infrastructure stack defined in [infra/docker-compose.yml](). It covers the composition of services, their dependencies, networking configuration, volume management, and routing mechanisms. For deployment procedures and the automated setup process, see [Setup and Deployment](#4.2). For individual service configuration details, see [Configuration Management](#4.3).

The infrastructure stack provides a production-ready MLOps platform consisting of seven containerized services orchestrated through Docker Compose. The architecture implements industry-standard patterns including edge proxy routing, centralized authentication, persistent storage abstraction, and service mesh networking.

## Service Inventory

The docker-compose stack defines seven services across three functional categories:

| Service | Container Name | Image | Primary Port(s) | Purpose |
|---------|---------------|-------|-----------------|---------|
| `traefik` | `traefik` | `traefik:v3.0` | 80, 443, 8080 | Edge router and SSL termination |
| `homepage` | `homepage` | `ghcr.io/gethomepage/homepage:latest` | 3000 | Dashboard and service monitoring |
| `mlflow` | `mlflow` | `ghcr.io/kshitijrajsharma/opengeoaimodelshub/mlflow:latest` | 5000 | ML experiment tracking server |
| `minio` | `minio` | `minio/minio:RELEASE.2025-04-22T22-12-26Z` | 9000, 9001 | S3-compatible object storage |
| `postgres` | `postgres` | `postgis/postgis:16-3.4-alpine` | 5432 | PostgreSQL database with PostGIS |
| `hbbs` | `hbbs` | `rustdesk/rustdesk-server:latest` | 21115, 21116, 21118 | RustDesk ID/Rendezvous server |
| `hbbr` | `hbbr` | `rustdesk/rustdesk-server:latest` | 21117, 21119 | RustDesk Relay server |

Sources: [infra/docker-compose.yml:1-176]()

## Service Dependency Architecture

The following diagram illustrates the startup order and runtime dependencies between services:

```mermaid
graph TB
    traefik["traefik<br/>(Edge Router)"]
    homepage["homepage<br/>(Dashboard)"]
    mlflow["mlflow<br/>(ML Tracking)"]
    minio["minio<br/>(Object Storage)"]
    postgres["postgres<br/>(Database)"]
    hbbs["hbbs<br/>(RustDesk ID Server)"]
    hbbr["hbbr<br/>(RustDesk Relay)"]
    
    mlflow -->|depends_on| minio
    mlflow -->|depends_on| postgres
    hbbs -->|depends_on| hbbr
    
    traefik -.->|routes to| homepage
    traefik -.->|routes to| mlflow
    traefik -.->|routes to| minio
    traefik -.->|routes to| postgres
    traefik -.->|routes to| hbbs
    
    homepage -.->|monitors via<br/>docker.sock| mlflow
    homepage -.->|monitors via<br/>docker.sock| minio
    homepage -.->|monitors via<br/>docker.sock| postgres
    
    mlflow -->|stores metadata| postgres
    mlflow -->|stores artifacts| minio
```

The `mlflow` service has hard dependencies on both `minio` and `postgres` [infra/docker-compose.yml:72-74](), ensuring the storage backend and metadata database are available before MLflow starts. The `hbbs` service depends on `hbbr` [infra/docker-compose.yml:152-153]() for relay functionality. All other services start independently with no explicit dependency constraints.

Sources: [infra/docker-compose.yml:72-74](), [infra/docker-compose.yml:152-153]()

## Network Architecture

All services connect to a single bridge network named `traefik-network` [infra/docker-compose.yml:177-180]():

```mermaid
graph LR
    subgraph "traefik-network (bridge)"
        traefik_c["traefik:80,443,8080"]
        homepage_c["homepage:3000"]
        mlflow_c["mlflow:5000"]
        minio_c["minio:9000,9001"]
        postgres_c["postgres:5432"]
        hbbs_c["hbbs:21115,21116,21118"]
        hbbr_c["hbbr:21117,21119"]
    end
    
    internet["Public Internet"] -->|":80,:443"| traefik_c
    
    traefik_c -->|"internal routing"| homepage_c
    traefik_c -->|"internal routing"| mlflow_c
    traefik_c -->|"internal routing"| minio_c
    traefik_c -->|"internal routing"| postgres_c
    traefik_c -->|"internal routing"| hbbs_c
    
    mlflow_c -->|"http://minio:9000"| minio_c
    mlflow_c -->|"postgresql://postgres:5432"| postgres_c
    
    hbbs_c -->|"relay coordination"| hbbr_c
```

The commented-out `mlflow-network` [infra/docker-compose.yml:181-183]() suggests a previous multi-network design that has been simplified to a single network topology. All inter-service communication uses container names as DNS hostnames (e.g., `http://minio:9000`).

### Port Exposure Strategy

Only `traefik` and the RustDesk services expose ports to the host:

- **traefik**: Exposes `:80`, `:443`, and `:8080` [infra/docker-compose.yml:6-9]() for HTTP/HTTPS traffic and dashboard access
- **hbbs**: Exposes `:21115`, `:21116` (TCP/UDP), `:21118` [infra/docker-compose.yml:147-151]() for RustDesk client connections
- **hbbr**: Exposes `:21117`, `:21119` [infra/docker-compose.yml:171-173]() for relay functionality

All other services remain internal to the Docker network and are accessed exclusively through Traefik routing.

Sources: [infra/docker-compose.yml:177-183](), [infra/docker-compose.yml:6-9](), [infra/docker-compose.yml:147-151](), [infra/docker-compose.yml:171-173]()

## Volume Management

The stack persists data using five Docker volumes with configurable mount paths:

| Service | Volume Type | Host Path (Default) | Container Path | Purpose |
|---------|-------------|---------------------|----------------|---------|
| `traefik` | bind mount | `./volumes/traefik-data` | `/data` | ACME certificates, configuration |
| `traefik` | bind mount | `/var/run/docker.sock` | `/var/run/docker.sock` (ro) | Docker API access for dynamic routing |
| `homepage` | bind mount | `./homepage-config` | `/app/config` | Dashboard configuration files |
| `homepage` | bind mount | `/var/run/docker.sock` | `/var/run/docker.sock` (ro) | Docker API access for monitoring |
| `minio` | bind mount | `./volumes/minio` | `/data/minio` | Object storage data |
| `postgres` | bind mount | `./volumes/postgres` | `/var/lib/postgresql/data` | Database data files |
| `hbbs`/`hbbr` | bind mount | `./volumes/rustdesk` | `/root` | RustDesk server state |

All volume paths support environment variable overrides via `${VAR_NAME:-default}` syntax [infra/docker-compose.yml:12](), [infra/docker-compose.yml:50](), [infra/docker-compose.yml:95](), [infra/docker-compose.yml:125](), [infra/docker-compose.yml:146]().

The Docker socket mounts (`/var/run/docker.sock:ro`) provide read-only access for `traefik` [infra/docker-compose.yml:11]() to discover services dynamically and for `homepage` [infra/docker-compose.yml:51]() to monitor container status.

Sources: [infra/docker-compose.yml:10-12](), [infra/docker-compose.yml:49-51](), [infra/docker-compose.yml:94-95](), [infra/docker-compose.yml:124-125](), [infra/docker-compose.yml:145-146](), [infra/docker-compose.yml:169-170]()

## Service Configuration Details

### Traefik (Edge Router)

The `traefik` service [infra/docker-compose.yml:2-38]() acts as the ingress controller with the following configuration:

**Command-line Configuration** [infra/docker-compose.yml:13-28]():
- Enables the API dashboard [infra/docker-compose.yml:14]() with secure access (non-insecure mode)
- Configures Docker provider with explicit exposure [infra/docker-compose.yml:16-17]()
- Defines three entrypoints: `web` (:80), `websecure` (:443), `postgres` (:5432) [infra/docker-compose.yml:18-20]()
- Implements HTTP to HTTPS redirect [infra/docker-compose.yml:21-22]()
- Configures Let's Encrypt resolver with HTTP-01 challenge [infra/docker-compose.yml:23-26]()

**Self-Routing Labels** [infra/docker-compose.yml:29-36]():
The Traefik dashboard is self-hosted using its own routing labels, exposing it at `traefik.${DOMAIN}` with basic authentication middleware [infra/docker-compose.yml:36]().

Sources: [infra/docker-compose.yml:2-38]()

### Homepage (Dashboard)

The `homepage` service [infra/docker-compose.yml:40-60]() provides monitoring capabilities:

**Environment Configuration** [infra/docker-compose.yml:44-48]():
- `HOMEPAGE_ALLOWED_HOSTS`: Controls access restrictions
- `PUID`/`PGID`: User/group IDs for file permissions
- `TZ`: Timezone configuration

**Routing Configuration** [infra/docker-compose.yml:54-56]():
Routes to both apex domain and www subdomain: `Host(\`${DOMAIN}\`) || Host(\`www.${DOMAIN}\`)` [infra/docker-compose.yml:54](), making it the default landing page.

Sources: [infra/docker-compose.yml:40-60]()

### MLflow (Tracking Server)

The `mlflow` service [infra/docker-compose.yml:62-84]() implements the two-store pattern:

**Backend Store URI** [infra/docker-compose.yml:71]():
```
postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres/${POSTGRES_DB}
```
Uses PostgreSQL for experiment metadata, metrics, and parameters.

**Artifact Store URI** [infra/docker-compose.yml:71]():
```
s3://${MINIO_BUCKET_NAME}/
```
Uses MinIO (S3-compatible) for model artifacts, plots, and datasets.

**S3 Configuration** [infra/docker-compose.yml:67-70]():
- `MLFLOW_S3_ENDPOINT_URL`: Points to internal MinIO service at `http://minio:9000`
- `MLFLOW_S3_IGNORE_TLS`: Disables TLS for internal communication
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`: MinIO credentials

The service uses a custom image from GitHub Container Registry [infra/docker-compose.yml:63](), allowing for pre-built deployment without local image building.

Sources: [infra/docker-compose.yml:62-84]()

### MinIO (Object Storage)

The `minio` service [infra/docker-compose.yml:86-113]() exposes two interfaces:

**API Endpoint** [infra/docker-compose.yml:99-104]():
- Routed to `minio-api.${DOMAIN}` for S3 API operations
- Port 9000 [infra/docker-compose.yml:103]()
- Used by MLflow for artifact storage

**Console Interface** [infra/docker-compose.yml:105-110]():
- Routed to `minio.${DOMAIN}` for web UI
- Port 9001 [infra/docker-compose.yml:109]()
- Accessed via `--console-address ":9001"` flag [infra/docker-compose.yml:96]()

**Authentication** [infra/docker-compose.yml:91-92]():
Uses `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD` which correspond to AWS credentials for S3 compatibility.

Sources: [infra/docker-compose.yml:86-113]()

### PostgreSQL (Database)

The `postgres` service [infra/docker-compose.yml:115-136]() uses the PostGIS image with multiple geospatial extensions:

**Extensions** [infra/docker-compose.yml:123]():
```
postgis,hstore,postgis_topology,postgis_raster,pgrouting
```

**TCP Routing** [infra/docker-compose.yml:127-133]():
Unlike HTTP services, PostgreSQL uses Traefik's TCP routing with `HostSNI` rule [infra/docker-compose.yml:128](). The service is exposed at `postgres.${DOMAIN}:5432` with TLS termination [infra/docker-compose.yml:132-133]().

Sources: [infra/docker-compose.yml:115-136]()

### RustDesk (Remote Desktop)

RustDesk consists of two cooperating services:

**hbbs (ID/Rendezvous Server)** [infra/docker-compose.yml:138-162]():
- Handles client registration and NAT traversal
- Command: `hbbs -r rustdesk.${DOMAIN} -k ${RUSTDESK_KEY}` [infra/docker-compose.yml:144]()
- The `-r` flag specifies the relay server address

**hbbr (Relay Server)** [infra/docker-compose.yml:164-175]():
- Handles actual data relay when direct P2P fails
- Command: `hbbr -k ${RUSTDESK_KEY}` [infra/docker-compose.yml:168]()

Both services share the same encryption key (`${RUSTDESK_KEY}`) and data directory [infra/docker-compose.yml:146](), [infra/docker-compose.yml:170]().

Sources: [infra/docker-compose.yml:138-175]()

## Label-Based Routing Configuration

Traefik routing is entirely configured through Docker labels rather than static configuration files. This enables dynamic service discovery and zero-downtime updates.

### HTTP Routing Pattern

All HTTP services follow this label pattern:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<service>.rule=Host(`<subdomain>.${DOMAIN}`)"
  - "traefik.http.routers.<service>.entrypoints=websecure"
  - "traefik.http.routers.<service>.tls.certresolver=letsencrypt"
  - "traefik.http.services.<service>.loadbalancer.server.port=<port>"
  - "traefik.http.services.<service>.loadbalancer.server.url=http://<container>:<port>"
```

**Service Resolution Mapping**:

| Router Name | Host Rule | Target Service | Target Port |
|-------------|-----------|----------------|-------------|
| `homepage` | `${DOMAIN}` \|\| `www.${DOMAIN}` | `http://homepage:3000` | 3000 |
| `mlflow` | `mlflow.${DOMAIN}` | `http://mlflow:5000` | 5000 |
| `minio-api` | `minio-api.${DOMAIN}` | `http://minio:9000` | 9000 |
| `minio-console` | `minio.${DOMAIN}` | `http://minio:9001` | 9001 |
| `rustdesk` | `rustdesk.${DOMAIN}` | `http://hbbs:21118` | 21118 |

### TCP Routing Pattern

PostgreSQL uses TCP routing [infra/docker-compose.yml:127-133]():

```yaml
labels:
  - "traefik.tcp.routers.postgres.rule=HostSNI(`postgres.${DOMAIN}`)"
  - "traefik.tcp.routers.postgres.entrypoints=postgres"
  - "traefik.tcp.routers.postgres.service=postgres"
  - "traefik.tcp.services.postgres.loadbalancer.server.port=5432"
  - "traefik.tcp.routers.postgres.tls=true"
  - "traefik.tcp.routers.postgres.tls.certresolver=letsencrypt"
```

The `HostSNI` rule performs routing based on the Server Name Indication (SNI) field in the TLS handshake.

### Middleware Configuration

Only the Traefik dashboard uses middleware for authentication [infra/docker-compose.yml:35-36]():

```yaml
- "traefik.http.routers.traefik.middlewares=auth"
- "traefik.http.middlewares.auth.basicauth.users=${TRAEFIK_AUTH_USER}:${TRAEFIK_AUTH_PASSWORD_HASH}"
```

Sources: [infra/docker-compose.yml:52-58](), [infra/docker-compose.yml:75-81](), [infra/docker-compose.yml:97-110](), [infra/docker-compose.yml:126-133](), [infra/docker-compose.yml:154-160]()

## Container Restart Policy

All services use `restart: unless-stopped` [infra/docker-compose.yml:5](), [infra/docker-compose.yml:43](), [infra/docker-compose.yml:65](), [infra/docker-compose.yml:89](), [infra/docker-compose.yml:118](), [infra/docker-compose.yml:141](), [infra/docker-compose.yml:167](), ensuring:
- Automatic restart on failure
- Automatic restart on system reboot
- No restart if manually stopped via `docker-compose stop`

This policy integrates with systemd service management (see [Setup and Deployment](#4.2)) for production-grade reliability.

Sources: [infra/docker-compose.yml:5](), [infra/docker-compose.yml:43](), [infra/docker-compose.yml:65](), [infra/docker-compose.yml:89](), [infra/docker-compose.yml:118](), [infra/docker-compose.yml:141](), [infra/docker-compose.yml:167]()

## Service Communication Flow

The following diagram illustrates how services communicate at runtime:

```mermaid
sequenceDiagram
    participant Client as "External Client"
    participant Traefik as "traefik:443"
    participant MLflow as "mlflow:5000"
    participant MinIO as "minio:9000"
    participant Postgres as "postgres:5432"
    
    Client->>Traefik: "HTTPS mlflow.${DOMAIN}/api/2.0/mlflow/runs/create"
    Traefik->>Traefik: "Match Host() rule, terminate TLS"
    Traefik->>MLflow: "HTTP mlflow:5000/api/2.0/mlflow/runs/create"
    
    MLflow->>Postgres: "INSERT INTO runs (name, experiment_id, ...)"
    Postgres-->>MLflow: "run_id = abc123"
    
    MLflow->>MLflow: "Log metrics, parameters to Postgres"
    MLflow->>Postgres: "INSERT INTO metrics VALUES (...)"
    
    Client->>Traefik: "POST /api/2.0/mlflow/artifacts (binary data)"
    Traefik->>MLflow: "Forward to mlflow:5000"
    
    MLflow->>MinIO: "PUT http://minio:9000/${MINIO_BUCKET_NAME}/abc123/model.pth"
    Note over MLflow,MinIO: "Uses MLFLOW_S3_ENDPOINT_URL env var"
    MinIO-->>MLflow: "200 OK, ETag: xyz"
    
    MLflow->>Postgres: "UPDATE runs SET artifact_uri = 's3://...'"
    MLflow-->>Traefik: "200 OK"
    Traefik-->>Client: "HTTPS 200 OK"
```

This flow demonstrates:
1. Traefik performing TLS termination and routing based on `Host()` labels
2. MLflow coordinating between PostgreSQL (metadata) and MinIO (artifacts)
3. Internal service communication using container names as DNS hostnames
4. Environment variable configuration (`MLFLOW_S3_ENDPOINT_URL`) directing artifact storage

Sources: [infra/docker-compose.yml:67-71](), [infra/docker-compose.yml:77-81]()