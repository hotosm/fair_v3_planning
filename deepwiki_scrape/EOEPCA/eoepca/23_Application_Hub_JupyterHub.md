# Application Hub (JupyterHub)

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [system/clusters/creodias/processing-and-chaining/application-hub-sealed-secrets-create.sh](system/clusters/creodias/processing-and-chaining/application-hub-sealed-secrets-create.sh)
- [system/clusters/creodias/processing-and-chaining/application-hub-sealed-secrets.yaml](system/clusters/creodias/processing-and-chaining/application-hub-sealed-secrets.yaml)
- [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml](system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml)

</details>



## Purpose and Scope

The Application Hub provides a multi-user JupyterHub environment for interactive development, testing, and execution of Earth Observation processing applications. It enables users to develop Common Workflow Language (CWL) application packages in Jupyter notebooks before deploying them to the ADES for production execution. The hub integrates with EOEPCA's identity management system via OAuth2, provisions isolated user environments with configurable resource profiles, and provides access to workspace storage for data persistence.

For information about deploying applications to production, see [ADES (Application Deployment and Execution Service)](#6.1). For details on the development environment setup, see [Processor Development Environment (PDE)](#6.3).

---

## System Overview

The Application Hub is deployed as a Kubernetes-native JupyterHub instance in the `proc` namespace. It serves as the primary development interface for users to interact with EOEPCA services, providing authenticated notebook environments with pre-configured access to the platform's resource management and processing capabilities.

**Deployment Architecture**

```mermaid
graph TB
    subgraph "External Access"
        User["User Browser"]
    end
    
    subgraph "Ingress Layer"
        Ingress["Nginx Ingress<br/>applicationhub.develop.eoepca.org"]
        TLS["TLS Certificate<br/>applicationhub-tls"]
    end
    
    subgraph "proc Namespace"
        Hub["JupyterHub Hub<br/>application-hub"]
        Proxy["JupyterHub Proxy"]
        HubDB[("SQLite DB<br/>sqlite-pvc<br/>managed-nfs-storage")]
        Secrets["SealedSecret<br/>application-hub-secrets"]
    end
    
    subgraph "User Pods"
        Notebook1["Single-user Notebook<br/>jupyter/minimal-notebook"]
        Notebook2["Single-user Notebook<br/>eoepca/pde-container"]
        UserPVC["User PVC<br/>managed-nfs-storage"]
    end
    
    subgraph "um Namespace"
        Identity["Identity Service<br/>Keycloak"]
        OAuth["OAuth2 Endpoints<br/>auth.develop.eoepca.org"]
    end
    
    User -->|HTTPS| Ingress
    Ingress --> TLS
    Ingress --> Proxy
    Proxy --> Hub
    Hub --> HubDB
    Hub --> Secrets
    
    Hub -->|Spawns| Notebook1
    Hub -->|Spawns| Notebook2
    Notebook1 --> UserPVC
    Notebook2 --> UserPVC
    
    Hub -->|OAuth2 Flow| OAuth
    OAuth --> Identity
    User -->|Authenticate| OAuth
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:1-99]()

---

## HelmRelease Configuration

The Application Hub is deployed via a Flux CD HelmRelease that references the `eoepca/application-hub` Helm chart. The deployment is configured through the `proc-application-hub.yaml` manifest.

| Configuration Aspect | Value | Description |
|---------------------|-------|-------------|
| HelmRelease Name | `application-hub` | Flux resource identifier |
| Namespace | `proc` | Kubernetes namespace for processing services |
| Chart Version | `2.0.52` | EOEPCA application-hub chart version |
| Helm Repository | `eoepca` (common namespace) | Chart source reference |
| Reconciliation Interval | `1m0s` | Flux checks for updates every minute |

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:1-15](), [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:99]()

### Ingress Configuration

The hub is exposed via an ingress resource with TLS termination:

```mermaid
graph LR
    Internet["Internet Traffic"] -->|HTTPS| IngressHost["applicationhub.develop.eoepca.org"]
    IngressHost --> TLSSecret["Secret: applicationhub-tls"]
    IngressHost --> Path["Path: /<br/>Type: ImplementationSpecific"]
    Path --> HubService["Service: application-hub"]
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:16-27]()

---

## OAuth2 Authentication Integration

The Application Hub implements OAuth2 authentication against EOEPCA's Identity Service, enabling single sign-on and centralized user management. The authentication flow uses the authorization code grant with PKCE.

**OAuth2 Configuration Flow**

```mermaid
sequenceDiagram
    participant User
    participant JupyterHub as "JupyterHub Hub<br/>application-hub"
    participant OAuth as "OAuth2 Endpoints<br/>auth.develop.eoepca.org"
    participant Identity as "Identity Service<br/>Keycloak"
    
    User->>JupyterHub: Access https://applicationhub.develop.eoepca.org
    JupyterHub->>JupyterHub: No valid session
    JupyterHub->>OAuth: Redirect to OAUTH2_AUTHORIZE_URL
    OAuth->>User: Display login page
    User->>OAuth: Enter credentials
    OAuth->>Identity: Validate credentials
    Identity-->>OAuth: User authenticated
    OAuth->>JupyterHub: Redirect to OAUTH_CALLBACK_URL<br/>with authorization code
    JupyterHub->>OAuth: Exchange code at OAUTH2_TOKEN_URL<br/>using OAUTH_CLIENT_ID/SECRET
    OAuth-->>JupyterHub: Return access token
    JupyterHub->>OAuth: GET OAUTH2_USERDATA_URL<br/>with access token
    OAuth-->>JupyterHub: User info (user_name)
    JupyterHub->>JupyterHub: Extract username via OAUTH2_USERNAME_KEY
    JupyterHub->>User: Spawn single-user notebook
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:38-43]()

### OAuth2 Environment Variables

The hub's OAuth2 integration is configured via environment variables in the hub pod:

| Environment Variable | Value | Purpose |
|---------------------|-------|---------|
| `OAUTH_CALLBACK_URL` | `https://applicationhub.develop.eoepca.org/hub/oauth_callback` | Redirect URI after authentication |
| `OAUTH2_AUTHORIZE_URL` | `https://auth.develop.eoepca.org/oxauth/restv1/authorize` | OAuth2 authorization endpoint |
| `OAUTH2_TOKEN_URL` | `https://auth.develop.eoepca.org/oxauth/restv1/token` | Token exchange endpoint |
| `OAUTH2_USERDATA_URL` | `https://auth.develop.eoepca.org/oxauth/restv1/userinfo` | User profile endpoint |
| `OAUTH2_USERNAME_KEY` | `user_name` | JSON key for extracting username from userinfo response |
| `OAUTH_LOGOUT_REDIRECT_URL` | `https://applicationhub.develop.eoepca.org` | Post-logout redirect target |

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:38-43]()

### Secret Management

OAuth2 credentials and cryptographic keys are stored in a SealedSecret:

```mermaid
graph TB
    SealedSecret["SealedSecret<br/>application-hub-secrets<br/>namespace: proc"]
    Secret["Secret<br/>application-hub-secrets"]
    
    SecretKeys["Encrypted Keys:<br/>- JUPYTERHUB_CRYPT_KEY<br/>- OAUTH_CLIENT_ID<br/>- OAUTH_CLIENT_SECRET"]
    
    HubPod["Hub Pod"]
    EnvVars["Environment Variables"]
    
    SealedSecret -->|"Decrypted by<br/>sealed-secrets controller"| Secret
    Secret --> SecretKeys
    HubPod -->|"References via<br/>existingSecret"| Secret
    SecretKeys -->|"Mounted as"| EnvVars
```

The sealed secrets are created using the `application-hub-sealed-secrets-create.sh` script, which generates a Kubernetes Secret and pipes it through `kubeseal` for encryption. The sealed secret is then committed to Git and decrypted at runtime by the sealed-secrets controller.

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:34](), [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:47-63](), [system/clusters/creodias/processing-and-chaining/application-hub-sealed-secrets.yaml:1-17](), [system/clusters/creodias/processing-and-chaining/application-hub-sealed-secrets-create.sh:1-36]()

---

## Single-User Notebook Environments

When a user authenticates, JupyterHub spawns a dedicated single-user notebook server pod. The environment configuration is controlled by the `singleuser` and `profileList` sections of the HelmRelease.

**User Environment Spawning**

```mermaid
graph TB
    Hub["JupyterHub Hub"]
    KubeSpawner["KubeSpawner<br/>Kubernetes Pod Spawner"]
    
    subgraph "User Pod Specification"
        Profile["Selected Profile"]
        Image["Container Image"]
        Resources["CPU/Memory Limits"]
        Storage["PVC Mount"]
    end
    
    subgraph "Created Resources"
        Pod["Pod:<br/>{username}-{hash}"]
        PVC["PVC:<br/>claim-{username}"]
        NFSStorage["NFS Storage<br/>managed-nfs-storage"]
    end
    
    Hub -->|"User login"| KubeSpawner
    KubeSpawner --> Profile
    Profile --> Image
    Profile --> Resources
    Profile --> Storage
    
    KubeSpawner -->|"Create"| Pod
    KubeSpawner -->|"Provision"| PVC
    PVC --> NFSStorage
    Pod -->|"Mount"| PVC
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:85-98]()

### Default Container Image

The default single-user image is `jupyter/minimal-notebook:2343e33dec46`, providing a baseline Python environment. The image can be overridden via environment variable or profile selection:

```
JUPYTERHUB_SINGLE_USER_IMAGE: "eoepca/pde-container:1.0.3"
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:37](), [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:86-88]()

### Profile List Configuration

Users can select from multiple environment profiles at spawn time. The profile list defines different resource allocations and container images:

| Profile Display Name | Description | Default | Resource Overrides |
|---------------------|-------------|---------|-------------------|
| `Minimal environment` | Baseline Python environment | `True` | None (uses singleuser defaults) |
| `EOEPCA profile` | Enhanced environment for EO processing | `False` | `cpu_limit: 4`, `mem_limit: 8G` |

The profile configuration uses `kubespawner_override` to customize pod specifications. Note that profile definitions support arbitrary KubeSpawner configuration keys.

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:89-98]()

---

## Storage and Persistence

The Application Hub provisions persistent storage at multiple levels to ensure data durability across pod restarts.

**Storage Architecture**

```mermaid
graph TB
    subgraph "Hub Persistence"
        HubPod["Hub Pod"]
        HubDB["Hub Database<br/>sqlite-pvc"]
        HubPVC["PVC<br/>1Gi<br/>managed-nfs-storage"]
    end
    
    subgraph "User Persistence"
        UserPod1["User Notebook Pod<br/>alice"]
        UserPod2["User Notebook Pod<br/>bob"]
        UserPVC1["PVC<br/>claim-alice"]
        UserPVC2["PVC<br/>claim-bob"]
    end
    
    subgraph "Storage Backend"
        StorageClass["StorageClass<br/>managed-nfs-storage"]
        NFSServer["NFS Server"]
        NFSExport["/data/eoepca"]
    end
    
    HubPod --> HubDB
    HubDB --> HubPVC
    HubPVC --> StorageClass
    
    UserPod1 --> UserPVC1
    UserPod2 --> UserPVC2
    UserPVC1 --> StorageClass
    UserPVC2 --> StorageClass
    
    StorageClass --> NFSServer
    NFSServer --> NFSExport
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:44](), [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:73-83]()

### Hub Database Configuration

The JupyterHub hub maintains its state in a SQLite database stored on a PersistentVolume:

| Configuration Key | Value | Description |
|------------------|-------|-------------|
| `jupyterhub.hub.db.type` | `sqlite-pvc` | Use SQLite with PVC backing |
| `jupyterhub.hub.db.pvc.storage` | `1Gi` | Database volume size |
| `jupyterhub.hub.db.pvc.storageClassName` | `managed-nfs-storage` | NFS-backed storage class |
| `jupyterhub.hub.db.pvc.accessModes` | `[ReadWriteOnce]` | Single-writer access mode |

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:73-83]()

### User Notebook Storage

Each user's notebook environment receives a dedicated PersistentVolumeClaim provisioned by the `managed-nfs-storage` StorageClass. The storage class is configured via the `STORAGE_CLASS` environment variable. User home directories persist across pod restarts, maintaining notebooks, data, and configuration files.

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:44]()

---

## Integration with EOEPCA Services

The Application Hub environment is pre-configured to interact with other EOEPCA building blocks, enabling users to develop applications that leverage platform capabilities.

**Service Integration Points**

```mermaid
graph TB
    Notebook["User Notebook<br/>eoepca/pde-container"]
    
    subgraph "Resource Management"
        WorkspaceAPI["Workspace API<br/>develop-user-{username}"]
        ResourceCat["Resource Catalogue<br/>CSW/OpenSearch"]
        DataAccess["Data Access<br/>OGC WMS/WCS"]
    end
    
    subgraph "Processing Services"
        ADES["ADES<br/>OGC Processes API"]
        CWLDeploy["CWL Deployment"]
    end
    
    subgraph "Identity"
        OAuth["OAuth2 Tokens"]
    end
    
    Notebook -->|"Query metadata"| ResourceCat
    Notebook -->|"Preview data"| DataAccess
    Notebook -->|"Access workspace storage"| WorkspaceAPI
    Notebook -->|"Deploy application"| ADES
    Notebook -->|"Package workflow"| CWLDeploy
    
    Notebook -->|"Authenticated via"| OAuth
    OAuth -->|"Secures"| WorkspaceAPI
    OAuth -->|"Secures"| ADES
```

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:37](), [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:45]()

### Workspace Integration

The `RESOURCE_MANAGER_WORKSPACE_PREFIX` environment variable configures the naming convention for user workspaces. With the value `develop-user`, user workspaces are accessed at `develop-user-{username}`. This allows notebooks to programmatically construct workspace URLs and access user-specific resource catalogues and data access services.

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:45]()

### PDE Container Image

The `JUPYTERHUB_SINGLE_USER_IMAGE` can be set to `eoepca/pde-container:1.0.3`, which provides a Processor Development Environment with pre-installed tools for CWL development, STAC manipulation, and EOEPCA service interaction. This image includes libraries for interacting with the ADES and workspace services.

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:37]()

---

## Environment Configuration Summary

The following table summarizes all environment variables configured in the hub pod:

| Environment Variable | Source | Value/Description |
|---------------------|--------|-------------------|
| `JUPYTERHUB_ENV` | Static | `dev` - Development environment identifier |
| `JUPYTERHUB_SINGLE_USER_IMAGE` | Static | `eoepca/pde-container:1.0.3` - Default user container |
| `OAUTH_CALLBACK_URL` | Static | `https://applicationhub.develop.eoepca.org/hub/oauth_callback` |
| `OAUTH2_USERDATA_URL` | Static | `https://auth.develop.eoepca.org/oxauth/restv1/userinfo` |
| `OAUTH2_TOKEN_URL` | Static | `https://auth.develop.eoepca.org/oxauth/restv1/token` |
| `OAUTH2_AUTHORIZE_URL` | Static | `https://auth.develop.eoepca.org/oxauth/restv1/authorize` |
| `OAUTH_LOGOUT_REDIRECT_URL` | Static | `https://applicationhub.develop.eoepca.org` |
| `OAUTH2_USERNAME_KEY` | Static | `user_name` - Key in userinfo response |
| `STORAGE_CLASS` | Static | `managed-nfs-storage` - PVC storage class |
| `RESOURCE_MANAGER_WORKSPACE_PREFIX` | Static | `develop-user` - Workspace naming prefix |
| `JUPYTERHUB_CRYPT_KEY` | Secret | Reference to `application-hub-secrets` |
| `OAUTH_CLIENT_ID` | Secret | Reference to `application-hub-secrets` |
| `OAUTH_CLIENT_SECRET` | Secret | Reference to `application-hub-secrets` |

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:35-63]()

---

## Deployment and Lifecycle Management

The Application Hub follows EOEPCA's GitOps deployment model. Configuration changes are committed to the repository, and Flux CD automatically reconciles the cluster state.

**Deployment Workflow**

```mermaid
graph LR
    GitRepo["Git Repository<br/>proc-application-hub.yaml"]
    FluxSource["Flux SourceController<br/>Monitors repository"]
    FluxHelm["Flux HelmController<br/>Manages HelmRelease"]
    HelmChart["HelmChart<br/>eoepca/application-hub:2.0.52"]
    K8sResources["Kubernetes Resources<br/>Deployment, Service, Ingress"]
    
    GitRepo -->|"Poll every 1m"| FluxSource
    FluxSource -->|"Detects change"| FluxHelm
    FluxHelm -->|"Fetch"| HelmChart
    HelmChart -->|"Render"| FluxHelm
    FluxHelm -->|"Apply"| K8sResources
```

The reconciliation interval is set to `1m0s`, meaning Flux checks for configuration changes and chart updates every minute and automatically applies them to the cluster.

**Sources:** [system/clusters/creodias/processing-and-chaining/proc-application-hub.yaml:99]()