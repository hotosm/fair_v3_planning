# Database Systems

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



This document describes the database systems deployed within the EOEPCA platform, including PostgreSQL for metadata storage and Redis for message queuing. These databases support the Resource Management building block, providing persistent storage for catalogue metadata and asynchronous processing pipelines.

For information about object storage (S3/MinIO) and NFS file systems, see [S3 Storage Architecture](#7.1) and [NFS and Persistent Volumes](#7.3). For details on how databases are used within specific services, see [Resource Catalogue](#5.2) and [Data Access Services](#5.1).

## Database Architecture Overview

The EOEPCA platform deploys two primary database technologies:

| Database Type | Technology | Primary Use Cases | Deployment Pattern |
|---------------|------------|-------------------|-------------------|
| Relational Database | PostgreSQL | ISO 19115 metadata storage, pycsw catalogue backend | Global instance + per-workspace instances |
| In-Memory Database | Redis | Message queuing, registration pipelines, cache coordination | Global instance + per-workspace instances |

```mermaid
graph TB
    subgraph "Global Resource Management (rm namespace)"
        DataAccess["Data Access Service<br/>vs-renderer, vs-registrar"]
        ResourceCat["Resource Catalogue<br/>pycsw"]
        Registrar["Registrar Service<br/>Multiple Routes"]
        Harvester["Data Harvester<br/>OpenSearch Poller"]
        
        GlobalRedis[("Redis<br/>data-access-redis-master<br/>Port: 6379")]
        GlobalPostgres[("PostgreSQL<br/>resource-catalogue-db<br/>Database: pycsw")]
    end
    
    subgraph "User Workspace (e.g., eric-workspace)"
        WSDataAccess["Workspace Data Access<br/>vs-renderer, vs-registrar"]
        WSResourceCat["Workspace Resource Catalogue<br/>pycsw"]
        
        WSRedis[("Redis<br/>vs-redis-master<br/>Port: 6379")]
        WSPostgres[("PostgreSQL<br/>resource-catalogue-db<br/>Database: pycsw")]
    end
    
    Harvester -->|"Enqueue Items"| GlobalRedis
    GlobalRedis -->|"Consume: register_queue<br/>register_collection_queue<br/>register_ades_queue"| Registrar
    Registrar -->|"Write Metadata<br/>repository_database_uri"| GlobalPostgres
    ResourceCat -->|"Query Metadata"| GlobalPostgres
    DataAccess -->|"Seeding Queue<br/>seed_queue"| GlobalRedis
    
    WSDataAccess -->|"Register Items"| WSRedis
    WSRedis -->|"Consume: register_queue"| WSDataAccess
    WSDataAccess -->|"Write Metadata"| WSPostgres
    WSResourceCat -->|"Query Metadata"| WSPostgres
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:886-948]()
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:19-27]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:93-164]()

## PostgreSQL Databases

### Resource Catalogue Database

The primary PostgreSQL deployment is `resource-catalogue-db`, which serves as the backend for pycsw-based catalogue services. This database stores ISO 19115/19139 metadata records for Earth Observation products, collections, applications, and services.

**Database Connection Pattern:**

```mermaid
graph LR
    subgraph "Service Connections"
        Registrar["Registrar Service<br/>ItemBackend<br/>CollectionBackend<br/>ADESBackend<br/>CWLBackend"]
        PyCSW["pycsw Service<br/>OGC CSW/OpenSearch"]
    end
    
    PostgresDB[("PostgreSQL<br/>Host: resource-catalogue-db<br/>Port: 5432<br/>Database: pycsw<br/>User: postgres")]
    
    Registrar -->|"postgresql://postgres:mypass@<br/>resource-catalogue-db/pycsw"| PostgresDB
    PyCSW -->|"Query/Insert Metadata"| PostgresDB
```

**Connection String Format:**

The standard connection URI used throughout the platform follows this pattern:

```
postgresql://postgres:mypass@resource-catalogue-db/pycsw
```

This connection string appears in multiple registrar backend configurations for different metadata types.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:891-902]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:110-120]()

### Global Database Configuration

The global resource catalogue database is configured with performance tuning parameters suitable for metadata-heavy workloads:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `volume_size` | 5Gi | Persistent volume size |
| `shared_buffers` | 2GB | Memory for caching data |
| `effective_cache_size` | 6GB | Planner estimate of kernel cache |
| `maintenance_work_mem` | 512MB | Memory for maintenance operations |
| `checkpoint_completion_target` | 0.9 | Spread checkpoint writes |
| `wal_buffers` | 16MB | Write-ahead log buffer |
| `work_mem` | 4MB | Memory per query operation |
| `default_statistics_target` | 100 | Statistics sampling target |

These parameters optimize PostgreSQL for read-heavy catalogue queries while maintaining efficient write performance for metadata ingestion.

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:19-31]()

### Workspace Database Instances

Each user workspace provisions its own isolated PostgreSQL database instance. The workspace-specific databases use identical connection patterns but are deployed within the workspace namespace:

**Workspace Database Deployment:**

```mermaid
graph TB
    subgraph "Workspace: eric-workspace"
        WSRegistrar["Registrar Routes:<br/>- items<br/>- collections<br/>- ades<br/>- application<br/>- catalogue"]
        WSPyCSW["pycsw Service"]
        
        WSPG[("PostgreSQL<br/>resource-catalogue-db<br/>Database: pycsw<br/>Namespace: eric-workspace")]
        
        WSPVC["PersistentVolumeClaim<br/>StorageClass: managed-nfs-storage-retain"]
    end
    
    WSRegistrar -->|"Write Metadata"| WSPG
    WSPyCSW -->|"Query CSW/OpenSearch"| WSPG
    WSPG -->|"Persist Data"| WSPVC
```

Workspace databases use the `managed-nfs-storage-retain` storage class to ensure data persistence even after workspace deletion.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:17-19]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-228]()

### Registrar Backend Routes

The Registrar service connects to PostgreSQL through multiple specialized backend classes, each handling different metadata types:

| Backend Class | Queue Name | Metadata Type | Purpose |
|---------------|------------|---------------|---------|
| `registrar_pycsw.backend.ItemBackend` | `register_queue` | STAC Items | EO product metadata |
| `registrar_pycsw.backend.CollectionBackend` | `register_collection_queue` | STAC Collections | Collection-level metadata |
| `registrar_pycsw.backend.ADESBackend` | `register_ades_queue` | ADES Metadata | Processing service registration |
| `registrar_pycsw.backend.CWLBackend` | `register_application_queue` | CWL Applications | Application package metadata |
| `registrar_pycsw.backend.CatalogueBackend` | `register_catalogue_queue` | Catalogue References | Federated catalogue links |
| `registrar_pycsw.backend.JSONBackend` | `register_json_queue` | Generic JSON | Custom metadata records |
| `registrar_pycsw.backend.XMLBackend` | `register_xml_queue` | Generic XML | ISO 19139 XML records |

Each backend receives the `repository_database_uri` parameter pointing to the PostgreSQL database.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:888-947]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:100-164]()

## Redis Instances

### Message Queue Architecture

Redis serves as the message broker for asynchronous metadata registration and data processing workflows. The platform deploys Redis in a master configuration without clustering.

```mermaid
graph TB
    subgraph "Registration Pipeline"
        Harvester["Data Harvester<br/>OpenSearch Client"]
        RegistrationAPI["Registration API<br/>HTTP Endpoint"]
        
        Redis[("Redis Master<br/>Host: data-access-redis-master<br/>Port: 6379<br/>Password: disabled")]
        
        Registrar["Registrar Workers<br/>Consume Queues"]
    end
    
    subgraph "Queue Types"
        RegQueue["register_queue<br/>(STAC Items)"]
        CollQueue["register_collection_queue<br/>(Collections)"]
        AdesQueue["register_ades_queue<br/>(ADES Services)"]
        AppQueue["register_application_queue<br/>(Applications)"]
        CatQueue["register_catalogue_queue<br/>(Catalogues)"]
        SeedQueue["seed_queue<br/>(Cache Seeding)"]
        JSONQueue["register_json_queue<br/>(JSON Metadata)"]
        XMLQueue["register_xml_queue<br/>(XML Metadata)"]
    end
    
    Harvester -->|"Enqueue Discovered Items"| Redis
    RegistrationAPI -->|"Enqueue User Submissions"| Redis
    
    Redis --> RegQueue
    Redis --> CollQueue
    Redis --> AdesQueue
    Redis --> AppQueue
    Redis --> CatQueue
    Redis --> SeedQueue
    Redis --> JSONQueue
    Redis --> XMLQueue
    
    RegQueue --> Registrar
    CollQueue --> Registrar
    AdesQueue --> Registrar
    AppQueue --> Registrar
    CatQueue --> Registrar
    JSONQueue --> Registrar
    XMLQueue --> Registrar
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:959-960]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:893-947]()

### Global Redis Configuration

The global Redis instance is deployed as `data-access-redis-master` in the `rm` namespace:

**Connection Parameters:**
- **Host:** `data-access-redis-master`
- **Port:** `6379`
- **Authentication:** Disabled (`usePassword: false`)
- **Persistence:** Enabled with NFS-backed PersistentVolumeClaim

The harvester service connects to this Redis instance to enqueue metadata discovered from external OpenSearch endpoints:

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:958-960]()
- [system/clusters/creodias/resource-management/hr-registration-api.yaml:36]()

### Workspace Redis Instances

Each workspace deploys an isolated Redis instance named `vs-redis-master` within the workspace namespace:

| Configuration | Global Instance | Workspace Instance |
|---------------|-----------------|-------------------|
| Service Name | `data-access-redis-master` | `vs-redis-master` |
| Namespace | `rm` | `<workspace-name>` |
| Authentication | Disabled | Disabled |
| Clustering | Disabled | Disabled |
| Persistence | Enabled | Enabled |
| Storage Class | N/A | `managed-nfs-storage` |
| Volume Size | Default | 1Gi |

Workspace Redis instances handle metadata registration for user-uploaded data and processing results within the workspace boundary.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:175-177]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:230-240]()

### Queue-Based Registration Flow

The registration pipeline uses Redis queues to decouple metadata production from consumption:

```mermaid
sequenceDiagram
    participant Harvester as Data Harvester
    participant Redis as Redis Queue
    participant Registrar as Registrar Worker
    participant Postgres as PostgreSQL
    participant PyCSW as pycsw Service
    
    Harvester->>Harvester: Poll OpenSearch Endpoint
    Harvester->>Harvester: Post-process Metadata
    Harvester->>Redis: LPUSH register_queue<br/>(STAC Item JSON)
    
    loop Worker Poll
        Registrar->>Redis: BRPOP register_queue
        Redis-->>Registrar: Pop STAC Item
    end
    
    Registrar->>Registrar: Route to ItemBackend
    Registrar->>Postgres: INSERT INTO records<br/>(ISO 19115 XML)
    Postgres-->>Registrar: Success
    
    opt Success Queue Configured
        Registrar->>Redis: LPUSH seed_queue
    end
    
    PyCSW->>Postgres: SELECT * FROM records<br/>WHERE ...
    Postgres-->>PyCSW: Metadata Records
```

The `defaultSuccessQueue` configuration determines which queue receives successfully processed items for downstream operations like cache seeding.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:893-903]()

## Database Persistence and Storage

### Persistent Volume Configuration

Both PostgreSQL and Redis databases use PersistentVolumeClaims to ensure data durability:

**Global Resource Catalogue Database:**

The global PostgreSQL instance uses a configurable volume size (default 5Gi) and can be backed by various storage classes.

**Workspace Databases:**

```mermaid
graph LR
    subgraph "Workspace Persistence"
        PostgresPVC["PostgreSQL PVC<br/>StorageClass: managed-nfs-storage<br/>Size: 100Gi"]
        RedisPVC["Redis PVC<br/>StorageClass: managed-nfs-storage<br/>Size: 1Gi"]
        
        NFSStorage["NFS Server<br/>192.168.123.14"]
    end
    
    PostgresPVC --> NFSStorage
    RedisPVC --> NFSStorage
```

The workspace databases use the `managed-nfs-storage` storage class, which provisions volumes from the centralized NFS server. This ensures data survives pod restarts and enables workspace data portability.

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:20]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-240]()
- [system/clusters/creodias/system/storage/hr-storage.yaml:19-20]()

### Database Resource Limits

Services connecting to databases have defined resource requests and limits:

**Registrar Service Resources:**

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 100m | Not specified |
| Memory | 100Mi | Not specified |

**Scheduler Service Resources:**

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 100m | Not specified |
| Memory | 100Mi | Not specified |

These conservative resource allocations reflect the lightweight nature of the queue-based registration pipeline.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:883-886]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:1140-1143]()

## Connection String Patterns

### PostgreSQL Connection URIs

All registrar backends use consistent PostgreSQL connection strings with the following pattern:

```
postgresql://<username>:<password>@<host>/<database>
```

**Example Configurations:**

| Context | Connection String |
|---------|-------------------|
| Global Registrar | `postgresql://postgres:mypass@resource-catalogue-db/pycsw` |
| Workspace Registrar | `postgresql://postgres:mypass@resource-catalogue-db/pycsw` |

Note: The host `resource-catalogue-db` resolves differently depending on the Kubernetes namespace context. In the global `rm` namespace, it points to the global database service. In a workspace namespace, it resolves to the workspace-specific database.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:891]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:110]()

### Redis Connection Configuration

Redis connections use host/port configuration rather than URIs:

```yaml
redis:
  host: data-access-redis-master  # or vs-redis-master for workspaces
  port: 6379
```

The Registration API service also connects to the global Redis instance by referencing the service name:

**Registration API Redis Configuration:**

```yaml
redisServiceName: "data-access-redis-master"
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:958-960]()
- [system/clusters/creodias/resource-management/hr-registration-api.yaml:36]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:175-177]()

## Multi-Tenant Database Isolation

The platform implements database-level multi-tenancy by deploying separate database instances per workspace:

```mermaid
graph TB
    subgraph "Global Namespace: rm"
        GlobalRedis[("Redis<br/>data-access-redis-master")]
        GlobalPG[("PostgreSQL<br/>resource-catalogue-db")]
    end
    
    subgraph "Workspace: alice-workspace"
        AliceRedis[("Redis<br/>vs-redis-master")]
        AlicePG[("PostgreSQL<br/>resource-catalogue-db")]
    end
    
    subgraph "Workspace: bob-workspace"
        BobRedis[("Redis<br/>vs-redis-master")]
        BobPG[("PostgreSQL<br/>resource-catalogue-db")]
    end
    
    subgraph "Workspace: charlie-workspace"
        CharlieRedis[("Redis<br/>vs-redis-master")]
        CharliePG[("PostgreSQL<br/>resource-catalogue-db")]
    end
    
    style GlobalRedis fill:#f9f9f9
    style GlobalPG fill:#f9f9f9
    style AliceRedis fill:#e8f5e9
    style AlicePG fill:#e8f5e9
    style BobRedis fill:#e8f5e9
    style BobPG fill:#e8f5e9
    style CharlieRedis fill:#e8f5e9
    style CharliePG fill:#e8f5e9
```

This isolation ensures:
- **Data Privacy:** Each user's metadata is stored in a dedicated database instance
- **Resource Isolation:** Database load from one workspace does not impact others
- **Independent Scaling:** Workspace databases can be sized according to user needs
- **Namespace-Level Access Control:** Kubernetes RBAC restricts database access to workspace namespace

The Workspace API orchestrates database provisioning by instantiating HelmRelease templates with user-specific parameters.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-269]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68]()