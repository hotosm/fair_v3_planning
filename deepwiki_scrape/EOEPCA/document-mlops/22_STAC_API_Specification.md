# STAC API Specification

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/api/endpoint-specification.md](docs/api/endpoint-specification.md)

</details>



## Purpose and Scope

This document provides the technical specification for the SharingHub STAC (SpatioTemporal Asset Catalog) API. The STAC API is the primary interface for programmatically discovering and accessing AI models, datasets, and processors registered in the MLOps platform. It implements the STAC specification standard to provide a standardized catalog interface for Earth Observation and machine learning assets.

This page covers the API specification, including endpoints, data models, extensions, and authentication mechanisms. For practical examples of using the STAC API with client libraries, see [Using the STAC API](#7.2). For broader context on the SharingHub component that exposes this API, see [SharingHub](#3.2).

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## STAC Standard Overview

The STAC API exposed by SharingHub implements the STAC specification, which provides a common language to describe geospatial information and assets. In the MLOps context, STAC serves as a standardized discovery layer that:

- Organizes assets into a hierarchical catalog structure (Catalog → Collections → Items)
- Supports extensibility through STAC extensions (ml-model, eo, etc.)
- Enables filtering and search capabilities
- Provides standardized asset links for downloading models and datasets
- Maintains compatibility with existing STAC client tools and libraries

The SharingHub implementation dynamically generates the STAC catalog from GitLab projects, using GitLab topics to determine collection membership and project metadata to populate STAC item properties.

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## API Endpoint Structure

### Core STAC Endpoints

The SharingHub STAC API is exposed at the base path `/stac` and implements the following standard STAC endpoints:

**Diagram: STAC API Endpoint Hierarchy**

```mermaid
graph TB
    BASE["/stac"]
    ROOT["/stac/"]
    CATALOG["GET /stac/<br/>Root Catalog"]
    COLLECTIONS["GET /stac/collections<br/>List all Collections"]
    COLLECTION["GET /stac/collections/{collection_id}<br/>Get Collection by ID"]
    ITEMS["GET /stac/collections/{collection_id}/items<br/>List Items in Collection"]
    ITEM["GET /stac/collections/{collection_id}/items/{item_id}<br/>Get Item by ID"]
    OPENAPI["GET /openapi.json<br/>OpenAPI Specification"]
    
    BASE --> CATALOG
    BASE --> COLLECTIONS
    BASE --> OPENAPI
    COLLECTIONS --> COLLECTION
    COLLECTION --> ITEMS
    ITEMS --> ITEM
    
    CATALOG -.returns.-> ROOT_RESP["STAC Catalog Object<br/>id: gitlab-cs<br/>links to collections"]
    COLLECTIONS -.returns.-> COLL_LIST["Array of STAC Collection Objects"]
    COLLECTION -.returns.-> COLL_OBJ["STAC Collection Object<br/>with extent, keywords"]
    ITEMS -.returns.-> ITEMS_LIST["Array of STAC Item Objects"]
    ITEM -.returns.-> ITEM_OBJ["STAC Item Object<br/>with assets, properties, extensions"]
```

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### Endpoint Details

| Endpoint | Method | Purpose | Response Type |
|----------|--------|---------|---------------|
| `/stac/` | GET | Retrieve root STAC catalog | STAC Catalog JSON |
| `/stac/collections` | GET | List all available collections | Array of STAC Collections |
| `/stac/collections/{collection_id}` | GET | Get specific collection details | STAC Collection JSON |
| `/stac/collections/{collection_id}/items` | GET | List items in a collection | Array of STAC Items |
| `/stac/collections/{collection_id}/items/{item_id}` | GET | Get specific item details | STAC Item JSON |
| `/openapi.json` | GET | OpenAPI specification document | OpenAPI 3.0 JSON |

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## Data Models

### STAC Object Hierarchy

The STAC API organizes content in a three-tier hierarchy that maps directly to GitLab project organization:

**Diagram: STAC Data Model Hierarchy and GitLab Mapping**

```mermaid
graph TB
    subgraph "STAC Layer"
        CATALOG["STAC Catalog<br/>type: Catalog<br/>id: gitlab-cs"]
        COLLECTION1["STAC Collection<br/>type: Collection<br/>id: aimodel<br/>keywords: [AI, ML]"]
        COLLECTION2["STAC Collection<br/>type: Collection<br/>id: dataset<br/>keywords: [Data, EO]"]
        COLLECTION3["STAC Collection<br/>type: Collection<br/>id: processor<br/>keywords: [Processing]"]
        ITEM1["STAC Item<br/>type: Feature<br/>id: project-123<br/>extensions: [ml-model]"]
        ITEM2["STAC Item<br/>type: Feature<br/>id: project-456<br/>extensions: [eo]"]
        ITEM3["STAC Item<br/>type: Feature<br/>id: project-789<br/>extensions: [ml-model]"]
    end
    
    subgraph "GitLab Layer"
        GITLAB["GitLab Instance"]
        TOPIC1["Projects with<br/>sharinghub:aimodel topic"]
        TOPIC2["Projects with<br/>sharinghub:dataset topic"]
        TOPIC3["Projects with<br/>sharinghub:processor topic"]
        PROJECT1["GitLab Project<br/>ID: 123<br/>flood-detection"]
        PROJECT2["GitLab Project<br/>ID: 456<br/>sen1floods11-dataset"]
        PROJECT3["GitLab Project<br/>ID: 789<br/>wine-quality-model"]
    end
    
    CATALOG --> COLLECTION1
    CATALOG --> COLLECTION2
    CATALOG --> COLLECTION3
    
    COLLECTION1 --> ITEM1
    COLLECTION1 --> ITEM3
    COLLECTION2 --> ITEM2
    COLLECTION3
    
    GITLAB --> TOPIC1
    GITLAB --> TOPIC2
    GITLAB --> TOPIC3
    
    TOPIC1 --> PROJECT1
    TOPIC1 --> PROJECT3
    TOPIC2 --> PROJECT2
    
    PROJECT1 -.mapped to.-> ITEM1
    PROJECT2 -.mapped to.-> ITEM2
    PROJECT3 -.mapped to.-> ITEM3
```

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### STAC Catalog Object

The root catalog object represents the entire SharingHub catalog:

```json
{
  "type": "Catalog",
  "id": "gitlab-cs",
  "stac_version": "1.0.0",
  "description": "STAC Catalog for GitLab projects",
  "links": [
    {
      "rel": "self",
      "href": "/stac/",
      "type": "application/json"
    },
    {
      "rel": "root",
      "href": "/stac/",
      "type": "application/json"
    },
    {
      "rel": "child",
      "href": "/stac/collections/aimodel",
      "type": "application/json"
    }
  ]
}
```

**Key Properties:**
- `id`: Fixed identifier `gitlab-cs`
- `stac_version`: STAC specification version (1.0.0)
- `links`: Array of link objects connecting to collections

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### STAC Collection Object

Collections represent categories of assets (AI models, datasets, processors):

```json
{
  "type": "Collection",
  "id": "aimodel",
  "stac_version": "1.0.0",
  "title": "AI Model",
  "description": "AI Models and Machine Learning assets",
  "keywords": ["AI", "Machine Learning", "Model"],
  "license": "various",
  "extent": {
    "spatial": {
      "bbox": [[-180, -90, 180, 90]]
    },
    "temporal": {
      "interval": [[null, null]]
    }
  },
  "links": [
    {
      "rel": "self",
      "href": "/stac/collections/aimodel",
      "type": "application/json"
    },
    {
      "rel": "root",
      "href": "/stac/",
      "type": "application/json"
    },
    {
      "rel": "items",
      "href": "/stac/collections/aimodel/items",
      "type": "application/json"
    }
  ]
}
```

**Key Properties:**
- `id`: Collection identifier mapped from GitLab topic (e.g., `aimodel`, `dataset`, `processor`)
- `keywords`: Tags and categories for filtering
- `extent`: Spatial and temporal extent of collection items
- `links`: Navigation links to items and parent catalog

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### STAC Item Object

Items represent individual GitLab projects as discoverable assets:

```json
{
  "type": "Feature",
  "stac_version": "1.0.0",
  "id": "123",
  "collection": "aimodel",
  "geometry": null,
  "bbox": null,
  "properties": {
    "datetime": "2024-01-15T10:30:00Z",
    "title": "Flood Detection Model",
    "description": "Deep learning model for flood detection",
    "gitlab:project_id": 123,
    "gitlab:namespace": "ml-models",
    "gitlab:web_url": "https://gitlab.domain/ml-models/flood-detection",
    "created": "2024-01-10T08:00:00Z",
    "updated": "2024-01-15T10:30:00Z"
  },
  "assets": {
    "model": {
      "href": "https://sharinghub.domain/mlflow/models/flood-model.onnx",
      "type": "application/octet-stream",
      "roles": ["ml-model:checkpoint"],
      "title": "ONNX Model File"
    },
    "thumbnail": {
      "href": "https://gitlab.domain/ml-models/flood-detection/-/avatar",
      "type": "image/png",
      "roles": ["thumbnail"]
    }
  },
  "links": [
    {
      "rel": "self",
      "href": "/stac/collections/aimodel/items/123",
      "type": "application/json"
    },
    {
      "rel": "parent",
      "href": "/stac/collections/aimodel",
      "type": "application/json"
    },
    {
      "rel": "collection",
      "href": "/stac/collections/aimodel",
      "type": "application/json"
    }
  ],
  "stac_extensions": [
    "https://stac-extensions.github.io/ml-model/v1.0.0/schema.json"
  ]
}
```

**Key Properties:**
- `id`: GitLab project ID as string
- `collection`: Parent collection identifier
- `properties`: Metadata extracted from GitLab project
- `assets`: Downloadable files (models, datasets, documentation)
- `stac_extensions`: Array of STAC extension URIs applied to this item

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## STAC Extensions

SharingHub implements STAC extensions to provide domain-specific metadata:

**Diagram: STAC Extension Usage by Collection Type**

```mermaid
graph LR
    subgraph "STAC Item Types"
        AIMODEL["AI Model Items<br/>collection: aimodel"]
        DATASET["Dataset Items<br/>collection: dataset"]
        PROCESSOR["Processor Items<br/>collection: processor"]
    end
    
    subgraph "STAC Extensions"
        MLEXT["ml-model Extension<br/>v1.0.0"]
        EOEXT["eo Extension<br/>v1.1.0"]
        BASEEXT["Base STAC<br/>v1.0.0"]
    end
    
    subgraph "Extension Properties"
        MLPROPS["ml-model:architecture<br/>ml-model:learning_approach<br/>ml-model:prediction_type"]
        EOPROPS["eo:bands<br/>eo:cloud_cover<br/>eo:gsd"]
        BASEPROPS["datetime<br/>title<br/>description"]
    end
    
    AIMODEL --> MLEXT
    DATASET --> EOEXT
    PROCESSOR --> BASEEXT
    
    MLEXT --> MLPROPS
    EOEXT --> EOPROPS
    BASEEXT --> BASEPROPS
```

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### ml-model Extension

Applied to AI model items (collection `aimodel`):

| Property | Type | Description |
|----------|------|-------------|
| `ml-model:architecture` | string | Model architecture (e.g., "resnet50", "unet") |
| `ml-model:learning_approach` | string | Learning type ("supervised", "unsupervised") |
| `ml-model:prediction_type` | string | Task type ("classification", "segmentation") |
| `ml-model:framework` | string | Framework used (e.g., "pytorch", "tensorflow") |
| `ml-model:framework_version` | string | Framework version |
| `ml-model:memory_size` | integer | Model memory footprint in bytes |

### eo Extension

Applied to Earth Observation dataset items (collection `dataset`):

| Property | Type | Description |
|----------|------|-------------|
| `eo:bands` | array | Spectral bands included in dataset |
| `eo:cloud_cover` | number | Cloud cover percentage (0-100) |
| `eo:gsd` | number | Ground sample distance in meters |

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## Authentication and Authorization

### Authentication Flow

**Diagram: STAC API Authentication Request Flow**

```mermaid
sequenceDiagram
    participant Client
    participant NGINX["NGINX Ingress"]
    participant SharingHub["SharingHub STAC API"]
    participant GitLab["GitLab API"]
    participant KeyCloak["KeyCloak OIDC Provider"]
    
    Client->>NGINX: GET /stac/collections/aimodel/items/123<br/>Authorization: Bearer <token>
    NGINX->>SharingHub: Forward request with token
    
    alt Token Provided
        SharingHub->>SharingHub: Validate Bearer token format
        SharingHub->>GitLab: GET /api/v4/projects/123<br/>Authorization: Bearer <token>
        GitLab->>KeyCloak: Validate OIDC token
        KeyCloak-->>GitLab: Token valid/invalid
        GitLab-->>SharingHub: Project details + access level
        
        alt User has access
            SharingHub->>SharingHub: Generate STAC Item with assets
            SharingHub-->>Client: 200 OK + STAC Item JSON
        else User lacks access
            SharingHub-->>Client: 403 Forbidden
        end
    else No Token
        SharingHub->>GitLab: GET /api/v4/projects/123<br/>Use default token (if configured)
        GitLab-->>SharingHub: Project details (public only)
        
        alt Project is public
            SharingHub-->>Client: 200 OK + STAC Item JSON
        else Project is private
            SharingHub-->>Client: 403 Forbidden
        end
    end
```

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### Authentication Methods

The STAC API supports two authentication modes:

#### Bearer Token Authentication

Authenticated requests include a Bearer token in the Authorization header:

```http
GET /stac/collections/aimodel/items/123 HTTP/1.1
Host: sharinghub.domain
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

The token is validated against GitLab's OIDC integration with KeyCloak. Users can only access STAC items corresponding to GitLab projects they have permission to view.

#### Unauthenticated Access

If the `sharinghub-oidc` secret contains a `default-token`, unauthenticated requests are allowed for public projects:

```http
GET /stac/collections/aimodel/items HTTP/1.1
Host: sharinghub.domain
```

This enables public discovery while maintaining security for private projects. Private projects return `403 Forbidden` for unauthenticated requests.

**Sources:** [docs/api/endpoint-specification.md:1-9]()

### Permission Mapping

| GitLab Permission Level | STAC API Access |
|------------------------|-----------------|
| Guest | Read-only access to STAC item metadata |
| Reporter | Read-only access to STAC item metadata and assets |
| Developer | Read-only access to STAC item metadata and assets |
| Maintainer | Read-only access to STAC item metadata and assets |
| Owner | Read-only access to STAC item metadata and assets |
| No Access (Private) | 403 Forbidden |
| Public Project | Read-only access (if default token configured) |

The STAC API is read-only; all write operations (creating projects, uploading models) occur through GitLab and MLflow interfaces.

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## OpenAPI Specification

The complete OpenAPI 3.0 specification for the SharingHub STAC API is available at:

```
GET /openapi.json
```

This machine-readable specification includes:

- All endpoint definitions with request/response schemas
- Authentication requirements (Bearer token)
- Data model schemas for Catalog, Collection, Item objects
- Extension schemas (ml-model, eo)
- Example requests and responses
- Error response formats

The specification can be consumed by:
- **Swagger UI**: Interactive API documentation and testing interface (available at `https://sharinghub.develop.eoepca.org/openapi.json`)
- **OpenAPI Generators**: Auto-generate client libraries in multiple languages
- **API Testing Tools**: Postman, Insomnia, curl commands
- **STAC Clients**: pystac-client and other STAC-compliant tools

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## Error Responses

The STAC API returns standard HTTP status codes with JSON error bodies:

```json
{
  "code": 404,
  "type": "ItemNotFound",
  "description": "Item with id '999' not found in collection 'aimodel'"
}
```

**Common Error Codes:**

| Status Code | Error Type | Description |
|-------------|------------|-------------|
| 400 | BadRequest | Invalid request parameters or malformed request |
| 401 | Unauthorized | Missing or invalid Bearer token |
| 403 | Forbidden | Valid token but insufficient permissions |
| 404 | NotFound | Collection or item does not exist |
| 500 | InternalServerError | Server-side error (check SharingHub logs) |
| 502 | BadGateway | GitLab API unreachable |
| 503 | ServiceUnavailable | SharingHub temporarily unavailable |

**Sources:** [docs/api/endpoint-specification.md:1-9]()

## Query Parameters

The STAC API supports standard query parameters for filtering and pagination:

### Collection Items Endpoint

```
GET /stac/collections/{collection_id}/items?limit=10&bbox=-180,-90,180,90
```

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `limit` | integer | Maximum items to return (default: 100) | `limit=50` |
| `bbox` | array | Bounding box filter [west, south, east, north] | `bbox=-10,40,10,50` |
| `datetime` | string | Temporal filter (ISO 8601) | `datetime=2024-01-01/2024-12-31` |
| `ids` | array | Filter by specific item IDs | `ids=123,456,789` |

### Link Relation Types

STAC items and collections use standard link relation types:

| Relation | Description |
|----------|-------------|
| `self` | Canonical URL for this resource |
| `root` | Link to root catalog |
| `parent` | Link to parent collection (for items) |
| `child` | Link to child collection (for catalogs) |
| `item` | Link to item (for collections) |
| `collection` | Link to parent collection (for items) |

**Sources:** [docs/api/endpoint-specification.md:1-9]()