# Infrastructure System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [infra/Readme.md](infra/Readme.md)
- [infra/docker-compose.yml](infra/docker-compose.yml)
- [infra/setup.sh](infra/setup.sh)

</details>



The Infrastructure System provides a production-ready MLOps stack for GeoAI model development and deployment. It consists of a containerized service architecture orchestrated by Docker Compose, including experiment tracking (MLflow), object storage (MinIO), database services (PostgreSQL with PostGIS), reverse proxy with automatic SSL (Traefik), system monitoring (Homepage), and remote access (RustDesk). This page covers the overall infrastructure architecture, service configuration, and operational aspects.

For detailed information about individual services, see sections [4.1](#4.1) through [4.7](#4.7). For deployment procedures, see [6.1](#6.1). For the ML training pipeline that utilizes this infrastructure, see [3](#3).

## Stack Architecture

The infrastructure stack is defined in [infra/docker-compose.yml:1-184]() as a collection of seven containerized services orchestrated through Docker Compose. All services communicate through a shared `traefik-network` bridge network, with Traefik acting as the edge ingress controller.

```mermaid
graph TB
    subgraph "Edge Layer"
        traefik["traefik<br/>Container: traefik<br/>Image: traefik:v3.0"]
    end
    
    subgraph "Application Services"
        homepage["homepage<br/>Container: homepage<br/>Image: ghcr.io/gethomepage/homepage:latest"]
        mlflow["mlflow<br/>Container: mlflow<br/>Image: ghcr.io/.../mlflow:latest"]
    end
    
    subgraph "Storage Services"
        minio["minio<br/>Container: minio<br/>Image: minio/minio:RELEASE.2025-04-22T22-12-26Z"]
        postgres["postgres<br/>Container: postgres<br/>Image: postgis/postgis:16-3.4-alpine"]
    end
    
    subgraph "Remote Access Services"
        hbbs["hbbs<br/>Container: hbbs<br/>Image: rustdesk/rustdesk-server:latest"]
        hbbr["hbbr<br/>Container: hbbr<br/>Image: rustdesk/rustdesk-server:latest"]
    end
    
    traefik -->|"routes to"| homepage
    traefik -->|"routes to"| mlflow
    traefik -->|"routes to"| minio
    traefik -->|"routes to"| postgres
    traefik -->|"routes to"| hbbs
    
    mlflow -->|"backend-store-uri"| postgres
    mlflow -->|"default-artifact-root"| minio
    
    homepage -.->|"monitors via<br/>Docker socket"| mlflow
    homepage -.->|"monitors via<br/>Docker socket"| minio
    homepage -.->|"monitors via<br/>Docker socket"| postgres
    
    hbbs -->|"depends_on"| hbbr
```

**Service Container Mapping**

| Service | Container Name | Image | Primary Port(s) |
|---------|----------------|-------|-----------------|
| Traefik | `traefik` | `traefik:v3.0` | 80, 443, 8080 |
| Homepage | `homepage` | `ghcr.io/gethomepage/homepage:latest` | 3000 |
| MLflow | `mlflow` | `${MLFLOW_IMAGE}` | 5000 |
| MinIO | `minio` | `minio/minio:RELEASE.2025-04-22T22-12-26Z` | 9000, 9001 |
| PostgreSQL | `postgres` | `postgis/postgis:16-3.4-alpine` | 5432 |
| RustDesk (hbbs) | `hbbs` | `rustdesk/rustdesk-server:latest` | 21115, 21116, 21118 |
| RustDesk (hbbr) | `hbbr` | `rustdesk/rustdesk-server:latest` | 21117, 21119 |

Sources: [infra/docker-compose.yml:1-184]()

## Service Configuration

Each service is configured through Docker Compose service definitions with three primary configuration mechanisms: environment variables, command-line arguments, and Traefik labels.

### MLflow Service Configuration

The `mlflow` service [infra/docker-compose.yml:62-84]() is configured to use PostgreSQL as its backend store and MinIO as its artifact store:

```mermaid
graph LR
    mlflow_entrypoint["mlflow server<br/>--backend-store-uri<br/>--default-artifact-root<br/>--artifacts-destination"]
    
    env_vars["Environment Variables<br/>MLFLOW_S3_ENDPOINT_URL<br/>AWS_ACCESS_KEY_ID<br/>AWS_SECRET_ACCESS_KEY"]
    
    postgres_conn["postgresql+psycopg2://<br/>${POSTGRES_USER}:${POSTGRES_PASSWORD}<br/>@postgres/${POSTGRES_DB}"]
    
    minio_conn["s3://${MINIO_BUCKET_NAME}/"]
    
    env_vars -->|"configures S3 access"| mlflow_entrypoint
    postgres_conn -->|"backend-store-uri"| mlflow_entrypoint
    minio_conn -->|"artifact storage"| mlflow_entrypoint
```

The entrypoint command [infra/docker-compose.yml:71]() constructs the full MLflow server invocation:

```
mlflow server 
  --backend-store-uri postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres/${POSTGRES_DB} 
  --default-artifact-root s3://${MINIO_BUCKET_NAME}/ 
  --artifacts-destination s3://${MINIO_BUCKET_NAME}/ 
  -h 0.0.0.0
```

Sources: [infra/docker-compose.yml:62-84]()

### Traefik Routing Configuration

Traefik routing is configured through Docker labels attached to each service. The routing follows a consistent pattern of subdomain-based host matching:

```mermaid
graph TB
    entrypoint_web["entrypoint: web<br/>:80"]
    entrypoint_websecure["entrypoint: websecure<br/>:443"]
    entrypoint_postgres["entrypoint: postgres<br/>:5432"]
    
    redirect["HTTP → HTTPS<br/>Redirect Middleware"]
    
    letsencrypt["certificatesresolvers.letsencrypt<br/>ACME HTTP Challenge"]
    
    router_homepage["traefik.http.routers.homepage<br/>Host(`${DOMAIN}`) || Host(`www.${DOMAIN}`)"]
    router_mlflow["traefik.http.routers.mlflow<br/>Host(`mlflow.${DOMAIN}`)"]
    router_minio_api["traefik.http.routers.minio-api<br/>Host(`minio-api.${DOMAIN}`)"]
    router_minio_console["traefik.http.routers.minio-console<br/>Host(`minio.${DOMAIN}`)"]
    router_traefik["traefik.http.routers.traefik<br/>Host(`traefik.${DOMAIN}`)"]
    
    tcp_router_postgres["traefik.tcp.routers.postgres<br/>HostSNI(`postgres.${DOMAIN}`)"]
    
    entrypoint_web -->|"redirections.entrypoint"| redirect
    redirect --> entrypoint_websecure
    
    entrypoint_websecure -->|"tls.certresolver"| letsencrypt
    
    entrypoint_websecure --> router_homepage
    entrypoint_websecure --> router_mlflow
    entrypoint_websecure --> router_minio_api
    entrypoint_websecure --> router_minio_console
    entrypoint_websecure --> router_traefik
    
    entrypoint_postgres --> tcp_router_postgres
    
    router_homepage -->|"service"| svc_homepage["homepage:3000"]
    router_mlflow -->|"service"| svc_mlflow["mlflow:5000"]
    router_minio_api -->|"service"| svc_minio_api["minio:9000"]
    router_minio_console -->|"service"| svc_minio_console["minio:9001"]
```

The Traefik service itself is configured with command-line flags [infra/docker-compose.yml:13-28]() that define:

- API dashboard accessibility [infra/docker-compose.yml:14-15]()
- Docker provider for service discovery [infra/docker-compose.yml:16-17]()
- Three entry points: `web` (80), `websecure` (443), `postgres` (5432) [infra/docker-compose.yml:18-20]()
- Automatic HTTP to HTTPS redirect [infra/docker-compose.yml:21-22]()
- Let's Encrypt certificate resolver [infra/docker-compose.yml:23-26]()

Sources: [infra/docker-compose.yml:2-38](), [infra/docker-compose.yml:52-60](), [infra/docker-compose.yml:75-81](), [infra/docker-compose.yml:97-110](), [infra/docker-compose.yml:126-133]()

## Network Architecture

The stack uses a single Docker bridge network named `traefik-network` [infra/docker-compose.yml:177-180]() for inter-service communication. All seven services connect to this network, enabling DNS-based service discovery where each service is reachable by its container name.

```mermaid
graph TB
    subgraph "traefik-network (bridge driver)"
        direction TB
        
        traefik_container["traefik<br/>Internal: traefik:80, traefik:443"]
        homepage_container["homepage<br/>Internal: homepage:3000"]
        mlflow_container["mlflow<br/>Internal: mlflow:5000"]
        minio_container["minio<br/>Internal: minio:9000, minio:9001"]
        postgres_container["postgres<br/>Internal: postgres:5432"]
        hbbs_container["hbbs<br/>Internal: hbbs:21115-21118"]
        hbbr_container["hbbr<br/>Internal: hbbr:21117, hbbr:21119"]
    end
    
    external["External Network<br/>Internet"]
    
    external -->|"80, 443, 8080"| traefik_container
    external -->|"21115-21119"| hbbs_container
    external -->|"21117, 21119"| hbbr_container
    
    traefik_container -.->|"reverse proxy"| homepage_container
    traefik_container -.->|"reverse proxy"| mlflow_container
    traefik_container -.->|"reverse proxy"| minio_container
    traefik_container -.->|"reverse proxy"| postgres_container
    traefik_container -.->|"reverse proxy"| hbbs_container
    
    mlflow_container -->|"psycopg2 connection"| postgres_container
    mlflow_container -->|"S3 API calls"| minio_container
    
    homepage_container -.->|"Docker socket monitoring"| traefik_container
```

**Port Mappings**

| Service | Published Ports | Internal Ports | Protocol |
|---------|----------------|----------------|----------|
| traefik | 80, 443, 8080 | 80, 443, 8080 | HTTP/HTTPS |
| homepage | - | 3000 | HTTP (internal only) |
| mlflow | - | 5000 | HTTP (internal only) |
| minio | - | 9000, 9001 | HTTP (internal only) |
| postgres | - | 5432 | PostgreSQL (internal only) |
| hbbs | 21115-21118 (21116 UDP) | 21115-21118 | RustDesk protocol |
| hbbr | 21117, 21119 | 21117, 21119 | RustDesk protocol |

Services that don't have published ports are only accessible through Traefik's reverse proxy, providing an additional security layer.

Sources: [infra/docker-compose.yml:6-9](), [infra/docker-compose.yml:147-150](), [infra/docker-compose.yml:171-173](), [infra/docker-compose.yml:177-180]()

## Data Persistence

The stack implements persistent storage through Docker volumes mounted to host directories. Five primary data volumes are configured:

```mermaid
graph LR
    subgraph "Host Filesystem"
        traefik_vol["${TRAEFIK_DATA_DIR}<br/>./volumes/traefik-data<br/>acme.json (600)"]
        minio_vol["${MINIO_DATA_DIR}<br/>./volumes/minio"]
        postgres_vol["${POSTGRES_DATA_DIR}<br/>./volumes/postgres"]
        rustdesk_vol["${RUSTDESK_DATA_DIR}<br/>./volumes/rustdesk"]
        homepage_vol["${HOMEPAGE_CONFIG}<br/>./homepage-config"]
    end
    
    subgraph "Container Mounts"
        traefik_mount["/data"]
        minio_mount["/data/minio"]
        postgres_mount["/var/lib/postgresql/data"]
        rustdesk_mount["/root"]
        homepage_mount["/app/config"]
    end
    
    traefik_vol --> traefik_mount
    minio_vol --> minio_mount
    postgres_vol --> postgres_mount
    rustdesk_vol --> rustdesk_mount
    homepage_vol --> homepage_mount
```

**Volume Configuration**

| Service | Environment Variable | Default Path | Container Mount | Purpose |
|---------|---------------------|--------------|-----------------|---------|
| Traefik | `TRAEFIK_DATA_DIR` | `./volumes/traefik-data` | `/data` | SSL certificates (acme.json) |
| MinIO | `MINIO_DATA_DIR` | `./volumes/minio` | `/data/minio` | S3 object storage |
| PostgreSQL | `POSTGRES_DATA_DIR` | `./volumes/postgres` | `/var/lib/postgresql/data` | Database files |
| RustDesk | `RUSTDESK_DATA_DIR` | `./volumes/rustdesk` | `/root` | RustDesk server data |
| Homepage | `HOMEPAGE_CONFIG` | `./homepage-config` | `/app/config` | Dashboard configuration |

The Traefik ACME certificate file requires strict permissions [infra/setup.sh:120-121]():

```bash
touch volumes/traefik-data/acme.json
chmod 600 volumes/traefik-data/acme.json
```

Sources: [infra/docker-compose.yml:12](), [infra/docker-compose.yml:50](), [infra/docker-compose.yml:95](), [infra/docker-compose.yml:125](), [infra/docker-compose.yml:146](), [infra/docker-compose.yml:170](), [infra/setup.sh:116-122]()

## Automated Setup Process

The setup process is automated through [infra/setup.sh:1-254](), which performs initialization, credential generation, and service deployment. The script implements a multi-stage setup workflow:

```mermaid
sequenceDiagram
    participant User
    participant setup_sh as "setup.sh"
    participant docker as "Docker"
    participant env_file as ".env File"
    participant systemd
    
    User->>setup_sh: "./setup.sh"
    
    setup_sh->>setup_sh: "Validate prerequisites<br/>docker, docker compose, openssl"
    
    alt .env does not exist
        setup_sh->>env_file: "cp .env.template .env"
        setup_sh->>setup_sh: "generate_password(20)<br/>POSTGRES_PASSWORD"
        setup_sh->>setup_sh: "generate_key(20)<br/>AWS_ACCESS_KEY"
        setup_sh->>setup_sh: "generate_key(40)<br/>AWS_SECRET_KEY"
        setup_sh->>setup_sh: "generate_key(16)<br/>RUSTDESK_KEY"
        setup_sh->>setup_sh: "generate_password(16)<br/>TRAEFIK_PASSWORD"
        setup_sh->>docker: "htpasswd -nbB<br/>Generate hash"
        docker-->>setup_sh: "TRAEFIK_HASH"
        setup_sh->>env_file: "sed -i replacements"
        setup_sh-->>User: "Display credentials<br/>EXIT: Update DOMAIN and ACME_EMAIL"
    end
    
    setup_sh->>env_file: "source .env"
    setup_sh->>setup_sh: "Validate DOMAIN != example.com"
    
    setup_sh->>setup_sh: "mkdir -p volumes/{...}"
    setup_sh->>setup_sh: "chmod 600 acme.json"
    
    setup_sh->>docker: "docker compose down"
    setup_sh->>docker: "docker compose pull"
    setup_sh->>docker: "docker compose up -d"
    
    setup_sh->>setup_sh: "sleep 30"
    setup_sh->>docker: "docker compose ps"
    
    setup_sh->>systemd: "Create tech-infra.service"
    setup_sh->>systemd: "systemctl enable tech-infra"
    
    setup_sh->>setup_sh: "Create manage.sh script"
    
    setup_sh-->>User: "Display service URLs<br/>Display credentials"
```

### Credential Generation

The script generates secure credentials using two helper functions [infra/setup.sh:11-20]():

- `generate_password()`: Creates base64-encoded random strings [infra/setup.sh:11-14]()
- `generate_key()`: Creates hex-encoded random strings [infra/setup.sh:17-20]()

Generated credentials:
- `POSTGRES_PASSWORD`: 20-character password [infra/setup.sh:55]()
- `AWS_ACCESS_KEY`: 20-character hex key [infra/setup.sh:56]()
- `AWS_SECRET_KEY`: 40-character hex key [infra/setup.sh:57]()
- `RUSTDESK_KEY`: 16-character hex key [infra/setup.sh:58]()
- `TRAEFIK_PASSWORD`: 16-character password [infra/setup.sh:59]()
- `TRAEFIK_HASH`: bcrypt hash generated via `htpasswd` [infra/setup.sh:62]()

The Traefik password hash generation doubles dollar signs for Docker Compose compatibility [infra/setup.sh:62]():

```bash
TRAEFIK_HASH=$(docker run --rm httpd:2.4-alpine htpasswd -nbB admin "$TRAEFIK_PASSWORD" 2>/dev/null | cut -d ":" -f 2 | sed 's/\$/\$\$/g')
```

Sources: [infra/setup.sh:1-254]()

## Service Management

The setup script generates a management utility `manage.sh` [infra/setup.sh:164-208]() that provides operational commands:

**Management Commands**

| Command | Description | Example |
|---------|-------------|---------|
| `start` | Start all services | `./manage.sh start` |
| `stop` | Stop all services | `./manage.sh stop` |
| `restart` | Restart service(s) | `./manage.sh restart mlflow` |
| `logs` | View service logs | `./manage.sh logs postgres` |
| `status` | Check service status | `./manage.sh status` |
| `update` | Pull and restart with latest images | `./manage.sh update` |
| `backup` | Create full backup | `./manage.sh backup` |

The `backup` command [infra/setup.sh:193-199]() creates timestamped backups including:
- All volume data copied to `./backups/YYYYMMDD_HHMMSS/volumes/`
- PostgreSQL database dump via `pg_dump` saved to `postgres_dump.sql`

### Systemd Integration

The setup script creates a systemd service unit `tech-infra.service` [infra/setup.sh:141-159]() for automatic startup:

```
[Unit]
Description=Tech Infrastructure Services
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=$USER
Group=$USER
WorkingDirectory=$(pwd)
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

This enables system-level service management:

```bash
sudo systemctl start tech-infra
sudo systemctl stop tech-infra
sudo systemctl status tech-infra
```

Sources: [infra/setup.sh:140-162](), [infra/setup.sh:164-208]()

## Security Configuration

The infrastructure implements multiple security layers:

### Authentication Mechanisms

```mermaid
graph TB
    public["Public Access<br/>Port 80, 443"]
    
    traefik_auth["Traefik Dashboard<br/>Basic Auth Middleware"]
    
    services["Protected Services"]
    
    postgres_tls["PostgreSQL<br/>TLS Required"]
    minio_creds["MinIO<br/>Access Key / Secret Key"]
    mlflow_s3["MLflow<br/>S3 Credentials"]
    
    public -->|"HTTPS with<br/>Let's Encrypt"| traefik_auth
    
    traefik_auth -->|"TRAEFIK_AUTH_USER<br/>TRAEFIK_AUTH_PASSWORD_HASH"| services
    
    services --> postgres_tls
    services --> minio_creds
    services --> mlflow_s3
    
    postgres_tls -.->|"POSTGRES_USER<br/>POSTGRES_PASSWORD"| auth_vars["Environment Variables"]
    minio_creds -.->|"MINIO_ROOT_USER<br/>MINIO_ROOT_PASSWORD"| auth_vars
    mlflow_s3 -.->|"AWS_ACCESS_KEY_ID<br/>AWS_SECRET_ACCESS_KEY"| auth_vars
```

**Authentication Methods by Service**

| Service | Method | Configuration |
|---------|--------|---------------|
| Traefik Dashboard | HTTP Basic Auth | `traefik.http.middlewares.auth.basicauth.users` [infra/docker-compose.yml:36]() |
| PostgreSQL | Password + TLS | `POSTGRES_USER`, `POSTGRES_PASSWORD` [infra/docker-compose.yml:120-122]() |
| MinIO | Access Key/Secret | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` [infra/docker-compose.yml:91-92]() |
| MLflow | Inherits PostgreSQL + MinIO | Environment variables [infra/docker-compose.yml:66-70]() |

### SSL Certificate Management

Traefik automatically obtains and renews SSL certificates through Let's Encrypt ACME protocol [infra/docker-compose.yml:23-26]():

- Challenge type: HTTP-01 [infra/docker-compose.yml:25-26]()
- Storage: `/data/acme.json` (must be 600 permissions) [infra/docker-compose.yml:24]()
- Email: `${ACME_EMAIL}` for renewal notifications [infra/docker-compose.yml:23]()

### Network Isolation

Services without published ports are only accessible through Traefik's reverse proxy, preventing direct external access. The Docker socket is mounted read-only [infra/docker-compose.yml:11]() to prevent container privilege escalation.

Sources: [infra/docker-compose.yml:23-36](), [infra/docker-compose.yml:91-92](), [infra/docker-compose.yml:120-122](), [infra/setup.sh:59-62]()

## Environment Configuration

The infrastructure is configured through environment variables defined in `.env` file, templated from [infra/.env.template](). Key configuration categories:

**Core Configuration Variables**

| Variable | Purpose | Example | Used By |
|----------|---------|---------|---------|
| `DOMAIN` | Base domain for all services | `example.com` | All services |
| `ACME_EMAIL` | Let's Encrypt notifications | `admin@example.com` | Traefik |
| `MLFLOW_IMAGE` | MLflow Docker image | `ghcr.io/.../mlflow:latest` | MLflow service |

**Credential Variables**

| Variable | Purpose | Generated By |
|----------|---------|--------------|
| `POSTGRES_USER` | PostgreSQL username | Template default |
| `POSTGRES_PASSWORD` | PostgreSQL password | `generate_password(20)` |
| `POSTGRES_DB` | PostgreSQL database name | Template default |
| `AWS_ACCESS_KEY_ID` | MinIO/S3 access key | `generate_key(20)` |
| `AWS_SECRET_ACCESS_KEY` | MinIO/S3 secret key | `generate_key(40)` |
| `TRAEFIK_AUTH_USER` | Traefik dashboard user | Template default |
| `TRAEFIK_AUTH_PASSWORD` | Traefik dashboard password | `generate_password(16)` |
| `TRAEFIK_AUTH_PASSWORD_HASH` | Bcrypt hash for auth | `htpasswd -nbB` |
| `RUSTDESK_KEY` | RustDesk encryption key | `generate_key(16)` |
| `MINIO_BUCKET_NAME` | S3 bucket for artifacts | Template default |

**Volume Path Variables**

| Variable | Purpose | Default |
|----------|---------|---------|
| `TRAEFIK_DATA_DIR` | Traefik data directory | `./volumes/traefik-data` |
| `MINIO_DATA_DIR` | MinIO data directory | `./volumes/minio` |
| `POSTGRES_DATA_DIR` | PostgreSQL data directory | `./volumes/postgres` |
| `RUSTDESK_DATA_DIR` | RustDesk data directory | `./volumes/rustdesk` |
| `HOMEPAGE_CONFIG` | Homepage config directory | `./homepage-config` |

**Homepage Configuration Variables**

| Variable | Purpose |
|----------|---------|
| `HOMEPAGE_ALLOWED_HOSTS` | Allowed domains for homepage |
| `PUID` | User ID for file permissions |
| `PGID` | Group ID for file permissions |
| `TZ` | Timezone for services |

The setup script validates that `DOMAIN` and `ACME_EMAIL` are changed from their template defaults [infra/setup.sh:107-111]() before proceeding with deployment.

Sources: [infra/setup.sh:47-94](), [infra/setup.sh:104-111](), [infra/docker-compose.yml:23](), [infra/docker-compose.yml:44-48](), [infra/docker-compose.yml:66-70](), [infra/docker-compose.yml:90-93](), [infra/docker-compose.yml:119-123]()

## Service Dependency Chain

The infrastructure services have defined startup dependencies managed through Docker Compose `depends_on` directives:

```mermaid
graph TD
    postgres["postgres<br/>PostgreSQL+PostGIS<br/>No dependencies"]
    minio["minio<br/>MinIO Object Storage<br/>No dependencies"]
    
    mlflow["mlflow<br/>MLflow Tracking Server<br/>depends_on: minio, postgres"]
    
    hbbr["hbbr<br/>RustDesk Relay<br/>No dependencies"]
    hbbs["hbbs<br/>RustDesk Server<br/>depends_on: hbbr"]
    
    traefik["traefik<br/>Reverse Proxy<br/>No dependencies"]
    homepage["homepage<br/>Dashboard<br/>No dependencies"]
    
    minio --> mlflow
    postgres --> mlflow
    hbbr --> hbbs
    
    traefik -.->|"routes all services"| mlflow
    traefik -.->|"routes all services"| minio
    traefik -.->|"routes all services"| postgres
    traefik -.->|"routes all services"| homepage
    traefik -.->|"routes all services"| hbbs
```

The MLflow service explicitly depends on both storage services [infra/docker-compose.yml:72-74](), ensuring they are started before MLflow attempts to connect. The RustDesk `hbbs` server depends on the `hbbr` relay [infra/docker-compose.yml:152-153](). All services are configured with `restart: unless-stopped` [infra/docker-compose.yml:5]() to ensure automatic recovery from failures.

Sources: [infra/docker-compose.yml:5](), [infra/docker-compose.yml:43](), [infra/docker-compose.yml:65](), [infra/docker-compose.yml:72-74](), [infra/docker-compose.yml:89](), [infra/docker-compose.yml:118](), [infra/docker-compose.yml:141](), [infra/docker-compose.yml:152-153](), [infra/docker-compose.yml:167]()

## Service URLs and Access Points

After deployment, services are accessible through subdomain-based routing:

**Public Service Endpoints**

| Service | URL Pattern | Port | Purpose |
|---------|------------|------|---------|
| Homepage Dashboard | `https://${DOMAIN}` or `https://www.${DOMAIN}` | 443 | System monitoring and service overview |
| MLflow Tracking | `https://mlflow.${DOMAIN}` | 443 | Experiment tracking UI and API |
| MinIO Console | `https://minio.${DOMAIN}` | 443 | Object storage web console |
| MinIO API | `https://minio-api.${DOMAIN}` | 443 | S3-compatible API endpoint |
| Traefik Dashboard | `https://traefik.${DOMAIN}` | 443 | Reverse proxy monitoring (auth required) |
| PostgreSQL | `postgres.${DOMAIN}:5432` | 5432 (TCP) | Database connections (TLS required) |
| RustDesk Web | `https://rustdesk.${DOMAIN}` | 443 | Remote desktop web interface |
| RustDesk Relay | `rustdesk.${DOMAIN}:21115-21119` | 21115-21119 | Direct relay connections |

The setup script displays all access URLs and credentials upon completion [infra/setup.sh:212-253]().

Sources: [infra/docker-compose.yml:31](), [infra/docker-compose.yml:54](), [infra/docker-compose.yml:77](), [infra/docker-compose.yml:99-105](), [infra/docker-compose.yml:128](), [infra/docker-compose.yml:156](), [infra/setup.sh:212-253]()