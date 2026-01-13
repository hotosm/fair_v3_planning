# Building Blocks Overview

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [release-notes/release-0.3.md](release-notes/release-0.3.md)
- [system/clusters/creodias/resource-management/hr-data-access.yaml](system/clusters/creodias/resource-management/hr-data-access.yaml)
- [system/clusters/creodias/resource-management/hr-registration-api.yaml](system/clusters/creodias/resource-management/hr-registration-api.yaml)
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/hr-resource-catalogue.yaml)
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml](system/clusters/creodias/resource-management/hr-workspace-api.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml)

</details>



## Purpose and Scope

This document provides a detailed technical explanation of the three primary building blocks that comprise the EOEPCA platform architecture, their constituent components, and their integration patterns. The building blocks are:

1. **User Management** - Authentication, authorization, and identity services
2. **Resource Management** - Data cataloging, access services, and workspace provisioning
3. **Processing & Chaining** - Application deployment, execution, and development environments

For information about the GitOps deployment model and infrastructure provisioning, see [GitOps and Flux CD](#3.2). For details on specific building block implementations, see [User Management and Identity](#4), [Resource Management](#5), and [Processing and Chaining](#6).

---

## Building Block Architecture

The EOEPCA system is organized into three loosely-coupled but highly-integrated building blocks, each deployed to dedicated Kubernetes namespaces and managed through GitOps.

### High-Level Building Block Structure

```mermaid
graph TB
    subgraph "User Management (um namespace)"
        UM[User Management<br/>Building Block]
        LoginSvc["login-service<br/>(Gluu)"]
        IdentitySvc["identity-service<br/>(Keycloak)"]
        PDPEngine["pdp-engine"]
        UserProfile["user-profile"]
        PEP["Policy Enforcement Points<br/>(resource-guard instances)"]
    end
    
    subgraph "Resource Management (rm namespace)"
        RM[Resource Management<br/>Building Block]
        WorkspaceAPI["workspace-api"]
        ResourceCat["resource-catalogue<br/>(pycsw)"]
        DataAccess["data-access<br/>(OGC WMS/WCS/WMTS)"]
        RegistrationAPI["registration-api"]
        Harvester["harvester<br/>(OpenSearch)"]
        BucketOp["bucket-operator<br/>(MinIO)"]
    end
    
    subgraph "Processing & Chaining (proc namespace)"
        PC[Processing & Chaining<br/>Building Block]
        ADES["ades<br/>(OGC API Processes)"]
        AppHub["application-hub<br/>(JupyterHub)"]
        PDE["pde-hub<br/>(Development)"]
        Calrissian["Calrissian<br/>(CWL Executor)"]
    end
    
    subgraph "Storage Layer"
        S3["S3 Storage"]
        PostgreSQL["PostgreSQL"]
        Redis["Redis"]
        NFS["NFS Storage"]
    end
    
    subgraph "External Services"
        ExtData["External Data Sources<br/>(CreoDIAS, Sentinel Hubs)"]
        Users["End Users"]
    end
    
    Users -->|"Authenticate"| LoginSvc
    LoginSvc --> IdentitySvc
    Users -->|"Protected Requests"| PEP
    PEP -->|"Policy Check"| PDPEngine
    PDPEngine --> IdentitySvc
    
    PEP -.->|"Protect"| WorkspaceAPI
    PEP -.->|"Protect"| ResourceCat
    PEP -.->|"Protect"| DataAccess
    PEP -.->|"Protect"| ADES
    
    ExtData -->|"Harvest"| Harvester
    Harvester -->|"Queue"| Redis
    Redis -->|"Register"| RegistrationAPI
    RegistrationAPI -->|"Write Metadata"| ResourceCat
    
    WorkspaceAPI -->|"Provision Bucket"| BucketOp
    BucketOp -->|"Create"| S3
    WorkspaceAPI -->|"Deploy Templates"| ResourceCat
    WorkspaceAPI -->|"Deploy Templates"| DataAccess
    
    ADES -->|"Query Data"| ResourceCat
    ADES -->|"Execute CWL"| Calrissian
    ADES -->|"Stage-in/out"| S3
    
    DataAccess -->|"Read Data"| S3
    DataAccess -->|"Query Metadata"| ResourceCat
    ResourceCat --> PostgreSQL
    DataAccess --> Redis
    
    AppHub -->|"Develop Apps"| ADES
    PDE -->|"Build Packages"| ADES
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml](), [system/clusters/creodias/resource-management/hr-workspace-api.yaml](), [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml](), [release-notes/release-0.3.md:97-264]()

---

## Building Block 1: User Management

The User Management building block provides authentication, authorization, and identity services across the entire platform. It implements a UMA 2.0 (User-Managed Access) authorization pattern with fine-grained policy-based access control.

### User Management Components

| Component | Service Name | Purpose | Key Technologies |
|-----------|-------------|---------|------------------|
| Login Service | `login-service` | Primary authentication provider | Gluu Server (OpenDJ, oxAuth, oxTrust, oxPassport) |
| Identity Service | `identity-service` | Identity provider and token issuer | Keycloak, PostgreSQL |
| Policy Decision Point | `pdp-engine` | Central policy evaluation engine | MongoDB |
| User Profile | `user-profile` | User attribute management | SCIM 2.0 |
| Policy Enforcement Points | `resource-guard` instances | Service-level authorization enforcement | Multiple instances per protected service |

### User Management Component Diagram

```mermaid
graph TB
    subgraph "Login Service"
        Gluu["Gluu Server"]
        OpenDJ["OpenDJ<br/>(LDAP)"]
        oxAuth["oxAuth<br/>(OAuth/OIDC)"]
        oxTrust["oxTrust<br/>(Admin UI)"]
        oxPassport["oxPassport<br/>(Social Login)"]
    end
    
    subgraph "Identity Service"
        Keycloak["Keycloak Server"]
        KeycloakDB["PostgreSQL<br/>(Identity DB)"]
        KeycloakAPI["Keycloak API"]
    end
    
    subgraph "Policy Decision Point"
        PDPEngine["pdp-engine"]
        PDPMongo["MongoDB<br/>(Policy DB)"]
    end
    
    subgraph "Policy Enforcement"
        PEP1["resource-guard<br/>(ADES PEP)"]
        PEP2["resource-guard<br/>(Workspace PEP)"]
        PEP3["resource-guard<br/>(Data Access PEP)"]
    end
    
    UserProfile["user-profile<br/>(SCIM API)"]
    
    Gluu --> OpenDJ
    Gluu --> oxAuth
    Gluu --> oxTrust
    Gluu --> oxPassport
    
    oxAuth -->|"OIDC Flow"| Keycloak
    Keycloak --> KeycloakDB
    Keycloak --> KeycloakAPI
    
    PEP1 -->|"Policy Query"| PDPEngine
    PEP2 -->|"Policy Query"| PDPEngine
    PEP3 -->|"Policy Query"| PDPEngine
    PDPEngine --> PDPMongo
    PDPEngine -->|"Token Validation"| Keycloak
    
    UserProfile -->|"SCIM"| Keycloak
```

**Sources:** [release-notes/release-0.3.md:102-218]()

### Key Integration Points

The User Management building block provides:

- **OIDC Identity Provider**: Keycloak serves as the primary identity provider at `identity-service` endpoint
- **UMA Authorization Server**: Issues UMA tickets and RPT tokens for resource access
- **Policy Management**: Central policy repository in `pdp-engine` accessed by all PEPs
- **Resource Protection**: Each protected service deploys its own `resource-guard` (PEP) instance

For details on the UMA authentication flow, see [UMA Authentication Flow](#4.4). For PEP/PDP implementation details, see [Policy Enforcement (PEP/PDP)](#4.3).

---

## Building Block 2: Resource Management

The Resource Management building block handles data cataloging, visualization, access services, and user workspace provisioning. It manages both global platform resources and per-user isolated workspaces.

### Resource Management Components

| Component | Service Name | Purpose | Key Technologies |
|-----------|-------------|---------|------------------|
| Data Access | `data-access` | OGC-compliant data visualization and access | EOX ViewServer, OGC WMS/WCS/WMTS |
| Resource Catalogue | `resource-catalogue` | Metadata catalog and discovery | pycsw, OGC CSW, OpenSearch |
| Workspace API | `workspace-api` | Multi-tenant workspace orchestration | Kubernetes operator pattern |
| Registration API | `registration-api` | Data and metadata registration endpoint | Redis queue consumer |
| Harvester | `harvester` | External data source ingestion | OpenSearch clients |
| Bucket Operator | `bucket-operator` (via MinIO) | S3 bucket provisioning | MinIO API |

### Resource Management Architecture

```mermaid
graph TB
    subgraph "Global Services (rm namespace)"
        WorkspaceAPI["workspace-api"]
        GlobalRC["resource-catalogue<br/>(pycsw)"]
        GlobalDA["data-access"]
        RegistrationAPI["registration-api"]
        Harvester["harvester"]
        MinioAPI["minio-bucket-api"]
    end
    
    subgraph "Data Access Components"
        Renderer["renderer<br/>(4 replicas)"]
        Registrar["registrar"]
        Client["client"]
        Cache["cache/seeder"]
        DARedis["data-access-redis-master"]
    end
    
    subgraph "Catalogue Components"
        PyCSW["pycsw"]
        RCDB["resource-catalogue-db<br/>(PostgreSQL)"]
    end
    
    subgraph "Storage"
        S3Global["S3 Buckets"]
        NFSStorage["NFS PersistentVolumes"]
    end
    
    subgraph "External Sources"
        CreoDIAS["CreoDIAS OpenSearch"]
        Sentinel["Sentinel Hubs"]
    end
    
    GlobalDA --> Renderer
    GlobalDA --> Registrar
    GlobalDA --> Client
    GlobalDA --> Cache
    GlobalDA --> DARedis
    
    GlobalRC --> PyCSW
    PyCSW --> RCDB
    
    Harvester -->|"Poll"| CreoDIAS
    Harvester -->|"Poll"| Sentinel
    Harvester -->|"Enqueue"| DARedis
    DARedis -->|"Consume"| RegistrationAPI
    RegistrationAPI -->|"Write"| RCDB
    
    Renderer -->|"Read"| S3Global
    Registrar -->|"Register"| RCDB
    
    WorkspaceAPI -->|"Create Bucket"| MinioAPI
    MinioAPI --> S3Global
    WorkspaceAPI -->|"Provision PVCs"| NFSStorage
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:1-1144](), [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:1-82](), [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50](), [system/clusters/creodias/resource-management/hr-registration-api.yaml:1-37]()

### Data Access Service Details

The Data Access service is deployed as `data-access` in the `rm` namespace and provides OGC-compliant data visualization. Configuration is defined in [system/clusters/creodias/resource-management/hr-data-access.yaml:1-1144]().

**Key Configuration:**
- **Renderer**: 4 replicas for parallel processing ([hr-data-access.yaml:865-877]())
- **Ingress**: Exposed at `data-access.develop.eoepca.org` ([hr-data-access.yaml:42-47]())
- **S3 Data Source**: CloudFerro eodata at `http://data.cloudferro.com` ([hr-data-access.yaml:49-57]())
- **Cache Storage**: S3 cache bucket at `https://cf2.cloudferro.com:8080` ([hr-data-access.yaml:58-64]())

**Supported Collections:**
- Sentinel-2 L1C and L2A with multiple band configurations ([hr-data-access.yaml:218-252]())
- Landsat-8 L1TP and L1GT ([hr-data-access.yaml:253-278]())
- Sentinel-1 GRD and SLC ([hr-data-access.yaml:279-286]())
- Sentinel-3 OL_2_LFR ([hr-data-access.yaml:287-290]())

### Resource Catalogue Details

The Resource Catalogue is deployed as `resource-catalogue` using pycsw and provides OGC CSW 3.0/2.0.2 interfaces. Configuration is in [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:1-82]().

**Key Configuration:**
- **Database**: PostgreSQL with 5Gi volume and tuned performance parameters ([hr-resource-catalogue.yaml:19-31]())
- **Server URL**: `https://resource-catalogue.develop.eoepca.org/` ([hr-resource-catalogue.yaml:45]())
- **Transactions**: Enabled with allowed IPs set to "*" ([hr-resource-catalogue.yaml:47-48]())
- **INSPIRE**: Enabled with multi-language support ([hr-resource-catalogue.yaml:72-81]())

### Workspace API Details

The Workspace API orchestrates multi-tenant workspace provisioning. Configuration is in [system/clusters/creodias/resource-management/hr-workspace-api.yaml:1-50]().

**Key Configuration:**
- **Namespace Prefix**: `develop-user` for workspace namespaces ([hr-workspace-api.yaml:35]())
- **S3 Endpoint**: MinIO at `https://minio.develop.eoepca.org` ([hr-workspace-api.yaml:39]())
- **Harbor Registry**: `https://harbor.develop.eoepca.org` ([hr-workspace-api.yaml:41]())
- **Bucket API**: `http://minio-bucket-api:8080/bucket` ([hr-workspace-api.yaml:47]())
- **PEP Integration**: `http://workspace-api-pep:5576/resources` ([hr-workspace-api.yaml:48]())
- **Auto-Protection**: Enabled for automatic resource registration ([hr-workspace-api.yaml:49]())

The Workspace API uses HelmRelease templates to provision per-user instances of services. See [Multi-Tenant Workspaces](#5.5) for details.

### Registration Pipeline

The registration pipeline handles ingestion of metadata from external sources:

```mermaid
sequenceDiagram
    participant Harvester as harvester
    participant Redis as data-access-redis-master
    participant RegAPI as registration-api
    participant Registrar as registrar
    participant PyCSW as resource-catalogue-db
    
    Harvester->>Harvester: Poll OpenSearch endpoints
    Harvester->>Redis: Enqueue items to register_queue
    RegAPI->>Redis: Consume from register_queue
    RegAPI->>Registrar: Route to backend
    
    alt Collection Registration
        Registrar->>PyCSW: Write to CollectionBackend
    else Item Registration
        Registrar->>PyCSW: Write to ItemBackend
    else ADES Registration
        Registrar->>PyCSW: Write to ADESBackend
    else Application Registration
        Registrar->>PyCSW: Write to CWLBackend
    end
    
    PyCSW->>PyCSW: Store ISO 19115 metadata
```

**Registrar Routes** (configured in [hr-data-access.yaml:894-947]()):
- `collections`: STAC Collection route to `CollectionBackend`
- `ades`: JSON route for ADES service registration to `ADESBackend`
- `application`: JSON route for CWL application registration to `CWLBackend`
- `catalogue`: JSON route for catalogue federation to `CatalogueBackend`
- `json`/`xml`: Generic registration routes to `JSONBackend`/`XMLBackend`

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:878-947](), [system/clusters/creodias/resource-management/hr-registration-api.yaml:34-37]()

---

## Building Block 3: Processing & Chaining

The Processing & Chaining building block provides application deployment, workflow execution, and development environments for Earth Observation processing.

### Processing & Chaining Components

| Component | Service Name | Purpose | Key Technologies |
|-----------|-------------|---------|------------------|
| ADES | `ades` | Application deployment and execution service | OGC API Processes, ZOO-Project |
| Application Hub | `application-hub` | Interactive development environment | JupyterHub, OAuth2 |
| PDE Hub | `pde-hub` | Processor development tools | Theia IDE, Jenkins, Docker-in-Docker |
| CWL Executor | Calrissian (invoked by ADES) | Kubernetes-native workflow execution | Calrissian, CWL |

### ADES Integration Architecture

```mermaid
graph TB
    subgraph "ADES Service"
        ADESCore["ades<br/>(ZOO-Project)"]
        WPSAPI["OGC WPS 2.0 API"]
        ProcAPI["OGC API Processes"]
    end
    
    subgraph "Workflow Execution"
        Calrissian["Calrissian<br/>(CWL Executor)"]
        K8sPods["Processing Pods<br/>(Dynamic)"]
    end
    
    subgraph "Stage-in/Stage-out"
        StageIn["stars-t2:0.6.17.0<br/>(Stage-in)"]
        StageOut["stars-t2:0.6.17.0<br/>(Stage-out)"]
    end
    
    subgraph "Data Discovery"
        ResourceCat["resource-catalogue"]
        OpenSearch["OpenSearch API"]
    end
    
    subgraph "Storage"
        S3Input["S3 Input Data<br/>(eodata)"]
        S3Output["S3 Output Data<br/>(Workspace Bucket)"]
    end
    
    User["User"] -->|"Deploy Process"| ADESCore
    User -->|"Execute Job"| ADESCore
    ADESCore --> WPSAPI
    ADESCore --> ProcAPI
    
    ADESCore -->|"Query Inputs"| ResourceCat
    ResourceCat --> OpenSearch
    
    ADESCore -->|"Execute CWL"| Calrissian
    Calrissian -->|"Create"| K8sPods
    
    ADESCore -->|"Invoke"| StageIn
    StageIn -->|"Fetch"| S3Input
    
    K8sPods -->|"Process"| S3Input
    K8sPods -->|"Write"| S3Output
    
    ADESCore -->|"Invoke"| StageOut
    StageOut -->|"Write STAC"| S3Output
```

**Sources:** [release-notes/release-0.3.md:220-249]()

### ADES Capabilities

The ADES provides the following capabilities (from [release-notes/release-0.3.md:30-41]()):

1. **OGC Interfaces**: Implements OGC WPS 2.0 and OGC API Processes
2. **Process Management**:
   - List available processes
   - Deploy process (Docker container with CWL application package)
   - Execute process (create job)
   - Get job status
   - Undeploy process
3. **Data Integration**:
   - Stage-in via OpenSearch catalogue reference
   - Stage-out to S3 bucket with STAC manifests
4. **Workflow Execution**: Calrissian CWL engine with native Kubernetes integration
5. **User Context**: Dedicated user 'context' within ADES service

### Application Development Workflow

```mermaid
graph LR
    subgraph "Development (PDE)"
        Developer["Developer"]
        JupyterLab["JupyterLab"]
        TheiaIDE["Theia IDE"]
        Jenkins["Jenkins CI"]
        DockerBuild["Docker Build"]
    end
    
    subgraph "Testing (Application Hub)"
        AppHub["application-hub<br/>(JupyterHub)"]
        TestEnv["Test Environment"]
    end
    
    subgraph "Deployment (ADES)"
        DeployProc["Deploy Process"]
        CWLPackage["CWL Package"]
        DockerImage["Docker Image"]
    end
    
    subgraph "Execution"
        ExecuteJob["Execute Job"]
        Calrissian["Calrissian"]
        Results["Results to S3"]
    end
    
    Developer --> JupyterLab
    Developer --> TheiaIDE
    JupyterLab --> Jenkins
    TheiaIDE --> Jenkins
    Jenkins --> DockerBuild
    DockerBuild --> DockerImage
    
    DockerImage --> AppHub
    AppHub --> TestEnv
    TestEnv --> CWLPackage
    
    CWLPackage --> DeployProc
    DockerImage --> DeployProc
    DeployProc --> ExecuteJob
    ExecuteJob --> Calrissian
    Calrissian --> Results
```

**Sources:** [release-notes/release-0.3.md:42-49](), [release-notes/release-0.3.md:252-263]()

---

## Building Block Integration Patterns

The three building blocks integrate through well-defined interfaces and shared infrastructure components.

### Cross-Building-Block Integration

```mermaid
graph TB
    subgraph "User Management"
        Auth["Authentication<br/>(login-service)"]
        Authz["Authorization<br/>(pdp-engine)"]
        PEPs["Policy Enforcement<br/>(resource-guard)"]
    end
    
    subgraph "Resource Management"
        WS["Workspace API"]
        Cat["Resource Catalogue"]
        DA["Data Access"]
    end
    
    subgraph "Processing & Chaining"
        ADES["ADES"]
        AppH["Application Hub"]
    end
    
    User["User"]
    
    User -->|"1. Authenticate"| Auth
    Auth -->|"2. Issue ID Token"| User
    User -->|"3. Request (with Token)"| PEPs
    PEPs -->|"4. Validate Policy"| Authz
    PEPs -->|"5. Forward if Authorized"| WS
    PEPs -->|"5. Forward if Authorized"| Cat
    PEPs -->|"5. Forward if Authorized"| DA
    PEPs -->|"5. Forward if Authorized"| ADES
    
    WS -->|"Provision Resources"| Cat
    WS -->|"Provision Resources"| DA
    ADES -->|"Discover Data"| Cat
    DA -->|"Serve Data"| ADES
    AppH -->|"Deploy Applications"| ADES
    
    ADES -.->|"Protected by"| PEPs
    WS -.->|"Protected by"| PEPs
    Cat -.->|"Protected by"| PEPs
    DA -.->|"Protected by"| PEPs
```

### Key Integration Mechanisms

**Authentication Integration:**
- All services validate tokens against `identity-service` (Keycloak)
- UMA flow issues RPT tokens for resource access
- See [UMA Authentication Flow](#4.4) for sequence details

**Authorization Integration:**
- Each protected service deploys a `resource-guard` (PEP) sidecar
- PEPs query `pdp-engine` for policy decisions
- Resources auto-registered with protection policies (when `autoProtectionEnabled: "True"` in [hr-workspace-api.yaml:49]())

**Data Integration:**
- ADES queries `resource-catalogue` via OpenSearch API
- Data Access reads from same S3 buckets that ADES writes to
- Workspace API provisions isolated catalogue and data access instances per user

**Metadata Integration:**
- All components register metadata to `resource-catalogue` via `registration-api`
- Registrar routes metadata to appropriate backends (Item, Collection, ADES, CWL)
- pycsw stores metadata in PostgreSQL using ISO 19115 format

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:44-49](), [system/clusters/creodias/resource-management/hr-data-access.yaml:878-947]()

---

## Workspace Template System

The Resource Management building block implements a template-based system for provisioning per-user workspace instances. This enables multi-tenancy with isolated resources.

### Template Instantiation

```mermaid
graph TB
    subgraph "Global Service (rm namespace)"
        WSAPI["workspace-api"]
        Templates["workspace-charts<br/>ConfigMap"]
    end
    
    subgraph "Template Resources"
        TempDA["template-hr-data-access.yaml"]
        TempRC["template-hr-resource-catalogue.yaml"]
        TempRG["template-hr-resource-guard.yaml"]
    end
    
    subgraph "User Workspace (user-workspace namespace)"
        UserDA["data-access<br/>(user instance)"]
        UserRC["resource-catalogue<br/>(user instance)"]
        UserRG["resource-guard<br/>(user instance)"]
        UserBucket["S3 Bucket<br/>(user-workspace)"]
        UserPVC["PersistentVolumeClaim<br/>(NFS)"]
    end
    
    User["User Request"] -->|"POST /workspaces"| WSAPI
    WSAPI --> Templates
    Templates --> TempDA
    Templates --> TempRC
    Templates --> TempRG
    
    TempDA -->|"Substitute {{ workspace_name }}"| UserDA
    TempDA -->|"Substitute {{ bucket }}"| UserDA
    TempDA -->|"Substitute {{ access_key_id }}"| UserDA
    
    TempRC -->|"Substitute {{ workspace_name }}"| UserRC
    TempRG -->|"Substitute {{ default_owner }}"| UserRG
    
    WSAPI -->|"Create Bucket"| UserBucket
    WSAPI -->|"Provision Storage"| UserPVC
    
    UserDA --> UserBucket
    UserDA --> UserPVC
    UserRC --> UserPVC
```

**Template Variables** (used in [template-hr-data-access.yaml]() and [template-hr-resource-catalogue.yaml]()):
- `{{ workspace_name }}`: User workspace identifier
- `{{ bucket }}`: S3 bucket name
- `{{ access_key_id }}`: S3 access key
- `{{ secret_access_key }}`: S3 secret key
- `{{ default_owner }}`: User identity for policy enforcement

**Template Configuration Examples:**

From [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:28-44]():
```yaml
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
```

From [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:17-33]():
```yaml
global:
  namespace: "{{ workspace_name }}"
pycsw:
  config:
    server:
      url: "https://resource-catalogue.{{ workspace_name }}.develop.eoepca.org"
    metadata:
      identification_title: Resource Catalogue - {{ workspace_name }}
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-269](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68](), [system/clusters/creodias/resource-management/hr-workspace-api.yaml:46]()

---

## Storage Architecture Integration

All three building blocks integrate with a common storage layer:

| Storage Type | Purpose | Used By | Configuration |
|-------------|---------|---------|---------------|
| S3 (CloudFerro) | Input EO data | Data Access (read), ADES (stage-in) | `http://data.cloudferro.com` |
| S3 (MinIO) | User workspaces, outputs | Workspace API, ADES (stage-out), Data Access (cache) | `https://minio.develop.eoepca.org` |
| PostgreSQL | Catalogue metadata | Resource Catalogue, Identity Service | Per-service databases |
| Redis | Registration queues, caching | Data Access, Registration API | Per-service instances |
| NFS | Persistent volumes | Data Access DB, Resource Catalogue DB | `managed-nfs-storage` StorageClass |

**Data Access Storage** ([hr-data-access.yaml:49-64]()):
- Input data from CloudFerro: `http://data.cloudferro.com` (read-only)
- Cache bucket: `https://cf2.cloudferro.com:8080/cache-bucket` (read-write)

**Workspace Storage** ([hr-workspace-api.yaml:39](), [template-hr-data-access.yaml:39-44]()):
- Per-user buckets provisioned via MinIO API
- Workspace-specific NFS PVCs with `managed-nfs-storage` StorageClass ([template-hr-data-access.yaml:220-224]())

**ADES Storage:**
- Stage-in from CloudFerro eodata or STAC catalogs
- Stage-out to workspace S3 buckets
- STAC manifests describe inputs and outputs

For detailed storage architecture, see [Storage and Persistence](#7).

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64](), [system/clusters/creodias/resource-management/hr-workspace-api.yaml:38-39](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:33-44](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-228]()

---

## Deployment and Namespace Organization

The building blocks are deployed to separate Kubernetes namespaces, each managed by GitOps Kustomization resources:

| Building Block | Namespace | GitOps Kustomization | HelmReleases |
|---------------|-----------|---------------------|-------------|
| User Management | `um` | `./user-management` | `login-service`, `identity-service`, `pdp-engine`, `user-profile` |
| Resource Management | `rm` | `./resource-management` | `data-access`, `resource-catalogue`, `workspace-api`, `registration-api` |
| Processing & Chaining | `proc` | `./processing-and-chaining` | `ades`, `application-hub`, `pde-hub` |
| System Infrastructure | various | `./system` | `ingress-nginx`, `cert-manager`, `sealed-secrets` |

Each HelmRelease references charts from the `eoepca` HelmRepository in the `common` namespace. For example:

From [system/clusters/creodias/resource-management/hr-workspace-api.yaml:8-15]():
```yaml
chart:
  spec:
    chart: rm-workspace-api
    version: "1.4.0"
    sourceRef:
      kind: HelmRepository
      name: eoepca
      namespace: common
```

For complete GitOps deployment details, see [GitOps and Flux CD](#3.2).

**Sources:** [system/clusters/creodias/resource-management/hr-workspace-api.yaml:8-15](), [system/clusters/creodias/resource-management/hr-data-access.yaml:8-15](), [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:8-15]()

---

## Summary of Building Block Interactions

The three building blocks work together to provide a complete Earth Observation platform:

1. **User Management** provides identity and authorization services that protect all other services through PEP instances
2. **Resource Management** provisions workspaces, catalogs data, and provides visualization services
3. **Processing & Chaining** executes user applications on data discovered through Resource Management and protected by User Management

All building blocks:
- Share common storage infrastructure (S3, PostgreSQL, Redis, NFS)
- Are deployed via GitOps using Flux CD HelmReleases
- Follow OGC standards for interoperability (CSW, WMS, WCS, WMTS, WPS, API Processes, OpenSearch)
- Support multi-tenancy through workspace isolation
- Integrate through well-defined APIs and message queues

For specific implementation details, see the dedicated pages for each building block: [User Management and Identity](#4), [Resource Management](#5), and [Processing and Chaining](#6).