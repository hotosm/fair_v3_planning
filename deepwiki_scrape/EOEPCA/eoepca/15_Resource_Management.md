# Resource Management

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [system/clusters/creodias/resource-management/hr-data-access.yaml](system/clusters/creodias/resource-management/hr-data-access.yaml)
- [system/clusters/creodias/resource-management/hr-registration-api.yaml](system/clusters/creodias/resource-management/hr-registration-api.yaml)
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/hr-resource-catalogue.yaml)
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml](system/clusters/creodias/resource-management/hr-workspace-api.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml)

</details>



## Purpose and Scope

The Resource Management building block provides data cataloging, discovery, access, and workspace management capabilities within the EOEPCA platform. This system enables users to discover Earth Observation data through OGC-compliant catalog services, visualize and access data via standardized web services, and manage isolated workspaces with dedicated storage and catalog instances.

This page covers the overall architecture and integration of Resource Management services. For detailed information on specific components, see:
- [Data Access Services](#5.1) - OGC WMS/WCS/WMTS rendering and visualization
- [Resource Catalogue](#5.2) - pycsw-based metadata catalog
- [Workspace API](#5.3) - Workspace provisioning and orchestration
- [Data Registration and Harvesting](#5.4) - External data ingestion pipeline
- [Multi-Tenant Workspaces](#5.5) - Per-user workspace isolation architecture

## Architecture Overview

The Resource Management building block consists of both global services deployed in the `rm` namespace and per-user workspace services deployed in dedicated namespaces. The architecture supports multi-tenancy through a template-based provisioning system.

**Diagram: Resource Management Component Architecture**

```mermaid
graph TB
    subgraph "rm namespace - Global Services"
        WorkspaceAPI["workspace-api<br/>HelmRelease"]
        GlobalRC["resource-catalogue<br/>HelmRelease"]
        GlobalDA["data-access<br/>HelmRelease"]
        RegAPI["registration-api<br/>HelmRelease"]
        BucketAPI["minio-bucket-api"]
        
        RCDB[("resource-catalogue-db<br/>PostgreSQL")]
        RedisGlobal[("data-access-redis-master<br/>Redis")]
    end
    
    subgraph "External Data Sources"
        CREODIAS["CREODIAS OpenSearch<br/>datahub.creodias.eu"]
    end
    
    subgraph "Storage Layer"
        CloudFerro["CloudFerro S3<br/>data.cloudferro.com"]
        MinIO["MinIO<br/>minio.develop.eoepca.org"]
    end
    
    subgraph "workspace_name namespace"
        WSDA["vs (data-access)<br/>HelmRelease"]
        WSRC["rm-resource-catalogue<br/>HelmRelease"]
        WSGuard["resource-guard<br/>PEP"]
        
        WSDB[("vs-database<br/>PostgreSQL")]
        WSRedis[("vs-redis-master<br/>Redis")]
        WSNFS["PVC managed-nfs-storage"]
        WSBucket["S3 Bucket<br/>workspace_name"]
    end
    
    CREODIAS -->|"harvester polls"| GlobalDA
    GlobalDA -->|"enqueue items"| RedisGlobal
    RedisGlobal -->|"registrar consumes"| GlobalDA
    GlobalDA -->|"write metadata"| RCDB
    GlobalRC -->|"query"| RCDB
    
    WorkspaceAPI -->|"create namespace"| WSDA
    WorkspaceAPI -->|"provision bucket"| BucketAPI
    BucketAPI -->|"create"| WSBucket
    WorkspaceAPI -->|"instantiate templates"| WSRC
    
    WSDA -->|"read/write"| WSBucket
    WSDA -->|"register items"| WSRedis
    WSDA -->|"write metadata"| WSDB
    WSRC -->|"query"| WSDB
    WSDA -->|"persistence"| WSNFS
    WSRC -->|"persistence"| WSNFS
    
    GlobalDA -->|"read data"| CloudFerro
    GlobalDA -->|"cache"| MinIO
    WSDA -->|"read data"| WSBucket
    
    WSGuard -->|"enforce ownership"| WSDA
    WSGuard -->|"enforce ownership"| WSRC
    
    RegAPI -->|"enqueue to"| RedisGlobal
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50]()
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:1-82]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:1-50]()
- [system/clusters/creodias/resource-management/hr-registration-api.yaml:1-37]()

## Global Services Deployment

The global Resource Management services are deployed in the `rm` namespace and provide platform-wide data access and catalog capabilities.

### Global Services Table

| Service | HelmRelease Name | Chart | Version | Purpose |
|---------|-----------------|-------|---------|---------|
| Workspace API | `workspace-api` | `rm-workspace-api` | 1.4.0 | Orchestrates workspace provisioning |
| Resource Catalogue | `resource-catalogue` | `rm-resource-catalogue` | 1.4.0 | Global metadata catalog (pycsw) |
| Data Access | `data-access` | `data-access` | 1.4.0 | Global OGC services for public data |
| Registration API | `registration-api` | `rm-registration-api` | 1.4.0 | Data registration endpoint |

### Global Data Access Configuration

The global `data-access` service provides read-only access to platform-wide data collections stored in CloudFerro S3 and cached in MinIO.

**Key Configuration:**

```yaml
storage:
  data:
    data:
      type: S3
      endpoint_url: http://data.cloudferro.com
      region_name: RegionOne
  cache:
    type: S3
    endpoint_url: "https://cf2.cloudferro.com:8080/cache-bucket"
    bucket: cache-bucket
```

**Ingress Configuration:**

```yaml
ingress:
  hosts:
    - host: data-access.develop.eoepca.org
  tls:
    - hosts:
        - data-access.develop.eoepca.org
      secretName: data-access-tls
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:42-47]()

### Global Resource Catalogue Configuration

The global `resource-catalogue` is based on pycsw and stores metadata in a PostgreSQL database with a 5Gi volume.

**Database Configuration:**

```yaml
db:
  volume_size: 5Gi
  config:
    shared_buffers: 2GB
    effective_cache_size: 6GB
    maintenance_work_mem: 512MB
    max_connections: 300
```

**Catalog Metadata:**

```yaml
pycsw:
  config:
    server:
      url: https://resource-catalogue.develop.eoepca.org/
    manager:
      transactions: "true"
      allowed_ips: "*"
    metadata:
      identification_title: EOEPCA Resource Catalogue
      provider_name: EOEPCA
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:19-31]()
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:44-57]()

## Data Harvesting and Registration Pipeline

The Resource Management system implements a queue-based pipeline for harvesting external data sources and registering metadata into the catalog.

**Diagram: Harvesting and Registration Flow**

```mermaid
sequenceDiagram
    participant Harvester as "harvester<br/>(data-access)"
    participant OpenSearch as "CREODIAS OpenSearch<br/>datahub.creodias.eu"
    participant Redis as "data-access-redis-master<br/>register_queue"
    participant Registrar as "registrar<br/>(data-access)"
    participant Backend as "registrar_pycsw.backend"
    participant DB as "resource-catalogue-db<br/>PostgreSQL"
    participant PyCSW as "pycsw<br/>(resource-catalogue)"
    
    Harvester->>OpenSearch: "Poll OpenSearch API<br/>Sentinel2/Landsat8/Sentinel1"
    OpenSearch-->>Harvester: "Return STAC items<br/>with metadata"
    Harvester->>Harvester: "postprocess_sentinel2()<br/>transform to STAC"
    Harvester->>Redis: "Enqueue to register_queue"
    
    Registrar->>Redis: "Consume from register_queue"
    Redis-->>Registrar: "STAC item"
    
    Registrar->>Registrar: "Route to ItemBackend<br/>based on queue"
    Registrar->>Backend: "ItemBackend.register()"
    Backend->>DB: "INSERT metadata<br/>(ISO 19115 format)"
    
    Registrar->>Redis: "Enqueue to seed_queue<br/>(success callback)"
    
    PyCSW->>DB: "Query metadata<br/>via CSW/OpenSearch"
    DB-->>PyCSW: "Return records"
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:949-1087]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:878-947]()

### Harvester Configuration

The `harvester` component polls external OpenSearch endpoints to discover and ingest data. It is configured with multiple harvesters for different satellite missions.

**Harvester Definitions:**

| Harvester | Data Source | Queue | Postprocessor |
|-----------|-------------|-------|---------------|
| `Sentinel2` | `datahub.creodias.eu/resto/api/collections/Sentinel2` | `register` | `harvester_eoepca.postprocess.postprocess_sentinel2` |
| `Landsat8` | `datahub.creodias.eu/resto/api/collections/Landsat8` | `register` | `harvester_eoepca.postprocess.postprocess_landsat8` |
| `Sentinel1-GRD` | `datahub.creodias.eu/resto/api/collections/Sentinel1` | `register` | `harvester_eoepca.postprocess.postprocess_sentinel1` |
| `Sentinel3` | `datahub.creodias.eu/resto/api/collections/Sentinel3` | `register` | `harvester_eoepca.postprocess.postprocess_sentinel3` |

**Example Harvester Configuration:**

```yaml
harvesters:
  Sentinel2:
    resource:
      type: OpenSearch
      opensearch:
        url: https://datahub.creodias.eu/resto/api/collections/Sentinel2/describe.xml
        query:
          time:
            begin: 2019-09-10T00:00:00Z
            end: 2019-09-11T00:00:00Z
          bbox: 14.9,47.7,16.4,48.7
    postprocessors:
      - type: external
        process: harvester_eoepca.postprocess.postprocess_sentinel2
    queue: register
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:962-985]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:986-1009]()

### Registrar Queue Routes

The `registrar` component consumes items from Redis queues and routes them to backend handlers based on the queue name. Each route is configured with specific backends that write metadata to the catalog database.

**Registrar Routes:**

| Route | Queue | Backend | Purpose |
|-------|-------|---------|---------|
| `collections` | `register_collection_queue` | `registrar_pycsw.backend.CollectionBackend` | Register STAC collections |
| `items` | `register_queue` | `registrar_pycsw.backend.ItemBackend` | Register STAC items |
| `ades` | `register_ades_queue` | `registrar_pycsw.backend.ADESBackend` | Register ADES services |
| `application` | `register_application_queue` | `registrar_pycsw.backend.CWLBackend` | Register CWL applications |
| `catalogue` | `register_catalogue_queue` | `registrar_pycsw.backend.CatalogueBackend` | Register catalog endpoints |

**Example Route Configuration:**

```yaml
routes:
  collections:
    path: registrar.route.stac.Collection
    queue: register_collection_queue
    replace: true
    backends:
      - path: registrar_pycsw.backend.CollectionBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:894-947]()

## Product Types and Collections

The Data Access service defines product types that map satellite data assets to coverage types and browsing configurations. These definitions enable OGC WMS/WCS rendering of specific bands and derived products.

**Diagram: Product Type to Coverage Mapping**

```mermaid
graph LR
    subgraph "Collections"
        S2L2A["S2L2A Collection"]
        S2L1C["S2L1C Collection"]
        L8L1TP["L8L1TP Collection"]
        S1IWGRD1C["S1IWGRD1C Collection"]
    end
    
    subgraph "Product Types"
        S2MSI2A["S2MSI2A<br/>filter: s2:product_type=S2MSI2A"]
        S2MSI1C["S2MSI1C<br/>filter: s2:product_type=S2MSI1C"]
        L8MSI1TP["L8MSI1TP<br/>filter: platform=landsat-8<br/>landsat:correction=L1TP"]
        S1GRD["S1IWGRD1C_VVVH<br/>filter: sar:product_type=GRD"]
    end
    
    subgraph "Coverages"
        S2L2A_B04["S2L2A_B04<br/>asset: B04"]
        S2L2A_B03["S2L2A_B03<br/>asset: B03"]
        S2L2A_B02["S2L2A_B02<br/>asset: B02"]
        L8_B4["L8L1TP_B04<br/>asset: SR_B4"]
        S1_VV["S1IWGRD1C_VV<br/>asset: vv"]
    end
    
    subgraph "Browses"
        TrueColor["TRUE_COLOR<br/>RGB: B04/B03/B02<br/>range: [0,4000]"]
        FalseColor["FALSE_COLOR<br/>RGB: B08/B04/B03"]
        NDVI["NDVI<br/>expression: (B08-B04)/(B08+B04)"]
    end
    
    S2L2A --> S2MSI2A
    S2L1C --> S2MSI1C
    L8L1TP --> L8MSI1TP
    S1IWGRD1C --> S1GRD
    
    S2MSI2A --> S2L2A_B04
    S2MSI2A --> S2L2A_B03
    S2MSI2A --> S2L2A_B02
    L8MSI1TP --> L8_B4
    S1GRD --> S1_VV
    
    S2MSI2A --> TrueColor
    S2MSI2A --> FalseColor
    S2MSI2A --> NDVI
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:511-671]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:217-290]()

### Example Product Type Definition

The `S2MSI2A` product type for Sentinel-2 Level 2A data demonstrates the mapping structure:

```yaml
productTypes:
  - name: S2MSI2A
    filter:
      s2:product_type: S2MSI2A
    collections:
      - S2L2A
    coverages:
      S2L2A_B04:
        assets:
          - B04
      S2L2A_B03:
        assets:
          - B03
    defaultBrowse: TRUE_COLOR
    browses:
      TRUE_COLOR:
        asset: visual-10m
        red:
          expression: B04
          range: [0, 4000]
          nodata: 0
        green:
          expression: B03
          range: [0, 4000]
          nodata: 0
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:593-651]()

## Workspace Provisioning Architecture

The Workspace API orchestrates the creation of isolated, per-user environments by instantiating HelmRelease templates with user-specific values. This enables multi-tenant resource isolation.

**Diagram: Workspace Provisioning Flow**

```mermaid
sequenceDiagram
    participant User
    participant WorkspaceAPI as "workspace-api<br/>(rm namespace)"
    participant K8s as "Kubernetes API"
    participant BucketAPI as "minio-bucket-api<br/>bucket endpoint"
    participant MinIO as "MinIO S3"
    participant Flux as "Flux HelmController"
    participant Templates as "workspace-charts<br/>ConfigMap"
    
    User->>WorkspaceAPI: "POST /workspaces<br/>{name: 'eric-workspace'}"
    WorkspaceAPI->>K8s: "Create namespace<br/>'eric-workspace'"
    K8s-->>WorkspaceAPI: "Namespace created"
    
    WorkspaceAPI->>BucketAPI: "POST /bucket<br/>{bucket: 'eric-workspace'}"
    BucketAPI->>MinIO: "Create bucket<br/>with access keys"
    MinIO-->>BucketAPI: "Bucket created<br/>credentials returned"
    BucketAPI-->>WorkspaceAPI: "access_key_id, secret_access_key"
    
    WorkspaceAPI->>K8s: "Create Secret 'bucket'<br/>with S3 credentials"
    
    WorkspaceAPI->>Templates: "Read template-hr-data-access.yaml"
    Templates-->>WorkspaceAPI: "Template with placeholders"
    
    WorkspaceAPI->>WorkspaceAPI: "Substitute:<br/>{{ workspace_name }} -> 'eric-workspace'<br/>{{ bucket }} -> 'eric-workspace'<br/>{{ access_key_id }} -> 'AKIAIOSFODNN7EXAMPLE'<br/>{{ secret_access_key }} -> 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'"
    
    WorkspaceAPI->>K8s: "Apply HelmRelease<br/>'vs' in eric-workspace"
    WorkspaceAPI->>K8s: "Apply HelmRelease<br/>'rm-resource-catalogue' in eric-workspace"
    WorkspaceAPI->>K8s: "Apply HelmRelease<br/>'resource-guard' in eric-workspace"
    
    Flux->>K8s: "Detect new HelmReleases"
    Flux->>Flux: "Deploy data-access chart"
    Flux->>Flux: "Deploy resource-catalogue chart"
    
    K8s-->>WorkspaceAPI: "Workspace provisioned"
    WorkspaceAPI-->>User: "200 OK<br/>workspace details"
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-50]()

### Workspace API Configuration

The Workspace API is configured with template locations, bucket provisioning endpoints, and protection settings.

**Key Configuration Values:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `prefixForName` | `develop-user` | Prefix for workspace names |
| `workspaceSecretName` | `bucket` | Name of S3 credential secret |
| `namespaceForBucketResource` | `rm` | Namespace for bucket operator |
| `s3Endpoint` | `https://minio.develop.eoepca.org` | MinIO endpoint |
| `bucketEndpointUrl` | `http://minio-bucket-api:8080/bucket` | Bucket provisioning API |
| `workspaceChartsConfigMap` | `workspace-charts` | ConfigMap with HelmRelease templates |
| `pepBaseUrl` | `http://workspace-api-pep:5576/resources` | PEP for resource protection |
| `autoProtectionEnabled` | `True` | Automatically register resources with PEP |

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:35-49]()

### Template Structure

The workspace templates use placeholder substitution to generate user-specific HelmReleases. The template system replaces variables like `{{ workspace_name }}`, `{{ bucket }}`, `{{ access_key_id }}`, and `{{ secret_access_key }}`.

**Template Placeholders:**

```yaml
metadata:
  name: vs  # fixed name per workspace
spec:
  values:
    global:
      storage:
        data:
          data:
            endpoint_url: https://minio.develop.eoepca.org
            access_key_id: {{ access_key_id }}
            secret_access_key: {{ secret_access_key }}
            bucket: {{ bucket }}
      metadata:
        title: Data Access Service - {{ workspace_name }}
    vs:
      registrar:
        config:
          routes:
            items:
              backends:
                - path: "registrar_pycsw.backend.ItemBackend"
                  kwargs:
                    ows_url: "https://data-access.{{ workspace_name }}.develop.eoepca.org/ows"
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:56-58]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:108-112]()

## Workspace-Specific Services

Each workspace receives its own instances of data-access and resource-catalogue services, isolated within a dedicated namespace.

### Workspace Data Access

The workspace-specific `vs` (view server) HelmRelease provides OGC services over the user's S3 bucket.

**Key Differences from Global Service:**

| Aspect | Global | Workspace |
|--------|--------|-----------|
| Namespace | `rm` | `{{ workspace_name }}` |
| S3 Endpoint | `http://data.cloudferro.com` (read-only) | `https://minio.develop.eoepca.org` (read-write) |
| Bucket | CloudFerro public buckets | User-specific bucket |
| Redis | `data-access-redis-master` | `vs-redis-master` |
| Database | Shared PostgreSQL | Dedicated `vs-database` PostgreSQL |
| Ingress | `data-access.develop.eoepca.org` | `data-access.{{ workspace_name }}.develop.eoepca.org` |
| Replicas | `renderer: 4` | `renderer: 1` |

**Workspace Storage Configuration:**

```yaml
storage:
  data:
    data:
      type: "S3"
      endpoint_url: https://minio.develop.eoepca.org
      access_key_id: {{ access_key_id }}
      secret_access_key: {{ secret_access_key }}
      bucket: {{ bucket }}
      region_name: RegionOne
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:70-82]()

### Workspace Resource Catalogue

Each workspace has a dedicated pycsw instance with its own PostgreSQL database for storing user-specific metadata.

**Workspace Catalogue Configuration:**

```yaml
global:
  namespace: "{{ workspace_name }}"
db:
  volume_storage_type: managed-nfs-storage-retain
pycsw:
  config:
    server:
      url: "https://resource-catalogue.{{ workspace_name }}.develop.eoepca.org"
      federatedcatalogues: https://resource-catalogue.develop.eoepca.org/collections/S2MSI2A
    metadata:
      identification_title: Resource Catalogue - {{ workspace_name }}
```

**Federated Search:**

The workspace catalogue is configured with `federatedcatalogues` pointing to the global catalogue, enabling federated search across the user's private data and platform-wide public data.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:16-36]()

### Workspace Harvester

The workspace data-access includes a harvester configured to monitor a static STAC catalog in the user's S3 bucket.

**Workspace Harvester Configuration:**

```yaml
harvester:
  config:
    redis:
      host: vs-redis-master
      port: 6379
    harvesters:
      harvest-bucket-catalog:
        queue: "register_queue"
        resource:
          type: "STACCatalog"
          staccatalog:
            filesystem: s3bucket
            root_path: "/home/catalog.json"
    filesystems:
      s3bucket:
        type: s3
        s3:
          access_key_id: {{ access_key_id }}
          secret_access_key: {{ secret_access_key }}
          endpoint_url: https://minio.develop.eoepca.org
```

This allows users to maintain a `catalog.json` file in their bucket that the harvester will periodically process and register into their workspace catalogue.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:166-195]()

## Storage Architecture

Resource Management uses a multi-tier storage architecture combining S3 object storage, PostgreSQL databases, Redis queues, and NFS persistent volumes.

**Diagram: Storage Tiers and Access Patterns**

```mermaid
graph TB
    subgraph "S3 Object Storage"
        CloudFerroS3["CloudFerro S3<br/>data.cloudferro.com<br/>(Public EO Data)"]
        MinIOCache["MinIO Cache<br/>cf2.cloudferro.com:8080/cache-bucket<br/>(Rendered tiles)"]
        MinIOWorkspace["MinIO User Buckets<br/>minio.develop.eoepca.org<br/>(User data)"]
    end
    
    subgraph "Database Layer"
        GlobalRCDB["resource-catalogue-db<br/>PostgreSQL<br/>(Global metadata)"]
        WorkspaceDB["vs-database<br/>PostgreSQL per workspace<br/>(User metadata)"]
    end
    
    subgraph "Queue Layer"
        GlobalRedis["data-access-redis-master<br/>(Global queues:<br/>register, register_collection,<br/>register_ades, seed)"]
        WorkspaceRedis["vs-redis-master<br/>(Workspace queues:<br/>register_queue,<br/>register_collection_queue)"]
    end
    
    subgraph "Persistent Volumes"
        WSNFS["managed-nfs-storage<br/>PVC per workspace<br/>(Database + Redis persistence)"]
    end
    
    subgraph "Global Services"
        GlobalRenderer["renderer<br/>(4 replicas)"]
        GlobalRegistrar["registrar"]
        GlobalHarvester["harvester"]
    end
    
    subgraph "Workspace Services"
        WSRenderer["renderer<br/>(1 replica)"]
        WSRegistrar["registrar"]
        WSHarvester["harvester"]
    end
    
    GlobalHarvester -->|"Harvest"| CloudFerroS3
    GlobalRenderer -->|"Read STAC items"| CloudFerroS3
    GlobalRenderer -->|"Write tiles"| MinIOCache
    GlobalHarvester -->|"Enqueue"| GlobalRedis
    GlobalRegistrar -->|"Consume"| GlobalRedis
    GlobalRegistrar -->|"Write metadata"| GlobalRCDB
    
    WSHarvester -->|"Read catalog.json"| MinIOWorkspace
    WSRenderer -->|"Read STAC items"| MinIOWorkspace
    WSHarvester -->|"Enqueue"| WorkspaceRedis
    WSRegistrar -->|"Consume"| WorkspaceRedis
    WSRegistrar -->|"Write metadata"| WorkspaceDB
    
    WorkspaceDB -->|"Persist"| WSNFS
    WorkspaceRedis -->|"Persist"| WSNFS
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-53]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-240]()

## Registration API

The Registration API provides an HTTP endpoint for external clients to submit data for registration into the catalog system.

**Registration API Configuration:**

```yaml
fullnameOverride: registration-api
ingress:
  enabled: true
  hosts:
    - host: registration-api-open.develop.eoepca.org
      paths: ["/"]
workspaceK8sNamespace: "rm"
redisServiceName: "data-access-redis-master"
```

The Registration API enqueues items to the same Redis instance used by the harvester, allowing both automated harvesting and manual registration to flow through the same registrar pipeline.

**Sources:**
- [system/clusters/creodias/resource-management/hr-registration-api.yaml:17-36]()

## Resource Scaling Configuration

The global and workspace services are configured with different resource allocations based on expected load.

### Global Service Resources

| Component | Replicas | CPU Request | Memory Request | Memory Limit |
|-----------|----------|-------------|----------------|--------------|
| `renderer` | 4 | 0.5 | 1Gi | 3Gi |
| `registrar` | 1 | 100m | 100Mi | - |
| `harvester` | 1 | 100m | 100Mi | - |
| `scheduler` | 1 | 100m | 100Mi | - |

### Workspace Service Resources

| Component | Replicas | CPU Request | Memory Request | Memory Limit |
|-----------|----------|-------------|----------------|--------------|
| `renderer` | 1 | 100m | 300Mi | 3Gi |
| `registrar` | 1 | 100m | 100Mi | - |
| `harvester` | 1 | 100m | 100Mi | - |
| `scheduler` | 1 | 100m | 100Mi | - |
| `ingestor` | 0 | - | - | - |
| `preprocessor` | 0 | - | - | - |
| `cache` | 0 | - | - | - |
| `seeder` | 0 | - | - | - |

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:865-876]()
- [system/clusters/creodias/resource-management/hr-data-access.yaml:878-886]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:70-82]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:251-268]()

## Integration with Other Building Blocks

Resource Management integrates with User Management for authentication/authorization and with Processing & Chaining for data discovery and access.

### Integration Points

| Integration | Component | Purpose |
|-------------|-----------|---------|
| **User Management** | `workspace-api-pep` | Protects workspace API endpoints with UMA authentication |
| **User Management** | `resource-guard` (per workspace) | Protects workspace-specific services |
| **User Management** | `umaClientSecretName: rm-uma-user-agent` | OAuth2 client credentials for workspace API |
| **Processing & Chaining** | Resource Catalogue CSW | ADES queries catalog to discover input data |
| **Processing & Chaining** | Data Access OGC services | ADES stage-in from WCS endpoints |
| **Processing & Chaining** | S3 Buckets | ADES reads inputs from and writes outputs to workspace buckets |

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:44-49]()

## Summary

The Resource Management building block provides:

1. **Global data access** via OGC WMS/WCS/WMTS services over CloudFerro S3 storage
2. **Platform-wide catalog** using pycsw for metadata discovery via CSW and OpenSearch
3. **Automated harvesting** from external OpenSearch endpoints (CREODIAS)
4. **Queue-based registration** pipeline with Redis and configurable backends
5. **Multi-tenant workspaces** with isolated namespaces, S3 buckets, databases, and catalogs
6. **Template-based provisioning** for per-user service instantiation
7. **Federated search** enabling users to query both private and public data
8. **Integration with identity services** for policy-based access control

For implementation details of individual components, refer to the subsystem pages listed at the beginning of this document.