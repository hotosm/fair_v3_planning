# Workspace API

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



The Workspace API is an orchestrator service that provisions and manages isolated, user-specific workspaces within the EOEPCA platform. Each workspace provides a user with dedicated instances of Data Access services, Resource Catalogue, and S3 storage, all protected by workspace-specific Policy Enforcement Points (PEPs). This enables multi-tenant data isolation while maintaining a consistent service architecture across all users.

For information about the global Data Access and Resource Catalogue services, see [Data Access Services](#5.1) and [Resource Catalogue](#5.2). For details on the multi-tenant workspace architecture and template patterns, see [Multi-Tenant Workspaces](#5.5).

## Architecture Overview

The Workspace API operates as a control plane service deployed in the global `rm` namespace. It orchestrates the creation of user-specific Kubernetes resources by instantiating HelmRelease templates, provisioning S3 buckets, and configuring Harbor registry access.

**Workspace API Architecture**

```mermaid
graph TB
    User["User Request<br/>(authenticated)"]
    
    subgraph "rm Namespace (Global)"
        WorkspaceAPI["workspace-api<br/>Orchestrator Service"]
        Templates["ConfigMap:<br/>workspace-charts"]
        MinioAPI["minio-bucket-api<br/>S3 Provisioning"]
        PEP["workspace-api-pep<br/>Resource Guard"]
    end
    
    subgraph "workspace-name Namespace"
        NS["Namespace:<br/>workspace-name"]
        
        subgraph "Workspace Services"
            WSDA["HelmRelease: vs<br/>(Data Access)"]
            WSRC["HelmRelease:<br/>rm-resource-catalogue"]
            WSGuard["HelmRelease:<br/>resource-guard"]
        end
        
        subgraph "Storage Resources"
            PVC["PVC:<br/>managed-nfs-storage"]
            Bucket["S3 Bucket:<br/>workspace-name"]
        end
        
        WSDA --> PVC
        WSDA --> Bucket
        WSRC --> PVC
        WSGuard --> WSDA
        WSGuard --> WSRC
    end
    
    subgraph "External Services"
        Harbor["Harbor Registry"]
        Minio["MinIO S3"]
        Identity["Identity Service"]
    end
    
    User --> PEP
    PEP --> WorkspaceAPI
    WorkspaceAPI --> Templates
    WorkspaceAPI --> MinioAPI
    WorkspaceAPI --> Harbor
    WorkspaceAPI --> NS
    
    Templates --> WSDA
    Templates --> WSRC
    Templates --> WSGuard
    
    MinioAPI --> Minio
    Minio --> Bucket
    
    WSGuard --> Identity
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-269]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68]()

## Deployment Configuration

The Workspace API is deployed via a HelmRelease in the `rm` namespace using the `rm-workspace-api` Helm chart.

| Configuration Parameter | Value | Purpose |
|------------------------|-------|---------|
| `fullnameOverride` | `workspace-api` | Service name in cluster |
| `prefixForName` | `develop-user` | Prefix for generated workspace names |
| `workspaceSecretName` | `bucket` | Name for S3 credentials secret |
| `namespaceForBucketResource` | `rm` | Namespace where bucket resources are created |
| `s3Endpoint` | `https://minio.develop.eoepca.org` | MinIO S3 endpoint URL |
| `s3Region` | `RegionOne` | S3 region identifier |
| `harborUrl` | `https://harbor.develop.eoepca.org` | Harbor registry URL |
| `harborUsername` | `admin` | Harbor admin username |
| `harborPasswordSecretName` | `harbor` | Secret containing Harbor password |
| `umaClientSecretName` | `rm-uma-user-agent` | UMA client credentials for PEP integration |
| `workspaceChartsConfigMap` | `workspace-charts` | ConfigMap containing HelmRelease templates |
| `bucketEndpointUrl` | `http://minio-bucket-api:8080/bucket` | Internal bucket provisioning API |
| `pepBaseUrl` | `http://workspace-api-pep:5576/resources` | PEP API for auto-protection |
| `autoProtectionEnabled` | `True` | Automatically register resources with PEP |

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:16-50]()

## Workspace Provisioning Flow

The Workspace API follows a multi-step provisioning sequence when creating a new workspace for a user.

**Workspace Creation Sequence**

```mermaid
sequenceDiagram
    participant User
    participant PEP as workspace-api-pep
    participant API as workspace-api
    participant K8s as Kubernetes API
    participant BucketAPI as minio-bucket-api
    participant Harbor as Harbor Registry
    participant Flux as Flux Controllers
    
    User->>PEP: POST /workspace<br/>(with ID Token)
    PEP->>PEP: Validate UMA Token
    PEP->>API: Forward Request
    
    API->>API: Generate Workspace Name<br/>(prefixForName + username)
    
    API->>K8s: Create Namespace
    Note over K8s: namespace: develop-user-eric
    K8s-->>API: Namespace Created
    
    API->>BucketAPI: POST /bucket<br/>{name: workspace-name}
    BucketAPI->>BucketAPI: Create S3 Bucket
    BucketAPI->>BucketAPI: Generate Access Keys
    BucketAPI-->>API: {bucket, access_key, secret_key}
    
    API->>K8s: Create Secret<br/>(bucket credentials)
    Note over K8s: secret: bucket<br/>in workspace namespace
    
    API->>Harbor: Create Robot Account<br/>(workspace-specific)
    Harbor-->>API: Robot Credentials
    
    API->>K8s: Create Secret<br/>(harbor credentials)
    
    API->>API: Render Templates<br/>(substitute placeholders)
    Note over API: {{ workspace_name }}<br/>{{ bucket }}<br/>{{ access_key_id }}<br/>{{ secret_access_key }}
    
    API->>K8s: Create HelmRelease<br/>(vs - Data Access)
    API->>K8s: Create HelmRelease<br/>(rm-resource-catalogue)
    API->>K8s: Create HelmRelease<br/>(resource-guard)
    
    Flux->>K8s: Reconcile HelmReleases
    Note over Flux: Deploy actual services
    
    K8s-->>API: All Resources Created
    API-->>PEP: Workspace Created
    PEP-->>User: 200 OK<br/>{workspace_details}
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:34-50]()

## Template System

The Workspace API uses a template-based approach to generate workspace-specific HelmRelease manifests. Templates are stored in a ConfigMap (`workspace-charts`) and contain placeholders that are substituted during provisioning.

### Data Access Template

The Data Access template (`template-hr-data-access.yaml`) defines a complete Data Access service instance with workspace-specific configuration.

**Key Template Placeholders:**

| Placeholder | Example Value | Usage |
|------------|---------------|-------|
| `{{ workspace_name }}` | `develop-user-eric` | Namespace, DNS hostnames |
| `{{ bucket }}` | `develop-user-eric` | S3 bucket name |
| `{{ access_key_id }}` | `AKIAIOSFODNN7EXAMPLE` | S3 credentials |
| `{{ secret_access_key }}` | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` | S3 credentials |

**Template Structure:**

```yaml
# Data Access Service Configuration
global:
  ingress:
    tls:
      - hosts:
          - data-access.{{ workspace_name }}.develop.eoepca.org
  
  storage:
    data:
      data:
        type: "S3"
        endpoint_url: https://minio.develop.eoepca.org
        access_key_id: {{ access_key_id }}
        secret_access_key: {{ secret_access_key }}
        bucket: {{ bucket }}
  
  metadata:
    title: Data Access Service - {{ workspace_name }}
```

The template configures a reduced-capacity Data Access instance with:
- Single renderer replica (vs. 4 in global service)
- Workspace-specific S3 storage backend
- PostgreSQL persistence on NFS (`managed-nfs-storage`)
- Redis queue for registration events
- Harvester configured to read from user's bucket catalog

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-269]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:27-44]()

### Resource Catalogue Template

The Resource Catalogue template provisions a pycsw instance for workspace metadata management.

**Key Configuration:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `namespace` | `{{ workspace_name }}` | Deployment namespace |
| `url` | `https://resource-catalogue.{{ workspace_name }}.develop.eoepca.org` | CSW endpoint |
| `federatedcatalogues` | `https://resource-catalogue.develop.eoepca.org/collections/S2MSI2A` | Federated search to global catalogue |
| `identification_title` | `Resource Catalogue - {{ workspace_name }}` | Service title |
| `volume_storage_type` | `managed-nfs-storage-retain` | Persistent storage with retention policy |

The template enables federated catalogue search, allowing workspace catalogues to query both local and global collections while maintaining metadata isolation.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:17-34]()

## Registrar Configuration

Each workspace Data Access service includes a registrar component that processes metadata registration events from a Redis queue.

**Registrar Routes:**

The workspace registrar defines multiple routes for different data types:

| Route | Queue | Backend | Purpose |
|-------|-------|---------|---------|
| `items` | `register_queue` | `ItemBackend` | STAC Item registration to pycsw |
| `collections` | `register_collection_queue` | `CollectionBackend` | STAC Collection registration |
| `ades` | `register_ades_queue` | `ADESBackend` | ADES service registration |
| `application` | `register_application_queue` | `CWLBackend` | CWL application registration |
| `catalogue` | `register_catalogue_queue` | `CatalogueBackend` | Catalogue service registration |

All backends connect to the workspace-specific PostgreSQL database at `postgresql://postgres:mypass@resource-catalogue-db/pycsw`.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:84-164]()

## Harvester Configuration

The workspace harvester is configured to process STAC catalogs stored in the user's S3 bucket.

**S3 Bucket Harvester:**

```yaml
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
      region: RegionOne
```

This configuration enables the harvester to automatically discover and register STAC items from a static catalog (`/home/catalog.json`) in the user's bucket, integrating processing outputs into the workspace catalogue.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:166-195]()

## Storage Integration

Workspaces integrate with two storage backends for different persistence requirements.

### S3 Bucket Provisioning

The Workspace API interacts with the `minio-bucket-api` service to provision dedicated S3 buckets for each workspace.

**Bucket API Endpoint:**

- **URL:** `http://minio-bucket-api:8080/bucket`
- **Namespace:** `rm`
- **Function:** Creates buckets on MinIO and generates IAM access credentials

**Bucket Naming:**

Buckets follow the pattern: `{prefixForName}-{username}`, resulting in names like `develop-user-eric`.

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:38-47]()

### NFS Persistent Volumes

Workspace services use NFS-backed PersistentVolumeClaims for database and cache persistence.

**Storage Configuration:**

| Service | PVC Usage | Storage Class | Typical Size |
|---------|-----------|---------------|--------------|
| PostgreSQL Database | Metadata storage | `managed-nfs-storage` | 100Gi |
| Redis | Queue persistence | `managed-nfs-storage` | 1Gi |

The `managed-nfs-storage` storage class is defined globally and provides NFS-backed volumes suitable for multi-pod access patterns.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-240]()

## Resource Protection

The Workspace API integrates with the Policy Enforcement infrastructure to automatically protect workspace resources.

### Auto-Protection Flow

**Protection Registration**

```mermaid
graph LR
    API["workspace-api"]
    PEP["workspace-api-pep"]
    Resource["Workspace Service<br/>(Data Access / Catalogue)"]
    PDP["pdp-engine"]
    Identity["identity-service"]
    
    API -->|"1. Register Resource"| PEP
    PEP -->|"2. Create Protection Policy"| PDP
    PDP -->|"3. Store Policy"| Identity
    Resource -->|"4. Enforce on Access"| PEP
    
    Note1["Resource URI Pattern:<br/>/data-access/{workspace_name}<br/>/catalogue/{workspace_name}"]
    Note2["Default Owner:<br/>User who created workspace"]
    Note3["Policy Type:<br/>Ownership-based access"]
```

When `autoProtectionEnabled` is `True`, the Workspace API automatically registers each workspace service with the PEP at `http://workspace-api-pep:5576/resources`, creating ownership policies that restrict access to the workspace owner.

**UMA Client Configuration:**

- **Secret:** `rm-uma-user-agent` in `rm` namespace
- **Purpose:** Provides UMA credentials for PEP registration
- **Scope:** Allows Workspace API to register resources on behalf of users

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:44-49]()

## Harbor Registry Integration

Each workspace receives dedicated registry access through Harbor robot accounts.

**Harbor Configuration:**

| Parameter | Value |
|-----------|-------|
| URL | `https://harbor.develop.eoepca.org` |
| Admin Username | `admin` |
| Password Secret | `harbor` (in `rm` namespace) |

The Workspace API creates a robot account per workspace with permissions to:
- Pull public images
- Push/pull workspace-specific repositories

This enables users to:
1. Store application container images in private repositories
2. Deploy applications from private registries to ADES
3. Maintain separation between user workspaces

**Sources:**
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:41-43]()

## Service Endpoints

Once provisioned, workspace services are accessible via dedicated subdomains following consistent naming patterns.

**Workspace Endpoint Patterns:**

| Service | Endpoint Pattern | Example |
|---------|------------------|---------|
| Data Access - OGC Services | `https://data-access.{workspace_name}.develop.eoepca.org/ows` | `https://data-access.develop-user-eric.develop.eoepca.org/ows` |
| Data Access - Client UI | `https://data-access.{workspace_name}.develop.eoepca.org/` | `https://data-access.develop-user-eric.develop.eoepca.org/` |
| Resource Catalogue - CSW | `https://resource-catalogue.{workspace_name}.develop.eoepca.org` | `https://resource-catalogue.develop-user-eric.develop.eoepca.org` |
| MinIO S3 - Bucket Access | `https://minio.develop.eoepca.org/{workspace_name}` | `https://minio.develop.eoepca.org/develop-user-eric` |

All endpoints are protected by workspace-specific Resource Guards that enforce ownership policies via UMA authentication.

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:28-28]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:33-33]()

## Resource Scaling

Workspace services are deployed with reduced resource allocations compared to global services to support multi-tenancy at scale.

**Resource Allocation Comparison:**

| Component | Global Service | Workspace Service |
|-----------|----------------|-------------------|
| Renderer Replicas | 4 | 1 |
| Renderer CPU Request | 500m | 100m |
| Renderer Memory Request | 1Gi | 300Mi |
| Renderer Memory Limit | 3Gi | 3Gi |
| Registrar CPU Request | 100m | 100m |
| Registrar Memory Request | 100Mi | 100Mi |
| Harvester CPU Request | 100m | 100m |
| Scheduler CPU Request | 100m | 100m |
| Ingestor Replicas | 1 | 0 (disabled) |
| Preprocessor Replicas | 1 | 0 (disabled) |
| Cache Replicas | 1 | 0 (disabled) |
| Seeder Replicas | 1 | 0 (disabled) |

Workspace instances disable resource-intensive components (ingestor, preprocessor, cache, seeder) as users typically access data through the global services or direct S3 access.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:865-876]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:70-82]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:251-268]()

## Related Services

The Workspace API coordinates with several other Resource Management components:

- **[Data Access Services](#5.1)** - Global data visualization and OGC service endpoints
- **[Resource Catalogue](#5.2)** - Global metadata catalogue with federated search capabilities  
- **[Registration API](#5.4)** - Registration service for manual data registration
- **[Multi-Tenant Workspaces](#5.5)** - Detailed workspace architecture and isolation patterns

The Workspace API serves as the provisioning control plane for the multi-tenant workspace architecture described in section 5.5.