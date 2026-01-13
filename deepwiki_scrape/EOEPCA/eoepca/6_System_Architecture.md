# System Architecture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [README.md](README.md)
- [minikube/README.md](minikube/README.md)
- [release-notes/release-0.3.md](release-notes/release-0.3.md)
- [system/clusters/README.md](system/clusters/README.md)

</details>



## Purpose and Scope

This document provides a high-level architectural overview of the EOEPCA system, describing its major components, their interactions, and the organizational principles that govern the system design. The architecture is presented from multiple perspectives: logical building blocks, deployment structure, and runtime interactions.

For detailed deployment procedures, see [Deployment Guide](#2.1). For specific building block implementations, refer to [Building Blocks Overview](#3.1), [User Management and Identity](#4), [Resource Management](#5), and [Processing and Chaining](#6). Infrastructure provisioning details are covered in [Infrastructure Provisioning](#2.2) and [Terraform Infrastructure as Code](#8.2).

## Architectural Principles

The EOEPCA system is designed according to the following core principles:

### Standards-Based Architecture

The platform implements Open Geospatial Consortium (OGC) standards for interoperability:
- **OGC API Processes** - Application execution interfaces
- **OGC CSW 3.0** - Catalogue services
- **OGC WMS/WMTS/WCS** - Data visualization and access
- **OpenSearch** - Data discovery with EO, Geo, and Time extensions
- **STAC** - Metadata for stage-in/stage-out workflows

Sources: [README.md:163-170]()

### GitOps-Driven Deployment

All system components are deployed and managed using Flux CD, implementing a GitOps approach where the Git repository serves as the single source of truth for cluster state. This ensures version-controlled, auditable, and reproducible deployments.

Sources: [system/clusters/README.md:1-95](), [README.md:92-93]()

### Multi-Tenant Isolation

User workspaces are provisioned as isolated Kubernetes namespaces with dedicated S3 buckets, catalogues, and policy enforcement points, enabling secure multi-tenancy while maintaining resource sharing capabilities.

Sources: Diagram 4 from high-level overview

### Microservices on Kubernetes

The system is composed of loosely-coupled microservices deployed as containers orchestrated by Kubernetes, supporting both cloud (OpenStack via RKE) and local (Minikube/k3s) deployments.

Sources: [README.md:70](), [minikube/README.md:1-52]()

## System Decomposition

### Building Block Domains

The EOEPCA system is organized into three primary functional domains, each deployed in a dedicated Kubernetes namespace:

```mermaid
graph TB
    subgraph "um namespace"
        UM["User Management<br/>Building Block"]
        LoginSvc["login-service"]
        IdentitySvc["identity-service"]
        PDPEngine["pdp-engine"]
        UserProfile["user-profile"]
        UM --> LoginSvc
        UM --> IdentitySvc
        UM --> PDPEngine
        UM --> UserProfile
    end
    
    subgraph "rm namespace"
        RM["Resource Management<br/>Building Block"]
        WorkspaceAPI["workspace-api"]
        ResourceCat["resource-catalogue"]
        DataAccess["data-access"]
        Registrar["registrar"]
        Harvester["harvester"]
        BucketOp["bucket-operator"]
        RM --> WorkspaceAPI
        RM --> ResourceCat
        RM --> DataAccess
        RM --> Registrar
        RM --> Harvester
        RM --> BucketOp
    end
    
    subgraph "proc namespace"
        PC["Processing & Chaining<br/>Building Block"]
        ADES["ades"]
        AppHub["application-hub"]
        PDEHub["pde-hub"]
        PC --> ADES
        PC --> AppHub
        PC --> PDEHub
    end
    
    subgraph "system namespace"
        SYS["System Infrastructure"]
        IngressNginx["ingress-nginx"]
        CertManager["cert-manager"]
        SealedSecrets["sealed-secrets-controller"]
        FluxSystem["flux-system"]
        SYS --> IngressNginx
        SYS --> CertManager
        SYS --> SealedSecrets
        SYS --> FluxSystem
    end
    
    SYS -.->|"provides"| UM
    SYS -.->|"provides"| RM
    SYS -.->|"provides"| PC
    UM -.->|"protects"| RM
    UM -.->|"protects"| PC
    RM -.->|"data for"| PC
```

**Domain Architecture by Namespace**

| Namespace | Domain | Primary Services | Configuration Path |
|-----------|--------|-----------------|-------------------|
| `um` | User Management | `login-service`, `identity-service`, `pdp-engine`, `user-profile` | `system/clusters/{target}/user-management/` |
| `rm` | Resource Management | `workspace-api`, `resource-catalogue`, `data-access`, `registrar`, `harvester` | `system/clusters/{target}/resource-management/` |
| `proc` | Processing & Chaining | `ades`, `application-hub`, `pde-hub` | `system/clusters/{target}/processing-and-chaining/` |
| `flux-system` | GitOps Control Plane | `source-controller`, `kustomize-controller`, `helm-controller` | `system/clusters/{target}/system/flux-system/` |
| `ingress-nginx` | External Access | `ingress-nginx-controller` | `system/clusters/{target}/system/ingress-nginx/` |
| `cert-manager` | TLS Certificates | `cert-manager`, `cert-manager-webhook` | `system/clusters/{target}/system/cert-manager/` |

Sources: [README.md:128-160](), [release-notes/release-0.3.md:97-319](), Diagram 1 and Diagram 5 from high-level overview

### Directory Structure and GitOps Mapping

The repository structure directly maps to the deployment architecture:

```mermaid
graph LR
    subgraph "Git Repository Structure"
        Root["eoepca/"]
        System["system/"]
        Clusters["clusters/"]
        Target["develop/<br/>minikube/<br/>creodias/"]
        SystemDir["system/"]
        UM["user-management/"]
        RM["resource-management/"]
        Proc["processing-and-chaining/"]
        
        Root --> System
        System --> Clusters
        Clusters --> Target
        Target --> SystemDir
        Target --> UM
        Target --> RM
        Target --> Proc
    end
    
    subgraph "Flux Resources"
        GitRepo["GitRepository<br/>eoepca-system"]
        Kustomization["Kustomization<br/>eoepca-system"]
        HelmReleases["HelmRelease<br/>resources"]
    end
    
    subgraph "Kubernetes Cluster"
        Namespaces["Namespaces<br/>um, rm, proc"]
        Deployments["Deployments<br/>Pods, Services"]
    end
    
    SystemDir --> GitRepo
    UM --> HelmReleases
    RM --> HelmReleases
    Proc --> HelmReleases
    GitRepo --> Kustomization
    Kustomization --> HelmReleases
    HelmReleases --> Namespaces
    Namespaces --> Deployments
```

**Key Directory Paths:**

- `system/clusters/{target}/` - Deployment configuration for a specific cluster
- `system/clusters/{target}/system/` - Flux bootstrap and system infrastructure
- `system/clusters/{target}/user-management/` - HelmRelease manifests for Identity/Access services
- `system/clusters/{target}/resource-management/` - HelmRelease manifests for Catalogue/Data services
- `system/clusters/{target}/processing-and-chaining/` - HelmRelease manifests for ADES/Application Hub

Sources: [system/clusters/README.md:45-50](), [system/clusters/README.md:81-85]()

## Component Interaction Architecture

### Authentication and Authorization Flow

The system implements User-Managed Access (UMA) 2.0 for fine-grained authorization:

```mermaid
sequenceDiagram
    participant User
    participant Ingress as "ingress-nginx"
    participant PEP as "resource-guard<br/>(PEP)"
    participant PDP as "pdp-engine"
    participant Keycloak as "identity-service<br/>(Keycloak)"
    participant Backend as "workspace-api /<br/>ades / etc"
    
    User->>Ingress: "GET /workspaces"
    Ingress->>PEP: "Forward request"
    
    alt "No Bearer Token"
        PEP->>Keycloak: "POST /uma-configuration"
        Keycloak-->>PEP: "UMA ticket"
        PEP-->>User: "401 + WWW-Authenticate:<br/>ticket={ticket}"
        User->>Keycloak: "POST /token<br/>(username, password, ticket)"
        Keycloak-->>User: "RPT (access_token)"
        User->>Ingress: "GET /workspaces<br/>Authorization: Bearer {RPT}"
        Ingress->>PEP: "Forward with token"
    end
    
    PEP->>PDP: "POST /authz<br/>{token, resource, action}"
    PDP->>Keycloak: "Introspect token"
    Keycloak-->>PDP: "Token claims"
    PDP->>PDP: "Evaluate policies"
    PDP-->>PEP: "Decision: Permit/Deny"
    
    alt "Permit"
        PEP->>Backend: "Forward request<br/>+ user headers"
        Backend-->>PEP: "Response"
        PEP-->>User: "200 OK"
    else "Deny"
        PEP-->>User: "403 Forbidden"
    end
```

**Key Components:**

- `resource-guard` - PEP implementation protecting HTTP endpoints
- `pdp-engine` - Policy Decision Point evaluating access policies
- `identity-service` - Keycloak-based IdP providing UMA and OIDC
- `login-service` - Legacy Gluu-based authentication (deprecated in v1.4)

Sources: [README.md:150-159](), Diagram 2 from high-level overview, [release-notes/release-0.3.md:17-29]()

### Data Access and Processing Pipeline

```mermaid
graph TB
    subgraph "External Data Sources"
        S2["Sentinel-2<br/>CreoDIAS eodata"]
        STAC["STAC Catalogs"]
    end
    
    subgraph "Data Ingestion"
        Harvester["harvester<br/>OpenSearch polling"]
        Queue["Redis<br/>register_queue"]
        Registrar["registrar<br/>metadata ingestion"]
    end
    
    subgraph "Cataloging"
        PyCSW["resource-catalogue<br/>pycsw"]
        PGSQL[("PostgreSQL<br/>csw.records")]
    end
    
    subgraph "Access Services"
        VS["data-access<br/>ViewServer"]
        Renderer["renderer<br/>4 replicas"]
        Cache["cache-seeder"]
    end
    
    subgraph "Processing"
        User["User"]
        ADES["ades<br/>ZOO-Project"]
        Calrissian["Calrissian<br/>CWL executor"]
        K8sJobs["Kubernetes Jobs<br/>processing pods"]
    end
    
    subgraph "Storage"
        S3In["S3 Input<br/>eodata bucket"]
        S3Out["S3 Output<br/>workspace bucket"]
    end
    
    S2 --> Harvester
    STAC --> Harvester
    Harvester -->|"LPUSH"| Queue
    Queue -->|"RPOP"| Registrar
    Registrar -->|"INSERT"| PGSQL
    PGSQL --> PyCSW
    
    PyCSW -->|"OpenSearch"| User
    PyCSW -->|"CSW GetRecords"| VS
    PyCSW -->|"CSW GetRecords"| ADES
    
    VS --> Renderer
    Renderer -->|"stage-in"| S3In
    Renderer --> Cache
    
    User -->|"POST /processes/{id}/execution"| ADES
    ADES -->|"Query inputs"| PyCSW
    ADES -->|"Submit CWL"| Calrissian
    Calrissian -->|"Create"| K8sJobs
    K8sJobs -->|"Read data"| S3In
    K8sJobs -->|"Write results"| S3Out
    ADES -->|"Generate STAC"| S3Out
```

**Key Integration Points:**

- `harvester` pulls metadata from OpenSearch endpoints and enqueues to Redis
- `registrar` consumes queue and writes ISO 19115 records to PostgreSQL
- `pycsw` provides OGC CSW and OpenSearch interfaces over PostgreSQL
- `ades` orchestrates CWL workflows using Calrissian on Kubernetes
- Stage-in/out uses S3 for both input data (`eodata`) and output workspaces

Sources: [README.md:132-149](), [release-notes/release-0.3.md:31-80](), Diagram 3 from high-level overview

## Workspace Multi-Tenancy Architecture

Each user workspace is provisioned as an isolated environment with dedicated resources:

```mermaid
graph TB
    subgraph "Global Services (rm namespace)"
        GlobalWS["workspace-api<br/>HelmRelease"]
        MinioAPI["bucket-operator<br/>MinIO S3 API"]
        TemplateRepo["HelmRepository<br/>eoepca-helm-charts"]
    end
    
    subgraph "User: eric"
        NS1["Namespace<br/>ws-eric"]
        Guard1["resource-guard<br/>owner=eric"]
        RC1["resource-catalogue<br/>eric's metadata"]
        DA1["data-access<br/>eric's WMS"]
        Bucket1["S3 Bucket<br/>ws-eric"]
        PVC1["PersistentVolumeClaim<br/>managed-nfs"]
    end
    
    subgraph "User: alice"
        NS2["Namespace<br/>ws-alice"]
        Guard2["resource-guard<br/>owner=alice"]
        RC2["resource-catalogue<br/>alice's metadata"]
        DA2["data-access<br/>alice's WMS"]
        Bucket2["S3 Bucket<br/>ws-alice"]
        PVC2["PersistentVolumeClaim<br/>managed-nfs"]
    end
    
    subgraph "Template System"
        Templates["HelmRelease Templates<br/>- template-hr-data-access<br/>- template-hr-resource-catalogue<br/>- template-hr-resource-guard"]
    end
    
    GlobalWS -->|"1. Create namespace"| NS1
    GlobalWS -->|"2. Provision bucket"| MinioAPI
    MinioAPI --> Bucket1
    GlobalWS -->|"3. Instantiate from"| Templates
    Templates -->|"values:<br/>workspace_name=ws-eric<br/>default_owner=eric"| Guard1
    Templates --> RC1
    Templates --> DA1
    
    Guard1 --> Bucket1
    DA1 --> Bucket1
    RC1 --> PVC1
    
    GlobalWS -.->|"alice requests"| NS2
    NS2 --> Guard2
    NS2 --> RC2
    NS2 --> DA2
    Guard2 --> Bucket2
```

**Workspace Provisioning Sequence:**

1. User requests workspace via `workspace-api`
2. API creates Kubernetes namespace `ws-{username}`
3. `bucket-operator` provisions S3 bucket `ws-{username}`
4. API instantiates HelmRelease templates with user-specific values
5. Templates deploy `resource-guard`, `resource-catalogue`, `data-access` in namespace
6. PEP (`resource-guard`) enforces ownership policies based on `default_owner`

Sources: [README.md:148](), Diagram 4 from high-level overview

## Deployment Architecture

### Flux CD GitOps Workflow

```mermaid
graph TB
    subgraph "Git Repository"
        Repo["github.com/EOEPCA/eoepca<br/>branch: develop"]
        SystemPath["system/clusters/develop/"]
    end
    
    subgraph "Flux Controllers (flux-system namespace)"
        SourceCtrl["source-controller<br/>polls Git every 1m"]
        KustomizeCtrl["kustomize-controller<br/>reconciles Kustomizations"]
        HelmCtrl["helm-controller<br/>reconciles HelmReleases"]
        NotifyCtrl["notification-controller<br/>alerts on changes"]
    end
    
    subgraph "Flux Resources"
        GitRepoRes["GitRepository<br/>eoepca-system"]
        KustomizationRes["Kustomization<br/>eoepca-system"]
        HRs["HelmRelease<br/>um-login-service<br/>rm-workspace-api<br/>proc-ades<br/>..."]
    end
    
    subgraph "Kubernetes Cluster"
        Pods["Deployments<br/>Pods<br/>Services<br/>Ingresses"]
    end
    
    Repo -->|"Git pull"| SourceCtrl
    SourceCtrl --> GitRepoRes
    GitRepoRes --> KustomizeCtrl
    KustomizeCtrl --> KustomizationRes
    KustomizationRes --> HelmCtrl
    HelmCtrl --> HRs
    HRs -->|"helm install/upgrade"| Pods
    
    SystemPath -.->|"contains"| GitRepoRes
    SystemPath -.->|"contains"| KustomizationRes
    SystemPath -.->|"contains"| HRs
```

**GitOps Deployment Flow:**

1. Developer commits changes to `system/clusters/{target}/` in Git
2. `source-controller` detects changes via polling (interval: 1m)
3. `kustomize-controller` reconciles `Kustomization` resources
4. `helm-controller` reconciles `HelmRelease` resources
5. Helm charts are installed/upgraded in target namespaces
6. Kubernetes applies Deployment/StatefulSet/Job resources

**Flux Bootstrap Command:**

The deployment is initialized using `deployCluster.sh` which executes:

```bash
flux bootstrap github \
  --owner=EOEPCA \
  --repository=eoepca \
  --branch=develop \
  --path=system/clusters/develop/system \
  --personal=false
```

Sources: [system/clusters/README.md:51-77](), Diagram 5 from high-level overview

### Infrastructure Layers

```mermaid
graph TB
    subgraph "Physical Infrastructure"
        CREODIAS["OpenStack<br/>CREODIAS Cloud"]
        TF["Terraform<br/>IaC provisioning"]
    end
    
    subgraph "Kubernetes Cluster"
        RKE["Rancher Kubernetes Engine<br/>or Minikube/k3s"]
        Masters["Control Plane Nodes"]
        Workers["Worker Nodes"]
    end
    
    subgraph "Platform Services"
        Ingress["ingress-nginx<br/>External routing"]
        CertMgr["cert-manager<br/>TLS automation"]
        Sealed["sealed-secrets-controller<br/>Encrypted secrets"]
        StorageClass["Storage Classes<br/>managed-nfs, ebs"]
    end
    
    subgraph "EOEPCA Building Blocks"
        UM["User Management<br/>um namespace"]
        RM["Resource Management<br/>rm namespace"]
        Proc["Processing & Chaining<br/>proc namespace"]
        Workspaces["User Workspaces<br/>ws-* namespaces"]
    end
    
    TF -->|"provision VMs"| CREODIAS
    CREODIAS --> RKE
    RKE --> Masters
    RKE --> Workers
    
    Masters --> Ingress
    Masters --> CertMgr
    Masters --> Sealed
    Masters --> StorageClass
    
    Ingress --> UM
    Ingress --> RM
    Ingress --> Proc
    Ingress --> Workspaces
    
    StorageClass --> RM
    StorageClass --> Proc
    StorageClass --> Workspaces
```

**Deployment Targets:**

| Target | Infrastructure | Kubernetes | Configuration Path |
|--------|---------------|------------|-------------------|
| Cloud (Production) | OpenStack CREODIAS | RKE 1.24+ | `system/clusters/creodias/` |
| Development | OpenStack CREODIAS | RKE 1.24+ | `system/clusters/develop/` |
| Local Testing | Local machine | Minikube 1.25+ | `system/clusters/minikube/` |
| Local Testing (Alternative) | Local machine | k3s 1.24+ | `system/clusters/minikube/` |

**DNS and Ingress Pattern:**

Services are exposed using `nip.io` dynamic DNS:
- Pattern: `{service}.{ip-with-dashes}.nip.io`
- Example: `workspace-api.192-168-49-2.nip.io` (Minikube)
- Example: `workspace-api.185-52-193-87.nip.io` (CREODIAS)

This avoids the need for public DNS configuration during development/testing.

Sources: [README.md:88-96](), [README.md:100-112](), [minikube/README.md:1-52](), Diagram 5 from high-level overview

## Key Configuration Conventions

### HelmRelease Naming Convention

HelmReleases follow a consistent naming pattern that maps to service names:

| HelmRelease Name | Namespace | Service Name | Configuration File |
|-----------------|-----------|--------------|-------------------|
| `um-login-service` | `um` | `login-service` | `user-management/um-login-service.yaml` |
| `um-identity-service` | `um` | `identity-service` | `user-management/um-identity-service.yaml` |
| `um-pdp-engine` | `um` | `pdp-engine` | `user-management/um-pdp-engine.yaml` |
| `rm-workspace-api` | `rm` | `workspace-api` | `resource-management/rm-workspace-api.yaml` |
| `rm-resource-catalogue` | `rm` | `resource-catalogue` | `resource-management/rm-resource-catalogue.yaml` |
| `rm-data-access` | `rm` | `data-access` | `resource-management/rm-data-access.yaml` |
| `proc-ades` | `proc` | `ades` | `processing-and-chaining/proc-ades.yaml` |
| `proc-application-hub` | `proc` | `application-hub` | `processing-and-chaining/proc-application-hub.yaml` |

### Secret Management

Sensitive configuration is managed using SealedSecrets:
- Encrypted secrets are stored in Git as `SealedSecret` resources
- `sealed-secrets-controller` decrypts them into native `Secret` resources
- Pattern: `{service}-sealedsecret.yaml` files in component directories
- Created using: `kubeseal --format yaml < secret.yaml > sealedsecret.yaml`

For details on secret management, see [SealedSecrets](#10.1).

Sources: [system/clusters/README.md:46-50](), [README.md:100-112]()

## Summary

The EOEPCA system architecture implements a modular, standards-based platform for Earth Observation data exploitation. The architecture is characterized by:

1. **Clear Domain Separation** - Three building blocks (User Management, Resource Management, Processing & Chaining) deployed in isolated namespaces
2. **GitOps-Driven Operations** - Flux CD continuously reconciles cluster state with Git repository
3. **Multi-Tenant Isolation** - Workspace API provisions per-user namespaces with dedicated resources
4. **Standards Compliance** - OGC APIs for interoperability across data access, cataloging, and processing
5. **Infrastructure Flexibility** - Supports cloud (OpenStack/RKE) and local (Minikube/k3s) deployments

For specific component details, refer to the child pages under [Building Blocks Overview](#3.1) and the domain-specific sections ([User Management](#4), [Resource Management](#5), [Processing and Chaining](#6)).

Sources: [README.md:58-68](), [README.md:128-160](), [system/clusters/README.md:1-95]()