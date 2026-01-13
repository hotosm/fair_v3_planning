# Multi-Tenant Workspaces

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

This document describes the multi-tenant workspace architecture in EOEPCA, which enables isolated, per-user environments for data storage, processing, and cataloging. Each workspace is provisioned on-demand with dedicated S3 storage, a private resource catalogue, data access services, and policy-based access control.

For information about the Workspace API that orchestrates this provisioning, see [Workspace API](#5.3). For details on policy enforcement and access control mechanisms, see [Policy Enforcement (PEP/PDP)](#4.3).

## Architecture Overview

The workspace system provides isolated environments through a template-based provisioning mechanism. When a user requests a workspace, the Workspace API instantiates HelmRelease templates with user-specific values to create dedicated Kubernetes resources.

```mermaid
graph TB
    subgraph "Global Layer (rm namespace)"
        WS_API["Workspace API<br/>workspace-api"]
        GlobalRC["Global Resource Catalogue<br/>resource-catalogue"]
        GlobalDA["Global Data Access<br/>data-access"]
        BucketAPI["Bucket Provisioning<br/>minio-bucket-api"]
        Harbor["Container Registry<br/>Harbor"]
        Templates["Template ConfigMap<br/>workspace-charts"]
    end
    
    subgraph "User Workspace: eric-workspace"
        NS1["Kubernetes Namespace<br/>eric-workspace"]
        RG1["Resource Guard<br/>eric-specific PEP"]
        RC1["Resource Catalogue<br/>rm-resource-catalogue"]
        DA1["Data Access Service<br/>vs"]
        Bucket1["S3 Bucket<br/>eric-workspace"]
        PVC1["Persistent Volume<br/>NFS storage"]
    end
    
    subgraph "User Workspace: alice-workspace"
        NS2["Kubernetes Namespace<br/>alice-workspace"]
        RG2["Resource Guard<br/>alice-specific PEP"]
        RC2["Resource Catalogue<br/>rm-resource-catalogue"]
        DA2["Data Access Service<br/>vs"]
        Bucket2["S3 Bucket<br/>alice-workspace"]
        PVC2["Persistent Volume<br/>NFS storage"]
    end
    
    WS_API -->|"1. Read templates"| Templates
    WS_API -->|"2. Create namespace"| NS1
    WS_API -->|"3. Provision bucket"| BucketAPI
    BucketAPI --> Bucket1
    WS_API -->|"4. Configure registry"| Harbor
    WS_API -->|"5. Instantiate services"| RG1
    WS_API -->|"5. Instantiate services"| RC1
    WS_API -->|"5. Instantiate services"| DA1
    
    DA1 --> Bucket1
    DA1 --> PVC1
    RC1 --> PVC1
    RG1 -.->|"Protects"| DA1
    RG1 -.->|"Protects"| RC1
    
    WS_API -.->|"Same process"| NS2
    NS2 -.-> RG2
    NS2 -.-> RC2
    NS2 -.-> DA2
    DA2 -.-> Bucket2
    
    GlobalRC -->|"Federated search"| RC1
    GlobalRC -->|"Federated search"| RC2
    GlobalDA -.->|"Read-only access"| Bucket1
    GlobalDA -.->|"Read-only access"| Bucket2
```

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-269](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68]()

## Workspace Provisioning Flow

The following sequence illustrates the steps when a user requests a workspace through the Workspace API:

```mermaid
sequenceDiagram
    participant User
    participant WS_API as Workspace API<br/>workspace-api
    participant K8s as Kubernetes API
    participant BucketAPI as Bucket API<br/>minio-bucket-api:8080
    participant Harbor as Harbor Registry
    participant PEP as Resource Guard PEP<br/>workspace-api-pep:5576
    participant Templates as ConfigMap<br/>workspace-charts
    
    User->>WS_API: Request workspace creation
    WS_API->>K8s: Create namespace<br/>(e.g., eric-workspace)
    K8s-->>WS_API: Namespace created
    
    WS_API->>BucketAPI: POST /bucket<br/>{name: "eric-workspace"}
    BucketAPI-->>WS_API: Bucket credentials<br/>(access_key_id, secret_access_key)
    
    WS_API->>Harbor: Create project<br/>Configure access
    Harbor-->>WS_API: Registry configured
    
    WS_API->>Templates: Read HelmRelease templates<br/>(template-hr-data-access.yaml<br/>template-hr-resource-catalogue.yaml<br/>template-hr-resource-guard.yaml)
    Templates-->>WS_API: Template definitions
    
    WS_API->>WS_API: Substitute placeholders:<br/>{{ workspace_name }}<br/>{{ access_key_id }}<br/>{{ secret_access_key }}<br/>{{ bucket }}<br/>{{ default_owner }}
    
    WS_API->>K8s: Apply HelmRelease<br/>rm-resource-catalogue
    WS_API->>K8s: Apply HelmRelease<br/>vs (data-access)
    WS_API->>K8s: Apply HelmRelease<br/>resource-guard
    
    K8s-->>WS_API: Services deployed
    
    alt autoProtectionEnabled=True
        WS_API->>PEP: Register workspace resources<br/>POST /resources
        PEP-->>WS_API: Resources protected
    end
    
    WS_API-->>User: Workspace ready<br/>(endpoints, credentials)
```

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:35-49]()

## Template System

Workspace components are provisioned using HelmRelease templates stored in a ConfigMap named `workspace-charts`. These templates contain placeholder variables that are substituted at runtime with user-specific values.

### Template Placeholders

| Placeholder | Description | Example Value |
|-------------|-------------|---------------|
| `{{ workspace_name }}` | User's workspace namespace | `eric-workspace` |
| `{{ bucket }}` | S3 bucket name | `eric-workspace` |
| `{{ access_key_id }}` | S3 access key | Generated by bucket API |
| `{{ secret_access_key }}` | S3 secret key | Generated by bucket API |
| `{{ default_owner }}` | Workspace owner identity | User's ID token subject |

### Data Access Template Structure

The data access template configures a complete EOX View Server instance with workspace-specific storage and catalogue integration:

```mermaid
graph TB
    Template["template-hr-data-access.yaml"]
    
    subgraph "Key Template Sections"
        Ingress["Ingress Configuration<br/>Line 24-31<br/>Host: data-access.{{ workspace_name }}.develop.eoepca.org"]
        Storage["Storage Configuration<br/>Lines 33-44<br/>S3 bucket: {{ bucket }}<br/>Credentials: {{ access_key_id }}, {{ secret_access_key }}"]
        Registrar["Registrar Routes<br/>Lines 93-164<br/>PostgreSQL: resource-catalogue-db<br/>OWS URL: workspace-specific"]
        Harvester["Harvester Configuration<br/>Lines 174-195<br/>Harvest from /home/catalog.json<br/>in user's S3 bucket"]
        Database["PostgreSQL Persistence<br/>Lines 219-228<br/>StorageClass: managed-nfs-storage<br/>Size: 100Gi"]
        Redis["Redis Cache<br/>Lines 230-240<br/>StorageClass: managed-nfs-storage<br/>Size: 1Gi"]
    end
    
    Template --> Ingress
    Template --> Storage
    Template --> Registrar
    Template --> Harvester
    Template --> Database
    Template --> Redis
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:24-31](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:93-164](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:174-195]()

### Resource Catalogue Template Structure

The resource catalogue template provisions a pycsw-based catalogue with federated search capabilities:

```mermaid
graph TB
    Template["template-hr-resource-catalogue.yaml"]
    
    subgraph "Key Template Sections"
        Namespace["Namespace<br/>Line 17<br/>namespace: {{ workspace_name }}"]
        URL["Server URL<br/>Line 33<br/>url: https://resource-catalogue.{{ workspace_name }}.develop.eoepca.org"]
        Federation["Federated Catalogues<br/>Line 34<br/>federatedcatalogues:<br/>https://resource-catalogue.develop.eoepca.org/collections/S2MSI2A"]
        Title["Identification<br/>Line 36<br/>identification_title: Resource Catalogue - {{ workspace_name }}"]
        Storage["Database Storage<br/>Line 19<br/>volume_storage_type: managed-nfs-storage-retain"]
    end
    
    Template --> Namespace
    Template --> URL
    Template --> Federation
    Template --> Title
    Template --> Storage
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:17-67]()

## Workspace Components

Each provisioned workspace contains the following Kubernetes resources:

### Service Deployments

| Component | HelmRelease Name | Purpose |
|-----------|-----------------|---------|
| Resource Catalogue | `rm-resource-catalogue` | pycsw-based CSW/OpenSearch catalogue for workspace metadata |
| Data Access | `vs` | EOX View Server for WMS/WCS/WMTS visualization |
| Resource Guard | `resource-guard` | PEP enforcing ownership policies |

### Storage Resources

| Resource Type | Configuration | Purpose |
|---------------|---------------|---------|
| S3 Bucket | Provisioned via `minio-bucket-api:8080` | Object storage for user data and results |
| PVC (Database) | `managed-nfs-storage`, 100Gi | PostgreSQL data for resource catalogue |
| PVC (Redis) | `managed-nfs-storage`, 1Gi | Redis persistence for harvester queues |

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-240]()

## Storage Isolation

### S3 Bucket Provisioning

S3 buckets are provisioned through the bucket API endpoint, which generates unique credentials per workspace:

```mermaid
graph LR
    WS_API["Workspace API"]
    BucketAPI["minio-bucket-api<br/>:8080/bucket"]
    MinIO["MinIO Server<br/>minio.develop.eoepca.org"]
    
    WS_API -->|"POST /bucket<br/>{name: 'eric-workspace'}"| BucketAPI
    BucketAPI -->|"Create bucket"| MinIO
    BucketAPI -->|"Generate IAM credentials"| MinIO
    BucketAPI -.->|"Return credentials"| WS_API
    
    MinIO -->|"Bucket: eric-workspace"| Bucket1["S3 Bucket<br/>eric-workspace"]
    MinIO -->|"Bucket: alice-workspace"| Bucket2["S3 Bucket<br/>alice-workspace"]
```

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:38-47]()

### Bucket Configuration in Data Access

The data access service is configured with workspace-specific bucket credentials:

| Configuration Key | Template Value |
|-------------------|----------------|
| `global.storage.data.data.type` | `S3` |
| `global.storage.data.data.endpoint_url` | `https://minio.develop.eoepca.org` |
| `global.storage.data.data.access_key_id` | `{{ access_key_id }}` |
| `global.storage.data.data.secret_access_key` | `{{ secret_access_key }}` |
| `global.storage.data.data.bucket` | `{{ bucket }}` |
| `global.storage.data.data.region_name` | `RegionOne` |

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44]()

### Static Catalogue Harvesting

Each workspace data access service includes a harvester configured to continuously harvest a static STAC catalogue from the user's bucket:

```yaml
# Lines 180-195 in template-hr-data-access.yaml
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

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:180-195]()

## Access Control and Ownership

### Resource Guard PEP

Each workspace is protected by a dedicated Resource Guard (Policy Enforcement Point) that enforces ownership policies. The Workspace API configures automatic resource protection:

| Configuration Key | Value |
|-------------------|-------|
| `pepBaseUrl` | `http://workspace-api-pep:5576/resources` |
| `autoProtectionEnabled` | `True` |

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:48-49]()

### Workspace Service Isolation

Services within a workspace are isolated through:

1. **Namespace Isolation**: Each workspace runs in a dedicated Kubernetes namespace
2. **Ingress Routing**: Subdomain-based routing (e.g., `data-access.eric-workspace.develop.eoepca.org`)
3. **PEP Enforcement**: Resource Guard validates user identity against `default_owner` policy
4. **Credential Isolation**: Unique S3 credentials per workspace

```mermaid
graph TB
    User["User: eric"]
    
    subgraph "eric-workspace Namespace"
        RG["Resource Guard PEP<br/>Validates: owner == eric"]
        DA["Data Access<br/>data-access.eric-workspace.develop.eoepca.org"]
        RC["Resource Catalogue<br/>resource-catalogue.eric-workspace.develop.eoepca.org"]
        Bucket["S3 Bucket: eric-workspace<br/>Credentials: eric-specific"]
    end
    
    User -->|"Request with ID token"| RG
    RG -->|"Policy check: PERMIT"| DA
    RG -->|"Policy check: PERMIT"| RC
    DA --> Bucket
    RC -.->|"Metadata about"| Bucket
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:17](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:28]()

## Federated Catalogue Architecture

The workspace resource catalogues participate in a federated search hierarchy:

```mermaid
graph TB
    GlobalRC["Global Resource Catalogue<br/>resource-catalogue.develop.eoepca.org<br/>(rm namespace)"]
    
    subgraph "Workspace Catalogues"
        RC1["eric-workspace Catalogue<br/>resource-catalogue.eric-workspace.develop.eoepca.org"]
        RC2["alice-workspace Catalogue<br/>resource-catalogue.alice-workspace.develop.eoepca.org"]
        RC3["bob-workspace Catalogue<br/>resource-catalogue.bob-workspace.develop.eoepca.org"]
    end
    
    GlobalRC -->|"Federated search"| RC1
    GlobalRC -->|"Federated search"| RC2
    GlobalRC -->|"Federated search"| RC3
    
    RC1 -.->|"federatedcatalogues config<br/>Line 34"| GlobalRC
    RC2 -.->|"federatedcatalogues config"| GlobalRC
    RC3 -.->|"federatedcatalogues config"| GlobalRC
```

Each workspace catalogue is configured with the global catalogue as a federated endpoint, enabling cross-workspace discovery while maintaining isolation.

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:34]()

## Workspace API Configuration

The Workspace API is deployed in the `rm` namespace with the following key configuration:

| Configuration Key | Value | Purpose |
|-------------------|-------|---------|
| `prefixForName` | `develop-user` | Prefix for generated workspace names |
| `workspaceSecretName` | `bucket` | Name of secret storing bucket credentials |
| `namespaceForBucketResource` | `rm` | Namespace where bucket resources are created |
| `s3Endpoint` | `https://minio.develop.eoepca.org` | MinIO S3 endpoint |
| `s3Region` | `RegionOne` | S3 region identifier |
| `harborUrl` | `https://harbor.develop.eoepca.org` | Container registry URL |
| `harborUsername` | `admin` | Harbor admin user |
| `harborPasswordSecretName` | `harbor` | Secret containing Harbor password |
| `umaClientSecretName` | `rm-uma-user-agent` | UMA client for authentication |
| `workspaceChartsConfigMap` | `workspace-charts` | ConfigMap containing templates |
| `bucketEndpointUrl` | `http://minio-bucket-api:8080/bucket` | Bucket provisioning API |

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:35-49]()

## Registrar Backend Configuration

The workspace data access service includes a registrar component with multiple backend routes for different metadata types:

### Registrar Routes

| Route Name | Queue Name | Backend Class | Purpose |
|------------|-----------|---------------|---------|
| `items` | `register_queue` | `registrar_pycsw.backend.ItemBackend` | Register STAC items |
| `collections` | `register_collection_queue` | `registrar_pycsw.backend.CollectionBackend` | Register STAC collections |
| `ades` | `register_ades_queue` | `registrar_pycsw.backend.ADESBackend` | Register ADES services |
| `application` | `register_application_queue` | `registrar_pycsw.backend.CWLBackend` | Register CWL applications |
| `catalogue` | `register_catalogue_queue` | `registrar_pycsw.backend.CatalogueBackend` | Register catalogue metadata |
| `json` | `register_json_queue` | `registrar_pycsw.backend.JSONBackend` | Register JSON metadata |
| `xml` | `register_xml_queue` | `registrar_pycsw.backend.XMLBackend` | Register XML metadata |

All backends connect to the workspace-specific PostgreSQL database at `postgresql://postgres:mypass@resource-catalogue-db/pycsw`.

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:93-164]()

## Resource Scaling

Workspace services are configured with minimal resource requests suitable for single-user environments:

### Data Access Service Resources

| Component | Replicas | CPU Request | Memory Request | Memory Limit |
|-----------|----------|-------------|----------------|--------------|
| Renderer | 1 | 100m | 300Mi | 3Gi |
| Registrar | 1 | 100m | 100Mi | - |
| Harvester | 1 | 100m | 100Mi | - |
| Scheduler | 1 | 100m | 100Mi | - |

### Disabled Components

The following data access components are disabled (0 replicas) in workspace deployments to reduce resource consumption:
- Ingestor
- Preprocessor
- Cache
- Seeder (limited to zoom levels 0-6 when enabled)

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:70-268]()

## Global vs Workspace Services

The EOEPCA deployment maintains both global services (in the `rm` namespace) and per-workspace services:

### Global Services

| Service | Purpose | Access |
|---------|---------|--------|
| Global Data Access | Platform-wide data visualization | Read-only access to all workspace buckets |
| Global Resource Catalogue | Federated catalogue search | Aggregates metadata from all workspaces |
| Workspace API | Workspace provisioning orchestrator | Creates and manages workspaces |
| Registration API | Data registration endpoint | Enqueues metadata for processing |

### Workspace Services

| Service | Purpose | Access |
|---------|---------|--------|
| Workspace Data Access | User-specific data visualization | Full read/write access to workspace bucket |
| Workspace Resource Catalogue | Private metadata catalogue | Federated with global catalogue |
| Resource Guard | Policy enforcement | Validates ownership for all workspace requests |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:1-50](), [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:1-82](), [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50](), [system/clusters/creodias/resource-management/hr-registration-api.yaml:1-37]()