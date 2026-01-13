# Data Access Services

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

The Data Access Services component provides OGC-compliant visualization and access capabilities for Earth Observation data within the EOEPCA platform. This service enables users to view, query, and retrieve satellite imagery through standardized OGC WMS (Web Map Service), WCS (Web Coverage Service), and WMTS (Web Map Tile Service) interfaces. The service is built on the EOX ViewServer framework and integrates with S3 storage backends, the Resource Catalogue, and supports both global and workspace-specific deployments.

For information about data cataloging and metadata management, see [Resource Catalogue](#5.2). For workspace provisioning and multi-tenant isolation, see [Multi-Tenant Workspaces](#5.5). For data registration workflows, see [Data Registration and Harvesting](#5.4).

## Architecture Overview

The Data Access service consists of multiple cooperating components that handle data rendering, registration, harvesting, caching, and client visualization.

**Architecture: Data Access Service Components**

```mermaid
graph TB
    subgraph "External Clients"
        OGC["OGC Clients<br/>(WMS/WCS/WMTS)"]
        WebUI["Web UI Client"]
    end
    
    subgraph "Data Access Service (vs)"
        Renderer["vs.renderer<br/>4 replicas<br/>OGC service endpoint"]
        Registrar["vs.registrar<br/>Registration worker"]
        Harvester["vs.harvester<br/>OpenSearch harvester"]
        Client["vs.client<br/>Web visualization UI"]
        Scheduler["vs.scheduler<br/>Background task scheduler"]
        Seeder["vs.seeder<br/>Tile pre-generation"]
        Cache["vs.cache<br/>Tile cache manager"]
        Redis[("Redis<br/>register_queue<br/>seed_queue")]
    end
    
    subgraph "Storage Backends"
        S3Data["S3 Data Storage<br/>CloudFerro eodata<br/>endpoint: data.cloudferro.com"]
        S3Cache["S3 Cache Storage<br/>cf2.cloudferro.com:8080<br/>bucket: cache-bucket"]
        S3Workspace["S3 Workspace Storage<br/>MinIO<br/>minio.develop.eoepca.org"]
    end
    
    subgraph "Resource Catalogue"
        PyCSW["pycsw Database<br/>postgresql://...@resource-catalogue-db"]
    end
    
    subgraph "External Data Sources"
        CreoDIAS["CreoDIAS OpenSearch<br/>datahub.creodias.eu"]
    end
    
    OGC -->|GetCapabilities<br/>GetMap<br/>GetCoverage| Renderer
    WebUI --> Client
    Client -->|HTTP| Renderer
    
    Harvester -->|Poll| CreoDIAS
    Harvester -->|Enqueue items| Redis
    Redis -->|Consume| Registrar
    Registrar -->|Write metadata| PyCSW
    Registrar -->|Trigger seeding| Redis
    Redis -->|Consume seed_queue| Seeder
    
    Renderer -->|Read data| S3Data
    Renderer -->|Read data| S3Workspace
    Renderer -->|Query metadata| PyCSW
    Renderer -->|Store tiles| S3Cache
    
    Scheduler -->|Schedule tasks| Harvester
    Scheduler -->|Schedule tasks| Seeder
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:1-1144]()

## Component Breakdown

### Renderer

The renderer component is the primary OGC service endpoint, handling WMS, WCS, and WMTS requests. It runs with 4 replicas for high availability.

| Property | Value |
|----------|-------|
| Component | `vs.renderer` |
| Replica Count | 4 |
| CPU Requests | 0.5 cores |
| Memory Requests | 1 GiB |
| CPU Limits | 1.5 cores |
| Memory Limits | 3 GiB |
| Ingress Host | `data-access.develop.eoepca.org` |

**Configuration:**
- Reads data from S3 storage backends using GDAL virtual file system (`CPL_VSIL_CURL_ALLOWED_EXTENSIONS`)
- Queries metadata from Resource Catalogue PostgreSQL database
- Generates dynamic visualizations based on product type browse configurations
- Supports multiple grids and zoom levels

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:864-876]()

### Registrar

The registrar is a worker process that consumes items from Redis queues and registers them in both the EOxServer instance and the Resource Catalogue.

**Queue Routes:**

```mermaid
graph LR
    subgraph "Redis Queues"
        RegQueue["register_queue<br/>(items)"]
        CollQueue["register_collection_queue<br/>(collections)"]
        ADESQueue["register_ades_queue<br/>(ADES instances)"]
        AppQueue["register_application_queue<br/>(CWL apps)"]
        CatQueue["register_catalogue_queue<br/>(catalogues)"]
        JSONQueue["register_json_queue<br/>(JSON metadata)"]
        XMLQueue["register_xml_queue<br/>(XML metadata)"]
    end
    
    subgraph "Registrar Routes"
        ItemRoute["registrar.route.stac.ItemRoute"]
        CollRoute["registrar.route.stac.CollectionRoute"]
        ADESRoute["registrar.route.json.JSONRoute"]
        AppRoute["registrar.route.json.JSONRoute"]
        CatRoute["registrar.route.json.JSONRoute"]
        JSONRoute["registrar.route.json.JSONRoute"]
        XMLRoute["registrar.route.json.JSONRoute"]
    end
    
    subgraph "Backends"
        ItemBackend["registrar_pycsw.backend.ItemBackend"]
        CollBackend["registrar_pycsw.backend.CollectionBackend"]
        ADESBackend["registrar_pycsw.backend.ADESBackend"]
        CWLBackend["registrar_pycsw.backend.CWLBackend"]
        CatBackend["registrar_pycsw.backend.CatalogueBackend"]
        JSONBackend["registrar_pycsw.backend.JSONBackend"]
        XMLBackend["registrar_pycsw.backend.XMLBackend"]
    end
    
    RegQueue --> ItemRoute
    CollQueue --> CollRoute
    ADESQueue --> ADESRoute
    AppQueue --> AppRoute
    CatQueue --> CatRoute
    JSONQueue --> JSONRoute
    XMLQueue --> XMLRoute
    
    ItemRoute --> ItemBackend
    CollRoute --> CollBackend
    ADESRoute --> ADESBackend
    AppRoute --> CWLBackend
    CatRoute --> CatBackend
    JSONRoute --> JSONBackend
    XMLRoute --> XMLBackend
```

**Backend Configuration:**

| Route | Queue | Backend Class | Purpose |
|-------|-------|---------------|---------|
| items | `register` | `registrar_pycsw.backend.ItemBackend` | Register STAC items |
| collections | `register_collection_queue` | `registrar_pycsw.backend.CollectionBackend` | Register STAC collections |
| ades | `register_ades_queue` | `registrar_pycsw.backend.ADESBackend` | Register ADES instances |
| application | `register_application_queue` | `registrar_pycsw.backend.CWLBackend` | Register CWL applications |
| catalogue | `register_catalogue_queue` | `registrar_pycsw.backend.CatalogueBackend` | Register catalogue metadata |
| json | `register_json_queue` | `registrar_pycsw.backend.JSONBackend` | Register generic JSON |
| xml | `register_xml_queue` | `registrar_pycsw.backend.XMLBackend` | Register generic XML |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:878-948]()

### Harvester

The harvester polls external OpenSearch endpoints to discover new data products and enqueues them for registration.

**Configured Harvesters:**

| Harvester Name | Collection | Product Type | OpenSearch URL |
|----------------|------------|--------------|----------------|
| Sentinel2 | Sentinel2 | S2MSI1C, S2MSI2A | `datahub.creodias.eu/resto/api/collections/Sentinel2` |
| Landsat8 | Landsat8 | L8MSI1TP, L8MSI1GT | `datahub.creodias.eu/resto/api/collections/Landsat8` |
| Sentinel1-GRD | Sentinel1 | S1IWGRD1C_VVVH | `datahub.creodias.eu/resto/api/collections/Sentinel1` (GRD-COG) |
| Sentinel1-SLC | Sentinel1 | S1SLC | `datahub.creodias.eu/resto/api/collections/Sentinel1` (SLC) |
| Sentinel3 | Sentinel3 | S3A_OL_2_LFR | `datahub.creodias.eu/resto/api/collections/Sentinel3` (OL_2_LFR___) |

**Harvester Configuration Example:**

```yaml
Sentinel2:
  resource:
    type: OpenSearch
    opensearch:
      url: https://datahub.creodias.eu/resto/api/collections/Sentinel2/describe.xml
      format:
        type: 'application/json'
        json:
          property_mapping:
            start_datetime: 'startDate'
            end_datetime: 'completionDate'
            productIdentifier: 'productIdentifier'
      query:
        time:
          begin: 2019-09-10T00:00:00Z
          end: 2019-09-11T00:00:00Z
        collection: null
        bbox: 14.9,47.7,16.4,48.7
  postprocessors:
    - type: external
      process: harvester_eoepca.postprocess.postprocess_sentinel2
      kwargs: {}
  queue: register
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:949-1087]()

### Client

The client is a web-based visualization interface for browsing and viewing data layers.

**Configuration:**
- Service URL: `https://data-access.develop.eoepca.org/`
- EOxServer download enabled
- Time domain configuration for temporal filtering
- Custom date ranges for display and selection

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:1089-1107]()

### Scheduler

The scheduler manages periodic background tasks such as harvesting cycles and cache seeding operations.

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:1136-1144]()

### Seeder

The seeder pre-generates map tiles for faster access. It is triggered by items in the `seed_queue` after successful registration.

**Sources:** Configuration references in registrar `defaultSuccessQueue: seed_queue` [system/clusters/creodias/resource-management/hr-data-access.yaml:893]()

### Cache

The cache component stores pre-rendered tiles in S3 storage to reduce rendering load and improve response times.

**Cache Storage Configuration:**

| Property | Value |
|----------|-------|
| Type | S3 |
| Endpoint | `https://cf2.cloudferro.com:8080/cache-bucket` |
| Host | `cf2.cloudferro.com:8080` |
| Region | RegionOne |
| Bucket | `cache-bucket` |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:58-64]()

## OGC Service Interfaces

The Data Access service exposes OGC-compliant REST endpoints through the renderer component.

**Service Metadata Configuration**

```mermaid
graph TB
    subgraph "OGC Service Metadata"
        Title["title: EOEPCA Data Access Service<br/>developed by EOX"]
        Abstract["abstract: EOEPCA Data Access Service<br/>developed by EOX"]
        URL["url: https://ecma/cache/ows"]
        Contact["contactOrganization: EOX IT Services GmbH<br/>contactEmail: office@eox.at<br/>contactPosition: CTO"]
        Provider["providerName: EOX<br/>providerUrl: https://eox.at"]
        INSPIRE["inspireProfile: true<br/>defaultLanguage: eng"]
    end
    
    subgraph "Supported Operations"
        WMS["WMS Operations<br/>GetCapabilities<br/>GetMap<br/>GetFeatureInfo"]
        WCS["WCS Operations<br/>GetCapabilities<br/>DescribeCoverage<br/>GetCoverage"]
        WMTS["WMTS Operations<br/>GetCapabilities<br/>GetTile"]
    end
    
    Title --> WMS
    Abstract --> WCS
    Contact --> WMTS
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:66-89]()

## Data Model

The Data Access service uses a hierarchical model: **Product Types** → **Collections** → **Layers** → **Coverage Types**.

### Product Types

Product types define how STAC items are mapped to coverages and browses (visualizations).

**Product Type Structure:**

```mermaid
graph TB
    subgraph "Product Type: S2MSI2A"
        Filter["filter:<br/>s2:product_type: S2MSI2A"]
        Collections["collections:<br/>- S2L2A"]
        Coverages["coverages:<br/>S2L2A_B01, S2L2A_B02, ...<br/>S2L2A_B12"]
        Browses["browses:<br/>TRUE_COLOR<br/>FALSE_COLOR<br/>NDVI"]
        Masks["masks:<br/>clouds (validity: false)"]
    end
    
    Filter --> Collections
    Collections --> Coverages
    Collections --> Browses
    Collections --> Masks
    
    subgraph "Browse: TRUE_COLOR"
        Red["red: B04<br/>range: [0, 4000]<br/>nodata: 0"]
        Green["green: B03<br/>range: [0, 4000]<br/>nodata: 0"]
        Blue["blue: B02<br/>range: [0, 4000]<br/>nodata: 0"]
        Asset["asset: visual-10m"]
    end
    
    Browses --> Red
    Browses --> Green
    Browses --> Blue
    Browses --> Asset
```

**Example Product Types:**

| Product Type | Filter | Collections | Coverages | Browses |
|--------------|--------|-------------|-----------|---------|
| S2MSI1C | `s2:product_type: S2MSI1C` | S2L1C | 13 bands (B01-B12) | TRUE_COLOR, FALSE_COLOR, NDVI |
| S2MSI2A | `s2:product_type: S2MSI2A` | S2L2A | 12 bands (B01-B12, no B10) | TRUE_COLOR, FALSE_COLOR, NDVI |
| L8MSI1TP | `platform: landsat-8`<br/>`landsat:correction: L1TP` | L8L1TP | 7 bands (B01-B07) | TRUE_COLOR |
| S1IWGRD1C_VVVH | `constellation: sentinel-1`<br/>`sar:instrument_mode: IW`<br/>`sar:product_type: GRD`<br/>`sar:polarizations: ["VV", "VH"]` | S1IWGRD1C | VV, VH | COMPOSITE |
| S3A_OL_2_LFR | `constellation: Sentinel-3`<br/>`s3:productType: OL_2_LFR___` | S3A_OL_2_LFR | (empty) | THUMBNAIL |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:510-862]()

### Collections

Collections group product types for organization and discovery.

**Collection Configuration:**

```yaml
collections:
  S2L1C:
    product_types:
      - S2MSI1C
    coverage_types:
      - S2L1C_B01
      - S2L1C_B02
      # ... all bands
      
  S2L2A:
    product_types:
      - S2MSI2A
    product_levels:
      - Level-2A
    coverage_types:
      - S2L2A_B01
      - S2L2A_B02
      # ... all bands
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:217-290]()

### Layers

Layers define how collections are presented in WMS/WMTS services.

**Layer Properties:**

| Property | Description | Example |
|----------|-------------|---------|
| `id` | Layer identifier | `S2L2A__TRUE_COLOR` |
| `title` | Human-readable title | "Sentinel-2 Level 2A True Color" |
| `abstract` | Description | "Sentinel-2 Level 2A True Color" |
| `displayColor` | Color for UI display | `#eb3700` |
| `grids` | Supported grids and zoom levels | `WGS84`, zoom: 13 |
| `parentLayer` | Parent collection | `S2L2A` |
| `search` | Search configuration | histogram bin count, threshold |

**Example Layer Configuration:**

```yaml
layers:
  - id: S2L2A
    title: Sentinel-2 Level 2A True Color
    abstract: Sentinel-2 Level 2A True Color
    displayColor: '#eb3700'
    grids:
      - name: WGS84
        zoom: 13
    parentLayer: S2L2A
    search:
      histogramBinCount: 15
      histogramThreshold: 80
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:91-216]()

### Coverage Types

Coverage types define the band metadata for individual coverages.

**Coverage Type Structure:**

| Field | Description |
|-------|-------------|
| `name` | Coverage type identifier |
| `data_type` | Data type (e.g., Uint16) |
| `bands` | Array of band definitions |
| `bands[].identifier` | Band identifier |
| `bands[].name` | Band common name |
| `bands[].definition` | OGC definition URI |
| `bands[].description` | Band description |
| `bands[].nil_values` | No-data values |
| `bands[].uom` | Unit of measure |
| `bands[].wavelength` | Central wavelength |

**Example: Landsat-8 Red Band:**

```yaml
- name: "L8L1TP_B04"
  data_type: "Uint16"
  bands:
    - identifier: "SR_B4"
      name: "red"
      definition: "http://www.opengis.net/def/property/OGC/0/Radiance"
      description: "SR_B4"
      nil_values:
        - reason: "http://www.opengis.net/def/nil/OGC/0/unknown"
          value: 0
      uom: "W/m2/um"
      wavelength: 0.65
```

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:291-509]()

## Storage Configuration

The Data Access service integrates with multiple S3 storage backends for data and cache.

**Storage Backends Configuration**

```mermaid
graph LR
    subgraph "Storage Backends"
        DataS3["Data Storage<br/>type: S3<br/>endpoint: data.cloudferro.com<br/>region: RegionOne<br/>validate_bucket_name: false"]
        CacheS3["Cache Storage<br/>type: S3<br/>endpoint: cf2.cloudferro.com:8080<br/>bucket: cache-bucket<br/>region: RegionOne"]
        WorkspaceS3["Workspace Storage<br/>type: S3<br/>endpoint: minio.develop.eoepca.org<br/>bucket: (user-specific)<br/>region: RegionOne"]
    end
    
    subgraph "Environment Variables"
        AWSURL["AWS_ENDPOINT_URL_S3:<br/>minio.develop.eoepca.org"]
        AWSHTTPS["AWS_HTTPS: FALSE"]
        VSILEXT["CPL_VSIL_CURL_ALLOWED_EXTENSIONS:<br/>.TIF,.tif,.xml,.jp2,.jpg,.png,.nc"]
    end
    
    DataS3 -.-> VSILEXT
    WorkspaceS3 -.-> AWSURL
    WorkspaceS3 -.-> AWSHTTPS
```

**Global Storage Configuration:**

| Storage Type | Endpoint | Bucket | Region | Purpose |
|--------------|----------|--------|--------|---------|
| data.data | `http://data.cloudferro.com` | (auto-detected) | RegionOne | Source EO data |
| cache | `https://cf2.cloudferro.com:8080` | cache-bucket | RegionOne | Rendered tile cache |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:49-64]()

## Data Registration Pipeline

The registration pipeline processes incoming data items and makes them available through the OGC services.

**Registration Flow**

```mermaid
sequenceDiagram
    participant H as Harvester
    participant R as Redis
    participant Reg as Registrar
    participant EOX as EOxServer Backend
    participant PY as pycsw Backend
    participant SQ as seed_queue
    participant Seed as Seeder
    
    H->>R: Enqueue STAC Item<br/>(register_queue)
    R->>Reg: Consume from register_queue
    Reg->>Reg: Route via registrar.route.stac.ItemRoute
    Reg->>EOX: registrar.backend.eoxserver.ItemBackend<br/>Create coverage in EOxServer
    Reg->>PY: registrar_pycsw.backend.ItemBackend<br/>Insert ISO 19115 metadata
    PY-->>PY: Write to PostgreSQL:<br/>postgresql://...@resource-catalogue-db/pycsw
    Reg->>SQ: Enqueue to seed_queue<br/>(defaultSuccessQueue)
    SQ->>Seed: Consume seed task
    Seed->>Seed: Pre-generate tiles<br/>(minzoom to maxzoom)
```

**Workspace Registration Configuration:**

In workspace-specific instances, the registrar uses a simplified configuration without the default route, focusing on item registration:

```yaml
config:
  disableDefaultRoute: true
  routes:
    items:
      path: registrar.route.stac.ItemRoute
      queue: register_queue
      replace: true
      backends:
        - path: "registrar.backend.eoxserver.ItemBackend"
          kwargs:
            instance_base_path: "/var/www/pvs/dev"
            instance_name: "pvs_instance"
            product_types: []
            auto_create_product_types: True
            automatic_visibilities: ["wms", "wcs"]
        - path: "registrar_pycsw.backend.ItemBackend"
          kwargs:
            repository_database_uri: "postgresql://postgres:mypass@resource-catalogue-db/pycsw"
            ows_url: "https://data-access.{{ workspace_name }}.develop.eoepca.org/ows"
            public_s3_url: "https://minio.develop.eoepca.org/{{ bucket }}"
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:84-164]()

## Harvesting Pipeline

The harvesting pipeline continuously polls external OpenSearch endpoints to discover and ingest new data products.

**Harvesting Process**

```mermaid
graph TB
    subgraph "External Sources"
        CrS2["CreoDIAS Sentinel-2<br/>datahub.creodias.eu/resto/.../Sentinel2"]
        CrL8["CreoDIAS Landsat-8<br/>datahub.creodias.eu/resto/.../Landsat8"]
        CrS1["CreoDIAS Sentinel-1<br/>datahub.creodias.eu/resto/.../Sentinel1"]
        CrS3["CreoDIAS Sentinel-3<br/>datahub.creodias.eu/resto/.../Sentinel3"]
    end
    
    subgraph "Harvester Component"
        HS2["Harvester: Sentinel2<br/>time: 2019-09-10 to 2019-09-11<br/>bbox: 14.9,47.7,16.4,48.7"]
        HL8["Harvester: Landsat8<br/>time: 2020-09-01 to 2020-09-05<br/>bbox: 19.7,34.7,28.5,42.0"]
        HS1GRD["Harvester: Sentinel1-GRD<br/>productType: GRD-COG"]
        HS1SLC["Harvester: Sentinel1-SLC<br/>productType: SLC"]
        HS3["Harvester: Sentinel3<br/>productType: OL_2_LFR___"]
    end
    
    subgraph "Postprocessors"
        PP2["harvester_eoepca.postprocess<br/>.postprocess_sentinel2"]
        PP8["harvester_eoepca.postprocess<br/>.postprocess_landsat8"]
        PP1["harvester_eoepca.postprocess<br/>.postprocess_sentinel1"]
        PP3["harvester_eoepca.postprocess<br/>.postprocess_sentinel3"]
    end
    
    subgraph "Queue"
        RegQ["register queue<br/>(Redis)"]
    end
    
    CrS2 -->|OpenSearch query<br/>JSON format| HS2
    CrL8 -->|OpenSearch query<br/>JSON format| HL8
    CrS1 -->|OpenSearch query<br/>JSON format| HS1GRD
    CrS1 -->|OpenSearch query<br/>JSON format| HS1SLC
    CrS3 -->|OpenSearch query<br/>JSON format| HS3
    
    HS2 --> PP2
    HL8 --> PP8
    HS1GRD --> PP1
    HS1SLC --> PP1
    HS3 --> PP3
    
    PP2 --> RegQ
    PP8 --> RegQ
    PP1 --> RegQ
    PP3 --> RegQ
```

**Harvester Properties:**

Each harvester is configured with:
- `resource.type`: Always "OpenSearch"
- `resource.opensearch.url`: OpenSearch description document URL
- `resource.opensearch.format`: Response format (application/json)
- `resource.opensearch.format.json.property_mapping`: Maps OpenSearch properties to STAC fields
- `resource.opensearch.query`: Query parameters (time, bbox, collection, extra_params)
- `filter`: Additional filtering rules
- `postprocessors`: External processing functions to transform results
- `queue`: Target queue for enqueuing results

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:957-1087]()

## Workspace Integration

The Data Access service supports per-workspace deployments for multi-tenant isolation. Each workspace gets its own instance with isolated storage and catalogue.

**Workspace Data Access Template**

```mermaid
graph TB
    subgraph "Workspace API"
        WS["Workspace API<br/>workspace-api.rm"]
    end
    
    subgraph "Template"
        Tmpl["template-hr-data-access.yaml<br/>HelmRelease template"]
    end
    
    subgraph "User Workspace: eric-workspace"
        VSEric["Data Access: vs<br/>namespace: eric-workspace"]
        S3Eric["S3 Bucket<br/>bucket: eric-workspace<br/>access_key_id: {{ access_key_id }}<br/>secret_access_key: {{ secret_access_key }}"]
        CatEric["Resource Catalogue DB<br/>postgresql://...@resource-catalogue-db/pycsw"]
        HarvEric["Harvester<br/>harvest-bucket-catalog<br/>root_path: /home/catalog.json"]
    end
    
    WS -->|Instantiate with<br/>workspace_name=eric<br/>bucket=eric-workspace<br/>credentials| Tmpl
    Tmpl -->|Deploy| VSEric
    VSEric -->|Read/Write| S3Eric
    VSEric -->|Register metadata| CatEric
    HarvEric -->|Harvest STAC| S3Eric
```

**Template Variables:**

The workspace template uses the following placeholders that are replaced during instantiation:

| Variable | Purpose | Example Value |
|----------|---------|---------------|
| `{{ workspace_name }}` | Namespace and hostname segment | `eric-workspace` |
| `{{ bucket }}` | S3 bucket name | `eric-workspace` |
| `{{ access_key_id }}` | S3 access key | (from workspace secret) |
| `{{ secret_access_key }}` | S3 secret key | (from workspace secret) |

**Workspace-Specific Harvester:**

Each workspace includes a harvester that monitors a STAC catalog in the user's S3 bucket:

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
      public: False
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:1-269]()

## Deployment Architecture

**Global vs Workspace Instances**

```mermaid
graph TB
    subgraph "Global Instance (rm namespace)"
        GlobalDA["data-access<br/>HelmRelease<br/>host: data-access.develop.eoepca.org"]
        GlobalRenderer["renderer: 4 replicas<br/>CPU: 1.5 limit / 0.5 request<br/>Memory: 3Gi limit / 1Gi request"]
        GlobalHarvester["harvester<br/>Sentinel-2, Landsat-8<br/>Sentinel-1, Sentinel-3"]
        GlobalCache["cache storage<br/>S3: cf2.cloudferro.com:8080<br/>bucket: cache-bucket"]
        GlobalData["data storage<br/>S3: data.cloudferro.com"]
    end
    
    subgraph "Workspace Instance (eric-workspace)"
        WSData["vs<br/>HelmRelease<br/>host: data-access.eric-workspace.develop.eoepca.org"]
        WSRenderer["renderer: 1 replica<br/>CPU: 100m request<br/>Memory: 300Mi request / 3Gi limit"]
        WSHarvester["harvester<br/>harvest-bucket-catalog<br/>(user's STAC catalog)"]
        WSStorage["data storage<br/>S3: minio.develop.eoepca.org<br/>bucket: eric-workspace"]
    end
    
    GlobalDA --> GlobalRenderer
    GlobalDA --> GlobalHarvester
    GlobalRenderer --> GlobalData
    GlobalRenderer --> GlobalCache
    
    WSData --> WSRenderer
    WSData --> WSHarvester
    WSRenderer --> WSStorage
```

**Key Differences:**

| Aspect | Global Instance | Workspace Instance |
|--------|-----------------|-------------------|
| Namespace | `rm` | `{{ workspace_name }}` |
| Renderer Replicas | 4 | 1 |
| Renderer Resources | Higher (1.5 CPU / 3Gi RAM) | Lower (0.1 CPU / 300Mi RAM) |
| Data Source | CloudFerro eodata | User S3 bucket |
| Cache | S3 cache bucket | Disabled |
| Harvesting | External OpenSearch | User STAC catalog |
| Ingress | Enabled (public) | Disabled (via Resource Guard) |
| Seeder | Configured | Disabled (`replicaCount: 0`) |
| Ingestor | Disabled | Disabled |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:864-876](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:69-269]()

## Configuration Reference

### Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `REGISTRAR_REPLACE` | `"true"` | Enable replacement of existing registrations |
| `CPL_VSIL_CURL_ALLOWED_EXTENSIONS` | `.TIF,.TIFF,.tif,.tiff,.xml,.jp2,.jpg,.jpeg,.png,.nc` | GDAL allowed file extensions for remote access |
| `AWS_ENDPOINT_URL_S3` | `https://minio.develop.eoepca.org` | S3 endpoint for AWS SDK |
| `AWS_HTTPS` | `"FALSE"` | Disable HTTPS for internal MinIO |
| `startup_scripts` | `/registrar_pycsw/registrar_pycsw/initialize-collections.sh` | Initialization scripts |

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:26-33]()

### Ingress Configuration

```yaml
ingress:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: data-access.develop.eoepca.org
  tls:
    - hosts:
        - data-access.develop.eoepca.org
      secretName: data-access-tls
```

**Annotations:**
- Extended read timeout (600s) for long-running rendering operations
- CORS enabled for cross-origin client access
- Automatic TLS certificate provisioning via cert-manager

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:35-47]()

### Redis Configuration

Redis serves as the message queue for the registration and seeding pipelines.

**Connection Details:**

| Property | Value |
|----------|-------|
| Global Instance Host | `data-access-redis-master` |
| Workspace Instance Host | `vs-redis-master` |
| Port | 6379 |
| Password | None (usePassword: false) |

**Persistence:**
- Storage Class: `managed-nfs-storage`
- Size: 1Gi
- Cluster Mode: Disabled

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:958-960](), [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:230-240]()

### Database Configuration

The Data Access service uses PostgreSQL for EOxServer instance data and connects to the Resource Catalogue PostgreSQL for metadata.

**Workspace Database Configuration:**

```yaml
database:
  persistence:
    enabled: true
    existingClaim: ""  # claim auto-generated
    storageClass: managed-nfs-storage
    size: "100Gi"
```

**Resource Catalogue Connection:**

All backends connect to the shared pycsw database:

```
postgresql://postgres:mypass@resource-catalogue-db/pycsw
```

**Sources:** [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:219-228](), [system/clusters/creodias/resource-management/hr-data-access.yaml:891-902]()

### Client Configuration

The web client provides time-based filtering and download capabilities.

**Time Domain Configuration:**

```yaml
client:
  config:
    eoxserverDownloadEnabled: true
    timeDomain:
      - "2002-01-01T00:00:00Z"
      - "customClientDateFuture1"
    displayTimeDomain:
      - "customClientDatePast1"
      - "customClientDateFuture1"
    selectedTimeDomain:
      - "customClientDatePast2"
      - "today"
    customClientDaysPast1: 90
    customClientDaysPast2: 1
    customClientDaysFuture1: 7
```

**Time Domain Explanation:**
- `timeDomain`: Overall available time range
- `displayTimeDomain`: Time range shown in UI by default (past 90 days to future 7 days)
- `selectedTimeDomain`: Initial selection (past 1 day to today)

**Sources:** [system/clusters/creodias/resource-management/hr-data-access.yaml:1093-1106]()