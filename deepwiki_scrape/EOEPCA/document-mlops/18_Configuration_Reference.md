# Configuration Reference

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/admin/configuration.md](docs/admin/configuration.md)

</details>



This page provides a comprehensive overview of configuration for the EOEPCA MLOps Building Block components. It explains the configuration architecture, sources, and integration points between GitLab, SharingHub, and MLflow SharingHub.

For detailed configuration options for specific components, see:
- [SharingHub Configuration](#6.1) - Complete reference for SharingHub server settings, STAC API, categories, and integrations
- [MLflow SharingHub Configuration](#6.2) - Configuration options for experiment tracking and model registry

For deployment procedures that use these configurations, see [Deployment Guide](#5).

## Configuration Architecture

The MLOps Building Block uses a multi-layered configuration approach where each component has distinct configuration concerns, but they must be properly integrated to work together as a unified system.

```mermaid
graph TB
    subgraph "Configuration Layers"
        HELM["Helm Values<br/>(values.yaml)"]
        ENV["Environment Variables"]
        SECRETS["Kubernetes Secrets"]
        CONFIG["YAML Configuration<br/>(embedded in values)"]
    end
    
    subgraph "Component Configuration"
        GLCONFIG["GitLab Configuration<br/>gitlab-values.yaml"]
        SHCONFIG["SharingHub Configuration<br/>sharinghub-values.yaml"]
        MLFCONFIG["MLflow SharingHub Configuration<br/>mlflow-sharinghub-values.yaml"]
    end
    
    subgraph "Configuration Content"
        INTEGRATION["Integration Settings<br/>URLs, OAuth, Tokens"]
        STORAGE["Storage Settings<br/>S3, PostgreSQL"]
        STAC["STAC Catalog<br/>Categories, Extensions"]
        SECURITY["Security Settings<br/>OIDC, Sessions, TLS"]
    end
    
    HELM --> GLCONFIG
    HELM --> SHCONFIG
    HELM --> MLFCONFIG
    
    ENV --> GLCONFIG
    ENV --> SHCONFIG
    ENV --> MLFCONFIG
    
    SECRETS --> GLCONFIG
    SECRETS --> SHCONFIG
    SECRETS --> MLFCONFIG
    
    CONFIG --> SHCONFIG
    CONFIG --> MLFCONFIG
    
    GLCONFIG --> INTEGRATION
    SHCONFIG --> INTEGRATION
    MLFCONFIG --> INTEGRATION
    
    GLCONFIG --> STORAGE
    SHCONFIG --> STORAGE
    MLFCONFIG --> STORAGE
    
    SHCONFIG --> STAC
    SHCONFIG --> SECURITY
    MLFCONFIG --> SECURITY
```

**Configuration Hierarchy and Sources**

Sources: [docs/admin/configuration.md:1-281]()

### Configuration Sources

The MLOps Building Block uses three primary configuration sources, each serving different purposes:

| Source | Purpose | Usage | Security Level |
|--------|---------|-------|----------------|
| **Helm Values** | Deployment-time configuration | Define component behavior, resource limits, ingress rules | Public |
| **YAML Config** | Application-level settings | Embedded in Helm values under `config` field | Public/Semi-sensitive |
| **Environment Variables** | Runtime overrides | Override specific settings like `DEBUG`, `LOG_LEVEL` | Public |
| **Kubernetes Secrets** | Sensitive credentials | OAuth secrets, database passwords, API tokens | Sensitive |

```mermaid
graph LR
    subgraph "Configuration Priority"
        HIGH["1. Kubernetes Secrets<br/>(Highest Priority)"]
        MED["2. Environment Variables"]
        LOW["3. YAML Config in Helm"]
        DEFAULT["4. Application Defaults<br/>(Lowest Priority)"]
    end
    
    HIGH --> APP["Running Application"]
    MED --> APP
    LOW --> APP
    DEFAULT --> APP
    
    subgraph "Secret Types"
        OAUTH["OAuth Secrets<br/>gitlab-oidc<br/>sharinghub-oidc"]
        DB["Database Credentials<br/>mlflow-sharinghub-postgres"]
        S3SEC["S3 Credentials<br/>sharinghub-s3<br/>mlflow-sharinghub-s3"]
        SESSION["Session Keys<br/>sharinghub-secret<br/>mlflow-sharinghub"]
    end
    
    OAUTH --> HIGH
    DB --> HIGH
    S3SEC --> HIGH
    SESSION --> HIGH
```

**Configuration Source Priority and Secret Management**

Sources: [docs/admin/configuration.md:52-232]()

## Component Configuration Overview

Each component in the MLOps Building Block has distinct configuration requirements:

| Component | Configuration Scope | Key Areas | Primary Method |
|-----------|-------------------|-----------|----------------|
| **GitLab** | Comprehensive | S3 storage, OIDC, LFS, backups, ingress | Helm values |
| **SharingHub** | Extensive | Server settings, GitLab integration, STAC catalog, categories, tags, S3 store | YAML config in Helm values |
| **MLflow SharingHub** | Lightweight | SharingHub URL, backend store, artifacts store | Helm values + secrets |

### SharingHub as Central Configuration Hub

SharingHub is the most heavily configured component because it acts as the integration layer between GitLab and MLflow SharingHub:

```mermaid
graph TB
    subgraph "SharingHub Configuration Domains"
        SERVER["Server Settings<br/>server.debug<br/>server.log-level<br/>server.cache"]
        GITLAB["GitLab Integration<br/>gitlab.url<br/>gitlab.allow-public<br/>OAuth client"]
        STACCONF["STAC API<br/>stac.root<br/>stac.categories<br/>stac.extensions"]
        MLFLOWCONF["MLflow Integration<br/>mlflow.type<br/>mlflow.url"]
        S3CONF["S3 Store<br/>s3.bucket<br/>s3.endpoint<br/>services.store"]
        TAGS["UI Tags<br/>tags.sections<br/>tags.gitlab"]
    end
    
    subgraph "Configuration Usage"
        DISCOVERY["Project Discovery<br/>via GitLab topics"]
        CATALOG["STAC Catalog<br/>Generation"]
        AUTH["Authentication<br/>& Authorization"]
        STORAGE["Storage API<br/>DVC support"]
    end
    
    GITLAB --> DISCOVERY
    GITLAB --> AUTH
    STACCONF --> CATALOG
    TAGS --> CATALOG
    MLFLOWCONF --> CATALOG
    S3CONF --> STORAGE
    SERVER --> AUTH
    SERVER --> CATALOG
    SERVER --> STORAGE
```

**SharingHub Configuration Domains and Usage**

Sources: [docs/admin/configuration.md:7-280]()

### Configuration File Structure

The SharingHub configuration is embedded in the Helm values as a YAML string under the `config` field:

```yaml
# In sharinghub-values.yaml
config: |
  server:
    debug: false
    log-level: INFO
  gitlab:
    url: https://gitlab.example.com
    allow-public: true
  stac:
    root:
      id: gitlab-cs
      title: SharingHub STAC Catalog
    categories:
      - ai-model:
          gitlab_topic: sharinghub:aimodel
```

This structure allows the entire application configuration to be version-controlled and deployed atomically.

Sources: [docs/admin/configuration.md:9-15]()

## Integration Configuration

The key to a functioning MLOps Building Block is proper configuration of integration points between components:

```mermaid
graph TB
    subgraph "GitLab Configuration"
        GLURL["GitLab URL"]
        GLOAUTH["OAuth Application<br/>client-id, client-secret"]
        GLPUBLIC["Public Projects<br/>allow-public: true/false"]
    end
    
    subgraph "SharingHub Configuration"
        SHGLURL["gitlab.url"]
        SHGLOAUTH["OAuth client in secret<br/>sharinghub-oidc"]
        SHDEFTOKEN["Default Token<br/>(optional)"]
        SHMLFURL["mlflow.url"]
        SHMLFTYPE["mlflow.type: mlflow-sharinghub"]
    end
    
    subgraph "MLflow SharingHub Configuration"
        MLFSHURL["sharinghubUrl"]
        MLFSTACCOL["sharinghubStacCollection<br/>(e.g., ai-model)"]
        MLFDEFTOKEN["sharinghubAuthDefaultToken"]
    end
    
    GLURL --> SHGLURL
    GLOAUTH --> SHGLOAUTH
    GLPUBLIC --> SHDEFTOKEN
    
    SHGLOAUTH --> AUTH["User Authentication"]
    SHDEFTOKEN --> AUTH
    
    SHMLFURL --> MLFSHURL
    SHMLFTYPE --> INTEGRATION["Integration Type"]
    
    MLFSHURL --> PERMCHECK["Permission Checking"]
    MLFSTACCOL --> PERMCHECK
    MLFDEFTOKEN --> PERMCHECK
```

**Integration Configuration Dependencies**

Sources: [docs/admin/configuration.md:88-117](), [docs/admin/configuration.md:283-298]()

### Critical Integration Points

| Integration | Configuration Location | Purpose | Required Settings |
|-------------|----------------------|---------|-------------------|
| **GitLab → SharingHub** | SharingHub `gitlab` section | Project discovery, authentication | `gitlab.url`, OAuth client secret |
| **SharingHub → MLflow** | SharingHub `mlflow` section | Model tracking integration | `mlflow.type`, `mlflow.url` |
| **MLflow → SharingHub** | MLflow `sharinghubUrl` | Permission validation | `sharinghubUrl`, `sharinghubStacCollection` |
| **User → GitLab** | GitLab OIDC provider | OIDC authentication | Keycloak client credentials |

## Security and Secrets Management

Sensitive configuration values must be stored in Kubernetes secrets and mounted into the application containers:

```mermaid
graph TB
    subgraph "Secret Creation"
        CMD1["kubectl create secret<br/>gitlab-oidc"]
        CMD2["kubectl create secret<br/>sharinghub-oidc"]
        CMD3["kubectl create secret<br/>sharinghub-s3"]
        CMD4["kubectl create secret<br/>mlflow-sharinghub"]
        CMD5["kubectl create secret<br/>mlflow-sharinghub-postgres"]
        CMD6["kubectl create secret<br/>mlflow-sharinghub-s3"]
    end
    
    subgraph "Secret Contents"
        OAUTH_GL["client-id<br/>client-secret"]
        OAUTH_SH["client-id<br/>client-secret<br/>default-token (optional)"]
        S3_SH["access-key<br/>secret-key"]
        MLF_SEC["secret-key<br/>backend-store-uri (optional)"]
        MLF_PG["password<br/>postgres-password"]
        MLF_S3["access-key-id<br/>secret-access-key"]
    end
    
    subgraph "Mounted As"
        ENV_VARS["Environment Variables"]
        VOL_MOUNTS["Volume Mounts"]
    end
    
    CMD1 --> OAUTH_GL
    CMD2 --> OAUTH_SH
    CMD3 --> S3_SH
    CMD4 --> MLF_SEC
    CMD5 --> MLF_PG
    CMD6 --> MLF_S3
    
    OAUTH_GL --> ENV_VARS
    OAUTH_SH --> ENV_VARS
    S3_SH --> ENV_VARS
    MLF_SEC --> ENV_VARS
    MLF_PG --> ENV_VARS
    MLF_S3 --> ENV_VARS
```

**Secret Management Flow**

Sources: [docs/admin/configuration.md:109-117](), [docs/admin/configuration.md:229-232](), [docs/admin/configuration.md:310-347]()

### Secret Types and Usage

| Secret Name | Namespace | Contents | Used By | Purpose |
|-------------|-----------|----------|---------|---------|
| `gitlab-oidc` | `gitlab` | `client-id`, `client-secret` | GitLab | OIDC authentication with Keycloak |
| `sharinghub-oidc` | `sharinghub` | `client-id`, `client-secret`, `default-token` | SharingHub | OAuth authentication with GitLab |
| `sharinghub-secret` | `sharinghub` | `secret-key` | SharingHub | Session cookie signing |
| `sharinghub-s3` | `sharinghub` | `access-key`, `secret-key` | SharingHub | S3 store API (DVC) |
| `mlflow-sharinghub` | `sharinghub` | `secret-key`, `backend-store-uri` | MLflow SharingHub | Flask secret, database URI |
| `mlflow-sharinghub-postgres` | `sharinghub` | `password`, `postgres-password` | PostgreSQL, MLflow | Database authentication |
| `mlflow-sharinghub-s3` | `sharinghub` | `access-key-id`, `secret-access-key` | MLflow SharingHub | Artifact storage |

## Configuration Validation and Troubleshooting

### Common Configuration Issues

```mermaid
graph TB
    subgraph "Authentication Issues"
        AUTH1["OAuth misconfigured"]
        AUTH2["Default token missing"]
        AUTH3["OIDC provider unreachable"]
    end
    
    subgraph "Integration Issues"
        INT1["GitLab URL mismatch"]
        INT2["MLflow URL incorrect"]
        INT3["Wrong STAC collection"]
    end
    
    subgraph "Storage Issues"
        STOR1["S3 credentials invalid"]
        STOR2["PostgreSQL connection failed"]
        STOR3["Bucket doesn't exist"]
    end
    
    subgraph "Symptoms"
        SYMP1["Users can't login"]
        SYMP2["Projects not appearing"]
        SYMP3["MLflow permission denied"]
        SYMP4["DVC push/pull fails"]
        SYMP5["Artifacts not stored"]
    end
    
    AUTH1 --> SYMP1
    AUTH2 --> SYMP2
    AUTH3 --> SYMP1
    
    INT1 --> SYMP2
    INT2 --> SYMP3
    INT3 --> SYMP3
    
    STOR1 --> SYMP4
    STOR2 --> SYMP5
    STOR3 --> SYMP4
```

**Common Configuration Issues and Symptoms**

Sources: [docs/admin/configuration.md:88-232]()

### Configuration Verification Checklist

| Check | Component | Verification Method | Expected Result |
|-------|-----------|---------------------|-----------------|
| **GitLab OAuth** | SharingHub | Access `/auth/gitlab` | Redirects to GitLab login |
| **Default Token** | SharingHub | Access `/api/stac` unauthenticated | Returns public projects |
| **STAC Catalog** | SharingHub | GET `/api/stac` | Returns root catalog with collections |
| **MLflow Permission** | MLflow SharingHub | Create experiment in project | Success if user has access |
| **S3 Store** | SharingHub | DVC push to project | Uploads to S3 bucket |
| **PostgreSQL** | MLflow SharingHub | Check pod logs | No connection errors |

## Configuration Updates and Changes

When updating configuration, different methods require different procedures:

| Configuration Type | Update Method | Restart Required | Downtime |
|-------------------|---------------|------------------|----------|
| **Helm Values** | Update ArgoCD Application | Yes | Brief |
| **YAML Config** | Update ArgoCD Application | Yes | Brief |
| **Secrets** | Update secret, restart pods | Yes | Brief |
| **Environment Variables** | Update deployment, restart | Yes | Brief |
| **Dynamic Settings** | API calls (if supported) | No | None |

The SharingHub server includes a caching system that improves performance but may delay configuration changes. Cache timeouts are configurable:

- `checker.cache-timeout`: Cache for API checks (default: 30.0 seconds)
- `s3.check-access.cache-timeout`: Cache for S3 permission checks (default: 30.0 seconds)
- `stac.projects.cache-timeout`: Cache for STAC item generation (default: 30.0 seconds)

For immediate effect after configuration changes, either wait for cache expiration or restart the affected pods.

Sources: [docs/admin/configuration.md:64-86]()

## Next Steps

For detailed configuration options and examples:

- **[SharingHub Configuration](#6.1)** - Complete reference including server settings, GitLab integration, STAC catalog structure, categories, tags, S3 store, and alert messages
- **[MLflow SharingHub Configuration](#6.2)** - Backend store options (SQLite, PostgreSQL), artifacts storage (local, S3), and SharingHub integration settings

For applying these configurations during deployment, see:

- **[Deployment Guide](#5)** - Complete deployment procedures
- **[SharingHub Deployment](#5.3)** - Specific deployment steps for SharingHub
- **[MLflow SharingHub Deployment](#5.4)** - Specific deployment steps for MLflow SharingHub

Sources: [docs/admin/configuration.md:1-359]()