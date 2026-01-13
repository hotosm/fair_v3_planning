# Storage and Persistence

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [system/clusters/creodias/resource-management/hr-data-access.yaml](system/clusters/creodias/resource-management/hr-data-access.yaml)
- [system/clusters/creodias/resource-management/hr-registration-api.yaml](system/clusters/creodias/resource-management/hr-registration-api.yaml)
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/hr-resource-catalogue.yaml)
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml](system/clusters/creodias/resource-management/hr-workspace-api.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml)
- [system/clusters/creodias/system/demo/hr-django-portal.yaml](system/clusters/creodias/system/demo/hr-django-portal.yaml)
- [system/clusters/creodias/system/storage/hr-storage.yaml](system/clusters/creodias/system/storage/hr-storage.yaml)
- [system/clusters/creodias/system/test/hr-cheese.yaml](system/clusters/creodias/system/test/hr-cheese.yaml)
- [system/clusters/creodias/user-management/um-login-service.yaml](system/clusters/creodias/user-management/um-login-service.yaml)
- [system/clusters/creodias/user-management/um-pdp-engine.yaml](system/clusters/creodias/user-management/um-pdp-engine.yaml)
- [system/clusters/creodias/user-management/um-user-profile.yaml](system/clusters/creodias/user-management/um-user-profile.yaml)

</details>



## Purpose and Scope

This document provides an overview of the storage and persistence architecture within the EOEPCA platform. It covers the three primary storage layers: S3 object storage for Earth Observation data and processing outputs, database systems for metadata and queuing, and NFS-backed persistent volumes for stateful application data.

For detailed information on specific storage subsystems, see:
- S3 storage tiers and bucket provisioning: [S3 Storage Architecture](#7.1)
- PostgreSQL and Redis configurations: [Database Systems](#7.2)
- NFS server setup and storage classes: [NFS and Persistent Volumes](#7.3)

For information on how workspaces provision isolated storage per user, see [Multi-Tenant Workspaces](#5.5).

---

## Storage Architecture Overview

The EOEPCA platform employs a three-tier storage architecture to handle different data lifecycle requirements and access patterns.

**Storage Tiers in EOEPCA Platform**

```mermaid
graph TB
    subgraph "S3 Object Storage Tier"
        CF["CloudFerro eodata Bucket<br/>data.cloudferro.com<br/>(Read-only EO Data)"]
        MinioGlobal["MinIO Global<br/>minio.develop.eoepca.org<br/>(Platform Storage)"]
        MinioCache["S3 Cache Bucket<br/>cf2.cloudferro.com:8080/cache-bucket<br/>(Rendered Tiles)"]
        MinioWorkspace["Workspace Buckets<br/>e.g. eric-workspace<br/>(User Outputs)"]
    end
    
    subgraph "Database Tier"
        PG1["resource-catalogue-db<br/>PostgreSQL<br/>(ISO 19115 Metadata)"]
        PG2["Workspace PostgreSQL<br/>pycsw Databases<br/>(User Metadata)"]
        Redis1["data-access-redis-master<br/>Redis<br/>(Registration Queues)"]
        Redis2["Workspace Redis<br/>vs-redis-master<br/>(User Queues)"]
    end
    
    subgraph "NFS Persistent Volume Tier"
        NFS["NFS Server<br/>192.168.123.14"]
        PVC1["eoepca-userman-pvc<br/>(User Management State)"]
        PVC2["managed-nfs-storage<br/>(Dynamic Workspace PVCs)"]
        PVC3["managed-nfs-storage-retain<br/>(Retained Workspace Data)"]
    end
    
    subgraph "Consumers"
        DA["Data Access<br/>Renderer"]
        RC["Resource Catalogue<br/>pycsw"]
        WS["Workspace API<br/>Provisioner"]
        UM["User Management<br/>Gluu/Keycloak"]
    end
    
    CF -->|"Read EO Products"| DA
    MinioGlobal -->|"Store/Retrieve"| WS
    MinioCache -->|"Cache Tiles"| DA
    MinioWorkspace -->|"User Data"| DA
    
    PG1 -->|"Query Metadata"| RC
    PG2 -->|"User Metadata"| RC
    Redis1 -->|"Process Jobs"| DA
    Redis2 -->|"User Jobs"| DA
    
    NFS -->|"Mount"| PVC1
    NFS -->|"Provision"| PVC2
    NFS -->|"Provision"| PVC3
    PVC1 -->|"State Files"| UM
    PVC2 -->|"Database Files"| PG2
    PVC2 -->|"Redis Snapshots"| Redis2
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64]()
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:36-49]()
- [system/clusters/creodias/system/storage/hr-storage.yaml:16-28]()

---

## S3 Object Storage

The platform uses S3-compatible object storage for high-volume Earth Observation data and processing artifacts. Three distinct S3 storage tiers serve different purposes in the data lifecycle.

### S3 Storage Configuration

| Storage Type | Endpoint | Purpose | Access Pattern | Configured In |
|--------------|----------|---------|----------------|---------------|
| CloudFerro eodata | `http://data.cloudferro.com` | Source EO data (Sentinel, Landsat) | Read-only | Data Access |
| MinIO Platform | `https://minio.develop.eoepca.org` | Workspace buckets, outputs | Read/Write | Workspace API |
| Cache Bucket | `https://cf2.cloudferro.com:8080/cache-bucket` | Rendered WMS/WMTS tiles | Read/Write | Data Access |

### Global Data Access S3 Configuration

The global Data Access service reads from the CloudFerro eodata bucket for source EO products and writes rendered tiles to a cache bucket.

```yaml
# From hr-data-access.yaml
storage:
  data:
    data:
      type: S3
      endpoint_url: http://data.cloudferro.com
      access_key_id: access
      secret_access_key: access
      region_name: RegionOne
      validate_bucket_name: false
  cache:
    type: S3
    endpoint_url: "https://cf2.cloudferro.com:8080/cache-bucket"
    host: "cf2.cloudferro.com:8080"
    region_name: RegionOne
    bucket: cache-bucket
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64]()

### Workspace-Specific S3 Configuration

Each workspace receives an isolated S3 bucket provisioned through MinIO. The Workspace API creates these buckets and injects credentials into workspace service configurations.

```yaml
# From hr-workspace-api.yaml
s3Endpoint: "https://minio.develop.eoepca.org"
s3Region: "RegionOne"
bucketEndpointUrl: "http://minio-bucket-api:8080/bucket"
```

The template configuration for workspace data access services receives S3 credentials as template parameters:

```yaml
# From template-hr-data-access.yaml
storage:
  data:
    data:
      type: "S3"
      endpoint_url: https://minio.develop.eoepca.org
      access_key_id: {{ access_key_id }}
      secret_access_key: {{ secret_access_key }}
      bucket: {{ bucket }}
      region_name: RegionOne
      validate_bucket_name: false
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:38-47]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44]()

### S3 Environment Variables

The Data Access renderer uses GDAL's Virtual File System (VSI) interface to read from S3. Configuration is passed via environment variables:

```yaml
# From hr-data-access.yaml
global:
  env:
    CPL_VSIL_CURL_ALLOWED_EXTENSIONS: .TIF,.TIFF,.tif,.tiff,.xml,.jp2,.jpg,.jpeg,.png,.nc
    AWS_ENDPOINT_URL_S3: https://minio.develop.eoepca.org
    AWS_HTTPS: "FALSE"
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:27-31]()

---

## Database Systems

PostgreSQL and Redis provide structured data persistence for metadata cataloging and asynchronous job processing.

### PostgreSQL for Metadata

The Resource Catalogue uses PostgreSQL to store ISO 19115-compliant metadata records through the pycsw OGC CSW implementation.

**PostgreSQL Database Architecture**

```mermaid
graph TB
    subgraph "Global Resource Management"
        RCDB["resource-catalogue-db<br/>PostgreSQL Pod"]
        RCPVC["PVC: 5Gi"]
    end
    
    subgraph "Workspace eric-workspace"
        WSDB["vs-database<br/>PostgreSQL Pod"]
        WSPVC["PVC: 100Gi<br/>managed-nfs-storage"]
    end
    
    subgraph "Registrar Services"
        RegGlobal["registrar<br/>(Global)"]
        RegWS["registrar<br/>(Workspace)"]
    end
    
    subgraph "Catalogue Services"
        PyCSW["resource-catalogue<br/>pycsw"]
        WSPyCSW["rm-resource-catalogue<br/>pycsw"]
    end
    
    RCPVC -->|"Mounts"| RCDB
    WSPVC -->|"Mounts"| WSDB
    
    RegGlobal -->|"postgresql://postgres:mypass@<br/>resource-catalogue-db/pycsw"| RCDB
    RegWS -->|"postgresql://postgres:mypass@<br/>resource-catalogue-db/pycsw"| WSDB
    
    PyCSW -->|"Query Metadata"| RCDB
    WSPyCSW -->|"Query Metadata"| WSDB
```

#### Global Resource Catalogue Database Configuration

The global Resource Catalogue database is configured with performance tuning for metadata queries:

```yaml
# From hr-resource-catalogue.yaml
db:
  volume_size: 5Gi
  config:
    enabled: true
    shared_buffers: 2GB
    effective_cache_size: 6GB
    maintenance_work_mem: 512MB
    checkpoint_completion_target: 0.9
    wal_buffers: 16MB
    default_statistics_target: 100
    random_page_cost: 4
    work_mem: 4MB
    cpu_tuple_cost: 0.4
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:20-30]()

#### Workspace Database Configuration

Each workspace provisions a dedicated PostgreSQL instance for user-specific metadata:

```yaml
# From template-hr-data-access.yaml
vs:
  database:
    persistence:
      enabled: true
      existingClaim: ""  # generated dynamically
      storageClass: managed-nfs-storage
      size: "100Gi"
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-228]()

#### Database Connection Strings

Registrar backends connect to PostgreSQL using standardized connection URIs:

```yaml
# From hr-data-access.yaml (example)
backends:
  - path: registrar_pycsw.backend.ItemBackend
    kwargs:
      repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
      ows_url: https://data-access.develop.eoepca.org/ows
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:889-892]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:110-112]()

### Redis for Queuing

Redis provides message queuing for asynchronous data registration and processing workflows.

**Redis Queue Architecture**

```mermaid
graph LR
    subgraph "Data Sources"
        Harvester["harvester<br/>(OpenSearch Polling)"]
        Manual["Registration API<br/>(Manual Submit)"]
    end
    
    subgraph "Redis Queues"
        RedisMaster["data-access-redis-master<br/>Port 6379"]
        Queues["Queues:<br/>register_queue<br/>register_collection_queue<br/>register_ades_queue<br/>register_application_queue<br/>seed_queue"]
    end
    
    subgraph "Consumers"
        Registrar["registrar<br/>(Multiple Routes)"]
        Seeder["seeder<br/>(Tile Generation)"]
    end
    
    Harvester -->|"Enqueue STAC Items"| RedisMaster
    Manual -->|"Enqueue Metadata"| RedisMaster
    RedisMaster -->|"Contains"| Queues
    Queues -->|"Consume"| Registrar
    Queues -->|"Consume"| Seeder
```

#### Global Redis Configuration

```yaml
# From hr-data-access.yaml (harvester config)
harvester:
  config:
    redis:
      host: data-access-redis-master
      port: 6379
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:958-960]()

#### Workspace Redis Configuration

Each workspace deploys a dedicated Redis instance for isolated queue processing:

```yaml
# From template-hr-data-access.yaml
vs:
  redis:
    usePassword: false
    persistence:
      existingClaim: ""  # generated dynamically
    master:
      persistence:
        existingClaim: ""  # generated dynamically
        storageClass: managed-nfs-storage
        size: "1Gi"
    cluster:
      enabled: false
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:230-240]()

#### Queue Routing Configuration

The registrar service defines multiple queue routes for different data types:

| Queue Name | Route Class | Backend | Purpose |
|------------|-------------|---------|---------|
| `register_queue` | `registrar.route.stac.ItemRoute` | `ItemBackend` | STAC items (products) |
| `register_collection_queue` | `registrar.route.stac.CollectionRoute` | `CollectionBackend` | STAC collections |
| `register_ades_queue` | `registrar.route.json.JSONRoute` | `ADESBackend` | ADES service metadata |
| `register_application_queue` | `registrar.route.json.JSONRoute` | `CWLBackend` | CWL application packages |
| `register_catalogue_queue` | `registrar.route.json.JSONRoute` | `CatalogueBackend` | Catalogue endpoints |
| `seed_queue` | N/A | Seeder | Tile seeding requests |

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:894-947]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:95-164]()

---

## NFS and Persistent Volumes

NFS provides shared filesystem storage for stateful Kubernetes workloads. A central NFS server provisions PersistentVolumeClaims (PVCs) through dynamically-managed StorageClasses.

### NFS Server Configuration

The platform deploys an NFS server and configures StorageClasses for different retention policies:

```yaml
# From hr-storage.yaml
storage:
  nfs:
    server:
      address: "192.168.123.14"
  domain:
    resman:
      storageClass: eoepca-nfs
    proc:
      enabled: true
      storageClass: eoepca-nfs
    userman:
      storageClass: eoepca-nfs
```

**Sources:**
- [system/clusters/creodias/system/storage/hr-storage.yaml:16-28]()

### Storage Classes

Two NFS-backed StorageClasses support different data lifecycle requirements:

| StorageClass | Reclaim Policy | Use Case | Provisioner |
|--------------|----------------|----------|-------------|
| `managed-nfs-storage` | Delete | Ephemeral workspace data | NFS Provisioner |
| `managed-nfs-storage-retain` | Retain | Long-term workspace data | NFS Provisioner |

#### Workspace Storage Class Usage

Workspace components specify `managed-nfs-storage` to dynamically provision PVCs:

```yaml
# From template-hr-data-access.yaml
vs:
  database:
    persistence:
      storageClass: managed-nfs-storage
      size: "100Gi"
  redis:
    master:
      persistence:
        storageClass: managed-nfs-storage
        size: "1Gi"
```

For database storage requiring data retention beyond workspace lifecycle:

```yaml
# From template-hr-resource-catalogue.yaml
db:
  volume_storage_type: managed-nfs-storage-retain
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:223-238]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:19]()

### User Management Persistent Volume

The User Management building block (Gluu/Keycloak) uses a shared PVC for state persistence across multiple components:

**User Management PVC Architecture**

```mermaid
graph TB
    PVC["eoepca-userman-pvc<br/>StorageClass: eoepca-nfs"]
    NFS["NFS Server<br/>192.168.123.14"]
    
    subgraph "User Management Pods"
        OpenDJ["opendj<br/>(LDAP Directory)"]
        OxAuth["oxauth<br/>(OAuth2/OIDC)"]
        OxTrust["oxtrust<br/>(Admin UI)"]
        PDP["pdp-engine<br/>(Policy Engine)"]
        Profile["user-profile<br/>(User Data)"]
    end
    
    NFS -->|"Provisions"| PVC
    PVC -->|"Mount /opt/opendj/config<br/>/opt/opendj/ldif<br/>/opt/opendj/logs"| OpenDJ
    PVC -->|"Mount /opt/gluu/jetty/oxauth"| OxAuth
    PVC -->|"Mount /opt/gluu/jetty/identity"| OxTrust
    PVC -->|"Mount PDP State"| PDP
    PVC -->|"Mount User Profiles"| Profile
```

#### PVC Configuration in User Management

All User Management components reference the same PVC:

```yaml
# From um-login-service.yaml
volumeClaim:
  name: eoepca-userman-pvc
  create: false

opendj:
  volumeClaim:
    name: eoepca-userman-pvc
oxauth:
  volumeClaim:
    name: eoepca-userman-pvc
oxtrust:
  volumeClaim:
    name: eoepca-userman-pvc
```

**Sources:**
- [system/clusters/creodias/user-management/um-login-service.yaml:16-43]()
- [system/clusters/creodias/user-management/um-pdp-engine.yaml:22-24]()
- [system/clusters/creodias/user-management/um-user-profile.yaml:19-21]()

---

## Storage Integration in Key Components

This section illustrates how major platform components integrate the three storage tiers.

### Data Access Service Storage Integration

**Data Access Storage Flow**

```mermaid
graph LR
    subgraph "Inputs"
        User["User Request<br/>(WMS/WCS)"]
        CF["CloudFerro eodata<br/>(Source Data)"]
        MinioWS["MinIO Workspace<br/>(User Data)"]
    end
    
    subgraph "Data Access Components"
        Renderer["renderer<br/>(4 replicas)"]
        Registrar["registrar<br/>(Registration)"]
        Harvester["harvester<br/>(Data Discovery)"]
    end
    
    subgraph "Storage Backends"
        PG["resource-catalogue-db<br/>(Metadata)"]
        Redis["data-access-redis-master<br/>(Queues)"]
        Cache["S3 Cache Bucket<br/>(Rendered Tiles)"]
    end
    
    User -->|"GetMap Request"| Renderer
    Renderer -->|"Read Products"| CF
    Renderer -->|"Read Products"| MinioWS
    Renderer -->|"Query Metadata"| PG
    Renderer -->|"Cache Tiles"| Cache
    
    Harvester -->|"Enqueue Metadata"| Redis
    Redis -->|"Consume Jobs"| Registrar
    Registrar -->|"Store Metadata"| PG
```

Configuration binding these components:

```yaml
# Registrar connects to PostgreSQL and Redis
registrar:
  config:
    defaultBackends:
      - path: registrar_pycsw.backend.ItemBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
    defaultSuccessQueue: seed_queue

# Harvester connects to Redis
harvester:
  config:
    redis:
      host: data-access-redis-master
      port: 6379

# Renderer reads from S3
global:
  storage:
    data:
      data:
        type: S3
        endpoint_url: http://data.cloudferro.com
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:878-960]()

### Workspace API Storage Provisioning

The Workspace API orchestrates storage provisioning across all three tiers when creating a user workspace.

**Workspace Storage Provisioning Sequence**

```mermaid
sequenceDiagram
    participant User
    participant WSAPI as "workspace-api"
    participant MinioAPI as "minio-bucket-api<br/>:8080/bucket"
    participant Flux as "Flux CD"
    participant K8s as "Kubernetes API"
    participant NFS as "NFS Server<br/>192.168.123.14"
    
    User->>WSAPI: "POST /workspaces<br/>{name: eric-workspace}"
    
    WSAPI->>MinioAPI: "POST /bucket<br/>{name: eric-workspace}"
    MinioAPI->>MinioAPI: "Create S3 Bucket"
    MinioAPI-->>WSAPI: "Credentials:<br/>access_key_id,<br/>secret_access_key"
    
    WSAPI->>Flux: "Create HelmRelease<br/>data-access<br/>(from template)"
    WSAPI->>Flux: "Create HelmRelease<br/>resource-catalogue<br/>(from template)"
    
    Flux->>K8s: "Create Namespace<br/>eric-workspace"
    Flux->>K8s: "Deploy PostgreSQL<br/>(with PVC request)"
    Flux->>K8s: "Deploy Redis<br/>(with PVC request)"
    
    K8s->>NFS: "Provision PVC<br/>managed-nfs-storage"
    NFS-->>K8s: "PV Created"
    K8s->>K8s: "Bind PVC to PV"
    K8s->>K8s: "Mount PVC to Pods"
    
    K8s-->>WSAPI: "Workspace Ready"
    WSAPI-->>User: "Workspace Created<br/>eric-workspace"
```

Configuration parameters injected into workspace templates:

```yaml
# Template parameters filled by Workspace API
storage:
  data:
    data:
      access_key_id: {{ access_key_id }}      # From MinIO API
      secret_access_key: {{ secret_access_key }} # From MinIO API
      bucket: {{ bucket }}                     # From MinIO API

vs:
  database:
    persistence:
      storageClass: managed-nfs-storage       # NFS provisioner
      size: "100Gi"
  redis:
    master:
      persistence:
        storageClass: managed-nfs-storage     # NFS provisioner
        size: "1Gi"
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:35-49]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-240]()

---

## Storage Resource Requirements

The following table summarizes typical storage allocations for platform components:

| Component | Storage Type | Size | Reclaim Policy | Purpose |
|-----------|--------------|------|----------------|---------|
| `resource-catalogue-db` | PostgreSQL on NFS | 5Gi | Delete | Global metadata catalogue |
| `eoepca-userman-pvc` | NFS PVC | Variable | Retain | User management state (LDAP, configs) |
| `data-access-redis-master` | Redis on NFS | Variable | Delete | Global registration queues |
| Workspace PostgreSQL | PostgreSQL on NFS | 100Gi | Retain (optional) | User metadata catalogue |
| Workspace Redis | Redis on NFS | 1Gi | Delete | User-specific queues |
| Workspace S3 Bucket | MinIO Object Storage | Unlimited | N/A | User outputs and processing results |
| Cache Bucket | S3 Object Storage | Unlimited | N/A | Rendered WMS/WMTS tiles |

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:20]()
- [system/clusters/creodias/user-management/um-login-service.yaml:17]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:224]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:238]()

---

## Storage Access Patterns

Different platform components exhibit distinct storage access patterns optimized for their workloads:

| Component | Read Pattern | Write Pattern | Consistency | Latency Sensitivity |
|-----------|--------------|---------------|-------------|---------------------|
| Data Access Renderer | High-volume sequential (EO products from S3) | Low-volume (cache tiles) | Eventual | High (user-facing) |
| Resource Catalogue | Random access (metadata queries) | Low-volume (registration) | Strong (ACID) | Medium |
| Registrar | Queue consumption (sequential) | Batch inserts (metadata) | Strong (transactional) | Low |
| Harvester | N/A | Queue writes (sequential) | At-least-once | Low |
| Workspace Database | Random access (user queries) | Low-volume (user data) | Strong (ACID) | Medium |

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:864-876]() (renderer replicas and resources)
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:20-30]() (database tuning)

---

## Summary

The EOEPCA storage architecture provides:

1. **S3 Object Storage**: High-throughput access to Earth Observation products via CloudFerro and MinIO, with per-workspace bucket isolation
2. **PostgreSQL Databases**: ACID-compliant metadata storage for OGC CSW catalogues, with performance tuning for geospatial queries
3. **Redis Queues**: Asynchronous job processing for data registration and tile seeding workflows
4. **NFS Persistent Volumes**: Shared filesystem storage for stateful application data with configurable retention policies

This multi-tier approach separates concerns based on data characteristics: object storage for large binary assets, relational databases for structured metadata, message queues for asynchronous workflows, and shared filesystems for application state. Each tier is provisioned either globally for platform-wide resources or per-workspace for user isolation.