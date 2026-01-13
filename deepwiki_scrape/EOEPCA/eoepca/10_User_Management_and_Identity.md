# User Management and Identity

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [system/clusters/creodias/system/demo/hr-django-portal.yaml](system/clusters/creodias/system/demo/hr-django-portal.yaml)
- [system/clusters/creodias/system/demo/hr-eoepca-portal.yaml](system/clusters/creodias/system/demo/hr-eoepca-portal.yaml)
- [system/clusters/creodias/system/demo/ss-django-secrets-create.sh](system/clusters/creodias/system/demo/ss-django-secrets-create.sh)
- [system/clusters/creodias/system/demo/ss-django-secrets.yaml](system/clusters/creodias/system/demo/ss-django-secrets.yaml)
- [system/clusters/creodias/system/storage/hr-storage.yaml](system/clusters/creodias/system/storage/hr-storage.yaml)
- [system/clusters/creodias/system/test/hr-cheese.yaml](system/clusters/creodias/system/test/hr-cheese.yaml)
- [system/clusters/creodias/system/test/identity-dummy-service-ingress.yaml](system/clusters/creodias/system/test/identity-dummy-service-ingress.yaml)
- [system/clusters/creodias/user-management/um-identity-service.yaml](system/clusters/creodias/user-management/um-identity-service.yaml)
- [system/clusters/creodias/user-management/um-login-service.yaml](system/clusters/creodias/user-management/um-login-service.yaml)
- [system/clusters/creodias/user-management/um-pdp-engine.yaml](system/clusters/creodias/user-management/um-pdp-engine.yaml)
- [system/clusters/creodias/user-management/um-user-profile.yaml](system/clusters/creodias/user-management/um-user-profile.yaml)

</details>



## Purpose and Scope

This document describes the User Management and Identity building block of the EOEPCA platform, which provides authentication and authorization capabilities for all platform services. The system implements User-Managed Access (UMA) 2.0 flows using OpenID Connect (OIDC) and enables fine-grained policy-based access control.

For detailed information on specific components, see:
- Identity Service (Keycloak) implementation: [Identity Service (Keycloak)](#4.1)
- Login Service (Gluu) implementation: [Login Service (Gluu)](#4.2)
- Policy enforcement architecture: [Policy Enforcement (PEP/PDP)](#4.3)
- Authentication flow details: [UMA Authentication Flow](#4.4)

## Architecture Overview

The User Management building block consists of four primary subsystems deployed in the `um` namespace:

| Component | Helm Chart | Primary Function |
|-----------|------------|------------------|
| Identity Service | `identity-service` | Keycloak-based identity provider, token management, UMA resource server |
| Login Service | `login-service` | Gluu-based authentication frontend with LDAP user directory |
| PDP Engine | `pdp-engine` | Policy Decision Point for authorization decisions |
| User Profile | `user-profile` | User metadata and profile management |

```mermaid
graph TB
    subgraph "um namespace"
        subgraph "Identity Service"
            Keycloak["identity-keycloak<br/>(Pod)"]
            IdentityAPI["identity-api<br/>(Pod)"]
            IdentityMgr["identity-manager<br/>(Pod)"]
            IdentityGK["identity-gatekeeper<br/>(Pod)"]
            IdentityDB[("identity-postgres<br/>(StatefulSet)")]
        end
        
        subgraph "Login Service"
            OpenDJ["opendj<br/>(LDAP Directory)"]
            OxAuth["oxauth<br/>(Authentication)"]
            OxTrust["oxtrust<br/>(Admin UI)"]
            OxPassport["oxpassport<br/>(Federation)"]
        end
        
        PDP["pdp-engine<br/>(Pod)"]
        UserProfile["user-profile<br/>(Pod)"]
    end
    
    subgraph "External Access"
        UserRequest["User Request"]
        ServiceRequest["Service Request"]
    end
    
    subgraph "Protected Services"
        PEP1["PEP (Ingress)<br/>ADES"]
        PEP2["PEP (Ingress)<br/>Workspace API"]
        PEP3["PEP (Ingress)<br/>Resource Catalogue"]
    end
    
    UserRequest -->|"auth.develop.eoepca.org"| OxAuth
    OxAuth -->|"Validate Credentials"| OpenDJ
    OxAuth -->|"Issue ID Token"| UserRequest
    
    ServiceRequest -->|"Protected Resource"| PEP1
    PEP1 -->|"Get UMA Ticket"| Keycloak
    PEP1 -->|"Check Policy"| PDP
    PDP -->|"Validate Token"| Keycloak
    
    IdentityAPI -->|"Manage Resources"| Keycloak
    IdentityMgr -->|"Admin Interface"| Keycloak
    IdentityGK -->|"Auth Proxy"| Keycloak
    Keycloak -->|"Store Data"| IdentityDB
    
    PDP -->|"Fetch User Data"| UserProfile
    UserProfile -->|"Sync with"| OpenDJ
```

**Sources:** [system/clusters/creodias/user-management/um-identity-service.yaml:1-78](), [system/clusters/creodias/user-management/um-login-service.yaml:1-68](), [system/clusters/creodias/user-management/um-pdp-engine.yaml:1-28](), [system/clusters/creodias/user-management/um-user-profile.yaml:1-24]()

## Identity Service Components

The Identity Service is deployed via the `um-identity-service` HelmRelease and consists of multiple sub-components:

### Keycloak Identity Provider

The core identity provider is accessible at `identity.keycloak.develop.eoepca.org` and provides:
- UMA 2.0 resource server capabilities
- Token issuance and validation (ID tokens, access tokens, RPTs)
- Client registration and management
- Realm and user federation

```mermaid
graph LR
    subgraph "identity-service Helm Chart"
        Keycloak["identity-keycloak<br/>Container: keycloak"]
        API["identity-api<br/>Container: identity-api"]
        Manager["identity-manager<br/>Container: identity-manager"]
        Gatekeeper["identity-gatekeeper<br/>Container: gatekeeper"]
        Postgres["identity-postgres<br/>Container: postgresql"]
    end
    
    subgraph "Ingress Routes"
        IngressKC["identity.keycloak.develop.eoepca.org<br/>TLS: identity-keycloak-tls-certificate"]
        IngressAPI["identity.api.develop.eoepca.org<br/>TLS: identity-api-tls-certificate"]
        IngressMgr["identity.manager.develop.eoepca.org<br/>TLS: identity-manager-tls-certificate"]
        IngressGK["identity.gatekeeper.develop.eoepca.org<br/>TLS: identity-gatekeeper-tls-certificate"]
    end
    
    subgraph "Storage"
        PVC["eoepca-userman-pvc<br/>(NFS PersistentVolumeClaim)"]
    end
    
    IngressKC -->|"Route /auth/*"| Keycloak
    IngressAPI -->|"Route /"| API
    IngressMgr -->|"Route /"| Manager
    IngressGK -->|"Route /"| Gatekeeper
    
    Keycloak -->|"Store realm config"| Postgres
    Keycloak -->|"Persistent data"| PVC
    API -->|"REST API to"| Keycloak
    Manager -->|"Web UI for"| Keycloak
    Gatekeeper -->|"Auth proxy to"| Keycloak
    Postgres -->|"Mount volume"| PVC
```

**Configuration Details:**

| Component | Hostname | Purpose |
|-----------|----------|---------|
| `identity-keycloak` | `identity.keycloak.develop.eoepca.org` | Main Keycloak server, UMA endpoints at `/auth/realms/{realm}/authz/*` |
| `identity-api` | `identity.api.develop.eoepca.org` | REST API for resource registration and policy management |
| `identity-manager` | `identity.manager.develop.eoepca.org` | Administrative web interface |
| `identity-gatekeeper` | `identity.gatekeeper.develop.eoepca.org` | OAuth2 proxy for protected admin access |

**Sources:** [system/clusters/creodias/user-management/um-identity-service.yaml:22-76]()

### Persistent Storage

All Identity Service components share the `eoepca-userman-pvc` PersistentVolumeClaim, which is backed by NFS storage. This PVC is pre-created and referenced by the HelmRelease:

```yaml
volumeClaim:
  name: eoepca-userman-pvc
  create: false
```

**Sources:** [system/clusters/creodias/user-management/um-identity-service.yaml:19-21](), [system/clusters/creodias/system/storage/hr-storage.yaml:27-28]()

## Login Service Components

The Login Service is deployed via the `um-login-service` HelmRelease and provides the authentication frontend using Gluu Server 4.x components:

```mermaid
graph TB
    subgraph "login-service Helm Chart"
        subgraph "OpenDJ LDAP"
            OpenDJ["opendj<br/>Container: opendj<br/>Port: 1636"]
            LDAPData[("LDAP Directory<br/>cn=users,dc=example,dc=org")]
        end
        
        subgraph "oxAuth (Authentication Server)"
            OxAuth["oxauth<br/>Container: oxauth<br/>Resources: 1000Mi RAM"]
            OxAuthCache["Embedded Cache"]
        end
        
        subgraph "oxTrust (Admin Portal)"
            OxTrust["oxtrust<br/>Container: oxtrust<br/>Resources: 1500Mi RAM"]
        end
        
        subgraph "oxPassport (Federation)"
            OxPassport["oxpassport<br/>Container: oxpassport<br/>Node.js service"]
        end
        
        Nginx["nginx-ingress<br/>Host: auth.develop.eoepca.org<br/>TLS: gluu-tls-certificate"]
    end
    
    subgraph "External"
        User["End User"]
        AdminUser["Administrator"]
    end
    
    User -->|"Login Request"| Nginx
    Nginx -->|"Route /oxauth/*"| OxAuth
    Nginx -->|"Route /identity/*"| OxTrust
    Nginx -->|"Route /passport/*"| OxPassport
    
    AdminUser -->|"Admin Console"| OxTrust
    
    OxAuth -->|"Query Users"| OpenDJ
    OxAuth -->|"Read LDAP"| LDAPData
    OxTrust -->|"Manage Users"| OpenDJ
    OxTrust -->|"Write LDAP"| LDAPData
    
    OxPassport -->|"External IdP"| OxAuth
```

**Component Resource Allocations:**

| Component | CPU Request | Memory Request | Purpose |
|-----------|-------------|----------------|---------|
| `opendj` | 100m | 300Mi | LDAP directory for user credentials and attributes |
| `oxauth` | 100m | 1000Mi | OAuth2/OIDC authentication server |
| `oxtrust` | 100m | 1500Mi | Administrative web interface for user/client management |
| `oxpassport` | 100m | 100Mi | External identity provider federation (optional) |

**Sources:** [system/clusters/creodias/user-management/um-login-service.yaml:23-56]()

### Domain Configuration

The Login Service is configured with the domain `auth.develop.eoepca.org` and requires the `nginxIp` parameter for proper routing:

```yaml
global:
  domain: auth.develop.eoepca.org
  nginxIp: 185.52.192.231
  namespace: um
```

This configuration ensures all Gluu components are accessible under a unified domain with proper ingress routing.

**Sources:** [system/clusters/creodias/user-management/um-login-service.yaml:19-56]()

## Policy Decision Point

The `pdp-engine` provides the Policy Decision Point (PDP) that makes authorization decisions based on XACML policies. It is deployed in the `um` namespace and integrates with both the Identity Service and Login Service:

```mermaid
graph LR
    subgraph "PDP Engine"
        PDP["pdp-engine<br/>Pod"]
        PolicyStore[("Policy Storage<br/>eoepca-userman-pvc")]
    end
    
    subgraph "Policy Enforcement Points"
        PEP1["ADES PEP<br/>(resource-guard)"]
        PEP2["Workspace PEP<br/>(resource-guard)"]
        PEP3["Catalogue PEP<br/>(resource-guard)"]
    end
    
    subgraph "Identity Services"
        Keycloak["identity-keycloak<br/>Token validation"]
        Gluu["oxauth<br/>User attributes"]
    end
    
    PEP1 -->|"authzRequest(user, resource, action)"| PDP
    PEP2 -->|"authzRequest(user, resource, action)"| PDP
    PEP3 -->|"authzRequest(user, resource, action)"| PDP
    
    PDP -->|"Validate RPT"| Keycloak
    PDP -->|"Fetch user attributes"| Gluu
    PDP -->|"Load policies"| PolicyStore
    
    PDP -->|"Decision: Permit/Deny"| PEP1
    PDP -->|"Decision: Permit/Deny"| PEP2
    PDP -->|"Decision: Permit/Deny"| PEP3
```

**Configuration:**

The PDP Engine shares the same persistent volume and domain configuration as other user management components:

```yaml
global:
  nginxIp: 185.52.192.231
  domain: auth.develop.eoepca.org
volumeClaim:
  name: eoepca-userman-pvc
  create: false
```

**Sources:** [system/clusters/creodias/user-management/um-pdp-engine.yaml:1-28]()

## User Profile Service

The `user-profile` service manages user metadata and profile information, acting as a bridge between the Identity Service and the Login Service:

```yaml
global:
  domain: auth.develop.eoepca.org
  nginxIp: 185.52.192.231
volumeClaim:
  name: eoepca-userman-pvc
  create: false
```

This service synchronizes user profile data from the LDAP directory (OpenDJ) and makes it available to other EOEPCA components that require user context.

**Sources:** [system/clusters/creodias/user-management/um-user-profile.yaml:1-24]()

## Policy Enforcement Integration

Policy Enforcement Points (PEPs) are integrated into EOEPCA services via Nginx Ingress annotations. The following example shows how a test service is protected:

```mermaid
sequenceDiagram
    participant Client
    participant Ingress as "Nginx Ingress"
    participant GK as "identity-gatekeeper.um.svc"
    participant Service as "Backend Service"
    
    Client->>Ingress: "GET /resource"
    Note over Ingress: "nginx.ingress.kubernetes.io/configuration-snippet:<br/>auth_request /auth"
    
    Ingress->>GK: "Internal auth_request /auth"
    Note over GK: "Proxy to identity-gatekeeper:3000"
    
    alt Token Valid
        GK-->>Ingress: "200 OK"
        Ingress->>Service: "Forward request"
        Service-->>Ingress: "Response"
        Ingress-->>Client: "200 + Data"
    else Token Invalid/Missing
        GK-->>Ingress: "401/403"
        Ingress-->>Client: "401/403 + WWW-Authenticate"
    end
```

**Nginx Configuration Snippet:**

The ingress configuration includes an internal authentication request to the `identity-gatekeeper` service:

```yaml
nginx.ingress.kubernetes.io/configuration-snippet: |
  auth_request /auth;
  # Preflighted requests
  if ($request_method = OPTIONS ) {
    return 200;
  }
  add_header Access-Control-Allow-Origin $http_origin always;
  add_header Access-Control-Allow-Methods "*";
  add_header Access-Control-Allow-Headers "Authorization, Origin, Content-Type";

nginx.ingress.kubernetes.io/server-snippet: |
  location ^~ /auth {
    internal;
    proxy_pass http://identity-gatekeeper.um.svc.cluster.local:3000/$request_uri;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Method $request_method;
    proxy_set_header X-Forwarded-URI $request_uri;
    proxy_busy_buffers_size 64k;
    proxy_buffers 8 32k;
    proxy_buffer_size 32k;
  }
```

This pattern enables transparent authentication enforcement at the ingress layer without modifying backend services.

**Sources:** [system/clusters/creodias/system/test/identity-dummy-service-ingress.yaml:9-31]()

## OIDC Client Integration

External services and portals integrate with the Identity Service using OIDC client credentials. The Django Portal example demonstrates this integration:

```mermaid
graph LR
    subgraph "Django Portal (demo namespace)"
        Portal["django-portal<br/>Pod"]
        Secrets["django-secrets<br/>SealedSecret"]
    end
    
    subgraph "OIDC Configuration"
        ClientID["OIDC_RP_CLIENT_ID<br/>(Gluu client)"]
        ClientSecret["OIDC_RP_CLIENT_SECRET"]
        KCClientID["KEYCLOAK_OIDC_RP_CLIENT_ID<br/>(Keycloak client)"]
        KCClientSecret["KEYCLOAK_OIDC_RP_CLIENT_SECRET"]
    end
    
    subgraph "Identity Providers"
        Gluu["oxauth<br/>auth.develop.eoepca.org"]
        Keycloak["identity-keycloak<br/>identity.keycloak.develop.eoepca.org"]
    end
    
    Secrets -->|"Inject"| ClientID
    Secrets -->|"Inject"| ClientSecret
    Secrets -->|"Inject"| KCClientID
    Secrets -->|"Inject"| KCClientSecret
    
    Portal -->|"Login via"| ClientID
    Portal -->|"Login via"| KCClientID
    
    ClientID -->|"Authenticate"| Gluu
    KCClientID -->|"Authenticate"| Keycloak
```

**Sealed Secret Structure:**

OIDC client credentials are stored as Kubernetes SealedSecrets for secure GitOps deployment. The secret includes:

| Key | Purpose |
|-----|---------|
| `OIDC_RP_CLIENT_ID` | Gluu/oxAuth client identifier |
| `OIDC_RP_CLIENT_SECRET` | Gluu/oxAuth client secret |
| `KEYCLOAK_OIDC_RP_CLIENT_ID` | Keycloak client identifier (alternative IdP) |
| `KEYCLOAK_OIDC_RP_CLIENT_SECRET` | Keycloak client secret |
| `DJANGO_SECRET` | Django application secret key |

**Secret Creation Script:**

The `ss-django-secrets-create.sh` script automates SealedSecret generation:

```bash
kubectl -n "${NAMESPACE}" create secret generic "${SECRET_NAME}" \
  --from-literal=OIDC_RP_CLIENT_ID="${client_id}" \
  --from-literal=OIDC_RP_CLIENT_SECRET="${client_secret}" \
  --from-literal=DJANGO_SECRET="${django_secret}" \
  --from-literal=KEYCLOAK_OIDC_RP_CLIENT_ID="${keycloak_client_id}" \
  --from-literal=KEYCLOAK_OIDC_RP_CLIENT_SECRET="${keycloak_client_secret}" \
  --dry-run=client -o yaml | \
kubeseal -o yaml --controller-name eoepca-sealed-secrets \
  --controller-namespace infra > ss-${SECRET_NAME}.yaml
```

**Sources:** [system/clusters/creodias/system/demo/ss-django-secrets.yaml:1-17](), [system/clusters/creodias/system/demo/ss-django-secrets-create.sh:22-33]()

## Portal Integration

Two demonstration portals are deployed to showcase authentication integration:

### EOEPCA Portal

A static portal providing an overview and access to EOEPCA services:

```yaml
configmap:
  configuration: develop
ingress:
  hosts:
    - host: eoepca-portal.develop.eoepca.org
      paths:
        - path: /
          pathType: Prefix
```

**Sources:** [system/clusters/creodias/system/demo/hr-eoepca-portal.yaml:20-31]()

### Django Portal

A dynamic Django-based portal with full OIDC authentication:

```yaml
domain: develop.eoepca.org
authHost: auth
configmap:
  user_prefix: develop-user
```

The Django Portal uses the `authHost: auth` configuration to connect to `auth.develop.eoepca.org` for authentication and retrieves OIDC client credentials from the `django-secrets` SealedSecret.

**Sources:** [system/clusters/creodias/system/demo/hr-django-portal.yaml:17-20]()

## Deployment Dependencies

The User Management building block has the following deployment dependencies:

```mermaid
graph TD
    subgraph "Prerequisites"
        NS["Namespace: um"]
        PVC["PersistentVolumeClaim:<br/>eoepca-userman-pvc"]
        NFS["NFS Server<br/>192.168.123.14"]
        StorageClass["StorageClass:<br/>eoepca-nfs"]
        SealedSecretCtrl["sealed-secrets<br/>(infra namespace)"]
        IngressCtrl["ingress-nginx<br/>(ingress-nginx namespace)"]
        CertMgr["cert-manager<br/>(cert-manager namespace)"]
    end
    
    subgraph "User Management Services"
        Login["um-login-service<br/>HelmRelease"]
        Identity["um-identity-service<br/>HelmRelease"]
        PDP["um-pdp-engine<br/>HelmRelease"]
        Profile["um-user-profile<br/>HelmRelease"]
    end
    
    NFS -->|"Provision"| StorageClass
    StorageClass -->|"Create"| PVC
    PVC -->|"Mount"| Login
    PVC -->|"Mount"| Identity
    PVC -->|"Mount"| PDP
    PVC -->|"Mount"| Profile
    
    IngressCtrl -->|"Route traffic to"| Login
    IngressCtrl -->|"Route traffic to"| Identity
    
    CertMgr -->|"Issue TLS certs for"| Login
    CertMgr -->|"Issue TLS certs for"| Identity
    
    Login -->|"Provides user store for"| Identity
    Identity -->|"Token validation for"| PDP
    Profile -->|"User metadata for"| PDP
```

**Storage Configuration:**

The NFS-backed storage is configured via the `storage` HelmRelease:

```yaml
nfs:
  server:
    address: "192.168.123.14"
domain:
  userman:
    storageClass: eoepca-nfs
```

This creates the `eoepca-nfs` StorageClass, which provisions PersistentVolumes from the NFS server for the `eoepca-userman-pvc` PersistentVolumeClaim.

**Sources:** [system/clusters/creodias/system/storage/hr-storage.yaml:18-28]()

## Helm Chart Versions

The User Management subsystem uses the following Helm chart versions (as of the latest deployment):

| HelmRelease | Chart Name | Version | Repository |
|-------------|------------|---------|------------|
| `um-identity-service` | `identity-service` | 1.0.75 | eoepca |
| `um-login-service` | `login-service` | 1.2.8 | eoepca |
| `um-pdp-engine` | `pdp-engine` | 1.1.12 | eoepca |
| `um-user-profile` | `user-profile` | 1.1.12 | eoepca |

All charts reference the `eoepca` HelmRepository in the `common` namespace, which is managed by Flux CD.

**Sources:** [system/clusters/creodias/user-management/um-identity-service.yaml:10-17](), [system/clusters/creodias/user-management/um-login-service.yaml:7-14](), [system/clusters/creodias/user-management/um-pdp-engine.yaml:7-14](), [system/clusters/creodias/user-management/um-user-profile.yaml:7-14]()

## Timeout Configuration

User Management services have extended timeout configurations to accommodate initialization and startup:

| Service | Timeout | Reason |
|---------|---------|--------|
| `um-identity-service` | 5m | Keycloak and PostgreSQL initialization |
| `um-login-service` | 25m | Gluu components require extended startup (OpenDJ, oxAuth, oxTrust) |
| `um-pdp-engine` | 25m | Policy loading and identity service synchronization |
| `um-user-profile` | 25m | LDAP synchronization and profile initialization |

These timeouts are configured in the HelmRelease specifications:

```yaml
timeout: 25m0s
interval: 1m0s
```

**Sources:** [system/clusters/creodias/user-management/um-login-service.yaml:67-68](), [system/clusters/creodias/user-management/um-pdp-engine.yaml:26-27](), [system/clusters/creodias/user-management/um-user-profile.yaml:22-23]()

## Summary

The User Management and Identity building block provides a comprehensive authentication and authorization infrastructure for the EOEPCA platform. Key characteristics include:

- **Dual Identity Providers**: Gluu (Login Service) for user authentication and Keycloak (Identity Service) for UMA resource management
- **Centralized Policy Enforcement**: PDP Engine makes authorization decisions based on XACML policies
- **Flexible Integration**: PEPs can be deployed via Ingress annotations or as dedicated resource-guard instances
- **Shared Storage**: All components use the `eoepca-userman-pvc` for persistent state
- **GitOps Deployment**: Flux CD manages continuous deployment with HelmRelease specifications
- **Sealed Secrets**: OIDC client credentials are encrypted for secure storage in Git

For implementation details of specific subsystems, refer to the child pages: [Identity Service (Keycloak)](#4.1), [Login Service (Gluu)](#4.2), [Policy Enforcement (PEP/PDP)](#4.3), and [UMA Authentication Flow](#4.4).