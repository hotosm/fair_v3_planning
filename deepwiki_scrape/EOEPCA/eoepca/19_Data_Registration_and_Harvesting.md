# Data Registration and Harvesting

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.gitattributes](.gitattributes)
- [bin/dump-policy.sh](bin/dump-policy.sh)
- [bin/list-container-images.sh](bin/list-container-images.sh)
- [bin/unregister-resource.sh](bin/unregister-resource.sh)
- [system/clusters/creodias/resource-management/hr-data-access.yaml](system/clusters/creodias/resource-management/hr-data-access.yaml)
- [system/clusters/creodias/resource-management/hr-registration-api.yaml](system/clusters/creodias/resource-management/hr-registration-api.yaml)
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/hr-resource-catalogue.yaml)
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml](system/clusters/creodias/resource-management/hr-workspace-api.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml)
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml](system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml)
- [system/clusters/creodias/user-management/kustomization.yaml](system/clusters/creodias/user-management/kustomization.yaml)
- [system/clusters/data/register-S2-L1C-data.sh](system/clusters/data/register-S2-L1C-data.sh)
- [system/clusters/data/register-S2-L2A-data.sh](system/clusters/data/register-S2-L2A-data.sh)
- [test/acceptance/02__Processing/01__ADES/data/app-deploy-body-atom.json](test/acceptance/02__Processing/01__ADES/data/app-deploy-body-atom.json)
- [test/acceptance/02__Processing/01__ADES/data/app-deploy-body-cwl.json](test/acceptance/02__Processing/01__ADES/data/app-deploy-body-cwl.json)
- [test/acceptance/02__Processing/01__ADES/data/application-package-atom.xml](test/acceptance/02__Processing/01__ADES/data/application-package-atom.xml)
- [test/acceptance/02__Processing/01__ADES/data/application-package-cwl.cwl](test/acceptance/02__Processing/01__ADES/data/application-package-cwl.cwl)
- [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py](test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py)
- [test/acceptance/03__ResourceCatalogue/Resource_catalogue.robot](test/acceptance/03__ResourceCatalogue/Resource_catalogue.robot)

</details>



## Purpose and Scope

This document describes the data registration and harvesting subsystem within EOEPCA's Resource Management building block. This subsystem is responsible for:

1. **Harvesting** metadata from external Earth Observation data sources via OpenSearch APIs
2. **Processing** harvested metadata through configurable postprocessors
3. **Registering** processed metadata to the Resource Catalogue (pycsw database)
4. Enabling both automated continuous harvesting and manual data registration workflows

For information about querying the registered data, see [Resource Catalogue](#5.2). For information about visualizing and accessing registered data, see [Data Access Services](#5.1).

---

## System Architecture

The data registration and harvesting pipeline consists of three primary components that work together to ingest external EO data metadata into the EOEPCA platform.

```mermaid
graph TB
    subgraph "External Data Sources"
        CreoDIAS["CreoDIAS OpenSearch<br/>datahub.creodias.eu/resto"]
        OtherOS["Other OpenSearch<br/>Endpoints"]
        STACCat["STAC Catalogs<br/>(User Workspace)"]
    end
    
    subgraph "Data Access Service (rm namespace)"
        Harvester["vs.harvester<br/>Component"]
        Redis["data-access-redis-master<br/>Redis Queues"]
        Registrar["vs.registrar<br/>Component"]
    end
    
    subgraph "Redis Queues"
        RegQueue["register_queue"]
        RegCollQueue["register_collection_queue"]
        RegADESQueue["register_ades_queue"]
        RegAppQueue["register_application_queue"]
        RegCatQueue["register_catalogue_queue"]
        RegJSONQueue["register_json_queue"]
        RegXMLQueue["register_xml_queue"]
        SeedQueue["seed_queue"]
    end
    
    subgraph "Registrar Backends"
        ItemBackend["registrar_pycsw.backend.ItemBackend"]
        CollBackend["registrar_pycsw.backend.CollectionBackend"]
        ADESBackend["registrar_pycsw.backend.ADESBackend"]
        CWLBackend["registrar_pycsw.backend.CWLBackend"]
        CatalogueBackend["registrar_pycsw.backend.CatalogueBackend"]
        JSONBackend["registrar_pycsw.backend.JSONBackend"]
        XMLBackend["registrar_pycsw.backend.XMLBackend"]
    end
    
    subgraph "Storage"
        PyCSWDB[("PostgreSQL<br/>resource-catalogue-db/pycsw")]
    end
    
    CreoDIAS -->|Poll OpenSearch| Harvester
    OtherOS -->|Poll OpenSearch| Harvester
    STACCat -->|Read STAC| Harvester
    
    Harvester -->|Enqueue Items| RegQueue
    RegQueue --> Redis
    RegCollQueue --> Redis
    RegADESQueue --> Redis
    RegAppQueue --> Redis
    RegCatQueue --> Redis
    RegJSONQueue --> Redis
    RegXMLQueue --> Redis
    
    Redis -->|Consume| Registrar
    
    Registrar -->|Route: items| ItemBackend
    Registrar -->|Route: collections| CollBackend
    Registrar -->|Route: ades| ADESBackend
    Registrar -->|Route: application| CWLBackend
    Registrar -->|Route: catalogue| CatalogueBackend
    Registrar -->|Route: json| JSONBackend
    Registrar -->|Route: xml| XMLBackend
    
    ItemBackend -->|Write Metadata| PyCSWDB
    CollBackend -->|Write Metadata| PyCSWDB
    ADESBackend -->|Write Metadata| PyCSWDB
    CWLBackend -->|Write Metadata| PyCSWDB
    CatalogueBackend -->|Write Metadata| PyCSWDB
    JSONBackend -->|Write Metadata| PyCSWDB
    XMLBackend -->|Write Metadata| PyCSWDB
    
    Registrar -.->|Success| SeedQueue
```

**Diagram: Data Registration and Harvesting Architecture**

The harvester continuously polls external OpenSearch endpoints, processes results through postprocessors, and enqueues metadata to Redis. The registrar consumes from Redis queues and routes items to appropriate backends that write to the pycsw PostgreSQL database.

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:949-1087](), [system/clusters/creodias/resource-management/hr-data-access.yaml:878-947]()

---

## Harvester Service Configuration

The `vs.harvester` component is responsible for polling external data sources and enqueueing metadata for registration. It is configured via the `harvester` section in the Data Access HelmRelease.

### Harvester Component Structure

```mermaid
graph LR
    subgraph "Harvester Pod"
        HarvesterProc["harvester process"]
        Config["config.harvesters<br/>Configuration"]
    end
    
    subgraph "Resource Types"
        OpenSearch["OpenSearch<br/>Type: OpenSearch"]
        STACType["STAC Catalog<br/>Type: STACCatalog"]
    end
    
    subgraph "Postprocessors"
        S2Post["harvester_eoepca.postprocess.<br/>postprocess_sentinel2"]
        L8Post["harvester_eoepca.postprocess.<br/>postprocess_landsat8"]
        S1Post["harvester_eoepca.postprocess.<br/>postprocess_sentinel1"]
        S3Post["harvester_eoepca.postprocess.<br/>postprocess_sentinel3"]
    end
    
    Config --> HarvesterProc
    HarvesterProc --> OpenSearch
    HarvesterProc --> STACType
    
    OpenSearch --> S2Post
    OpenSearch --> L8Post
    OpenSearch --> S1Post
    OpenSearch --> S3Post
    
    S2Post --> RedisQueue["Redis Queue"]
    L8Post --> RedisQueue
    S1Post --> RedisQueue
    S3Post --> RedisQueue
    STACType --> RedisQueue
```

**Diagram: Harvester Component and Processing Pipeline**

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:949-1087]()

### Configured Harvesters

The global Data Access service is configured with multiple harvesters for different satellite missions. Each harvester definition specifies:

| Harvester Name | Resource Type | OpenSearch URL | Query Parameters | Postprocessor | Target Queue |
|---------------|---------------|----------------|------------------|---------------|--------------|
| `Sentinel2` | OpenSearch | `datahub.creodias.eu/resto/.../Sentinel2/describe.xml` | bbox, time range | `postprocess_sentinel2` | `register` |
| `Landsat8` | OpenSearch | `datahub.creodias.eu/resto/.../Landsat8/describe.xml` | bbox, time range | `postprocess_landsat8` | `register` |
| `Sentinel1-GRD` | OpenSearch | `datahub.creodias.eu/resto/.../Sentinel1/describe.xml` | bbox, time, productType=GRD-COG | `postprocess_sentinel1` | `register` |
| `Sentinel1-SLC` | OpenSearch | `datahub.creodias.eu/resto/.../Sentinel1/describe.xml` | bbox, time, productType=SLC | `postprocess_sentinel1` | `register` |
| `Sentinel3` | OpenSearch | `datahub.creodias.eu/resto/.../Sentinel3/describe.xml` | bbox, time, productType=OL_2_LFR___ | `postprocess_sentinel3` | `register` |

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:962-1087]()

### Harvester Configuration Example

The Sentinel-2 harvester configuration demonstrates the key parameters:

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
  filter: {}
  postprocessors:
    - type: external
      process: harvester_eoepca.postprocess.postprocess_sentinel2
      kwargs: {}
  queue: register
```

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:962-985]()

### Redis Connection Configuration

The harvester connects to the Redis queue system via configuration:

```yaml
config:
  redis:
    host: data-access-redis-master
    port: 6379
```

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:958-960]()

### Workspace-Specific Harvester Configuration

User workspace Data Access instances include a different harvester configuration that reads STAC catalogs from the user's S3 bucket:

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

This allows users to register data within their workspace by maintaining a STAC catalog at `/home/catalog.json` in their bucket.

Sources: [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:174-195]()

---

## Registrar Service Configuration

The `vs.registrar` component consumes metadata from Redis queues and registers it to the Resource Catalogue database through specialized backends.

### Registrar Architecture

```mermaid
graph TB
    subgraph "Registrar Service"
        RegistrarProc["registrar process"]
        RouteEngine["Route Engine"]
    end
    
    subgraph "Input Queues"
        Q1["register_queue<br/>(default items)"]
        Q2["register_collection_queue"]
        Q3["register_ades_queue"]
        Q4["register_application_queue"]
        Q5["register_catalogue_queue"]
        Q6["register_json_queue"]
        Q7["register_xml_queue"]
    end
    
    subgraph "Route Handlers"
        R1["registrar.route.stac.ItemRoute"]
        R2["registrar.route.stac.CollectionRoute"]
        R3["registrar.route.json.JSONRoute"]
        R4["registrar.route.json.JSONRoute"]
        R5["registrar.route.json.JSONRoute"]
        R6["registrar.route.json.JSONRoute"]
        R7["registrar.route.json.JSONRoute"]
    end
    
    subgraph "Backend Writers"
        B1["registrar_pycsw.backend.ItemBackend"]
        B2["registrar_pycsw.backend.CollectionBackend"]
        B3["registrar_pycsw.backend.ADESBackend"]
        B4["registrar_pycsw.backend.CWLBackend"]
        B5["registrar_pycsw.backend.CatalogueBackend"]
        B6["registrar_pycsw.backend.JSONBackend"]
        B7["registrar_pycsw.backend.XMLBackend"]
    end
    
    Q1 --> RegistrarProc
    Q2 --> RegistrarProc
    Q3 --> RegistrarProc
    Q4 --> RegistrarProc
    Q5 --> RegistrarProc
    Q6 --> RegistrarProc
    Q7 --> RegistrarProc
    
    RegistrarProc --> RouteEngine
    
    RouteEngine -->|items route| R1
    RouteEngine -->|collections route| R2
    RouteEngine -->|ades route| R3
    RouteEngine -->|application route| R4
    RouteEngine -->|catalogue route| R5
    RouteEngine -->|json route| R6
    RouteEngine -->|xml route| R7
    
    R1 --> B1
    R2 --> B2
    R3 --> B3
    R4 --> B4
    R5 --> B5
    R6 --> B6
    R7 --> B7
    
    B1 --> DB[("PostgreSQL<br/>pycsw database")]
    B2 --> DB
    B3 --> DB
    B4 --> DB
    B5 --> DB
    B6 --> DB
    B7 --> DB
```

**Diagram: Registrar Route and Backend Architecture**

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:878-947]()

### Default Configuration

The registrar is configured with default behaviors and multiple specialized routes:

```yaml
registrar:
  config:
    defaultBackends:
      - path: registrar_pycsw.backend.ItemBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
          ows_url: https://data-access.develop.eoepca.org/ows
    defaultSuccessQueue: seed_queue
```

The `defaultSuccessQueue` setting causes successfully registered items to be enqueued to `seed_queue` for cache seeding operations.

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:888-893]()

### Registrar Routes and Backends

Each route configuration defines how a specific type of metadata is processed:

| Route Name | Queue | Route Handler | Backend | Purpose |
|------------|-------|---------------|---------|---------|
| `collections` | `register_collection_queue` | `registrar.route.stac.Collection` | `CollectionBackend` | STAC collection registration |
| `ades` | `register_ades_queue` | `registrar.route.json.JSONRoute` | `ADESBackend` | ADES service registration |
| `application` | `register_application_queue` | `registrar.route.json.JSONRoute` | `CWLBackend` | CWL application package registration |
| `catalogue` | `register_catalogue_queue` | `registrar.route.json.JSONRoute` | `CatalogueBackend` | Catalogue endpoint registration |
| `json` | `register_json_queue` | `registrar.route.json.JSONRoute` | `JSONBackend` | Generic JSON metadata |
| `xml` | `register_xml_queue` | `registrar.route.json.JSONRoute` | `XMLBackend` | Generic XML metadata |

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:894-947]()

### Backend Database Connection

All backends connect to the same pycsw PostgreSQL database but use different metadata schemas and indexing strategies:

```yaml
backends:
  - path: registrar_pycsw.backend.ItemBackend
    kwargs:
      repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
      ows_url: https://data-access.develop.eoepca.org/ows
```

The `ows_url` parameter provides the OGC Web Services endpoint URL that will be associated with registered items.

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:889-892]()

### Replace Behavior

Routes can be configured with `replace: true` to update existing entries rather than creating duplicates:

```yaml
routes:
  collections:
    path: registrar.route.stac.Collection
    queue: register_collection_queue
    replace: true
```

This is particularly important for collections and applications that may be re-registered with updated metadata.

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:895-898]()

---

## Registration Pipeline Flow

The complete registration pipeline involves multiple stages from external data source to queryable catalogue entry.

```mermaid
sequenceDiagram
    participant ExtOS as External OpenSearch<br/>(CreoDIAS)
    participant Harvester as vs.harvester
    participant Post as Postprocessor<br/>(harvester_eoepca)
    participant Redis as Redis Queue<br/>(register_queue)
    participant Registrar as vs.registrar
    participant Backend as ItemBackend
    participant PyCSW as PostgreSQL<br/>(pycsw DB)
    participant RC as Resource Catalogue<br/>(CSW/OpenSearch)
    
    Harvester->>ExtOS: Poll OpenSearch<br/>(time window, bbox)
    ExtOS-->>Harvester: Return results<br/>(JSON format)
    
    Harvester->>Post: Process metadata
    Post->>Post: Transform to STAC
    Post->>Post: Add CloudFerro paths
    Post-->>Harvester: STAC Item JSON
    
    Harvester->>Redis: lpush register_queue<br/>(STAC Item)
    
    Redis->>Registrar: Consume queue
    Registrar->>Registrar: Route to items handler
    Registrar->>Backend: Process STAC Item
    
    Backend->>Backend: Transform to ISO 19115
    Backend->>PyCSW: INSERT metadata
    PyCSW-->>Backend: Success
    
    Backend-->>Registrar: Registration complete
    Registrar->>Redis: lpush seed_queue<br/>(if configured)
    
    Note over RC: Metadata now queryable
    
    RC->>PyCSW: CSW GetRecords query
    PyCSW-->>RC: Return records
```

**Diagram: Complete Registration Pipeline Sequence**

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:949-1087](), [system/clusters/creodias/resource-management/hr-data-access.yaml:878-947]()

### Registration Pipeline Steps

1. **Polling**: Harvester polls external OpenSearch endpoint with spatial/temporal constraints
2. **Postprocessing**: Results are transformed to STAC format with platform-specific metadata
3. **Enqueueing**: STAC items are pushed to Redis queue (`register_queue`)
4. **Consumption**: Registrar consumes from queue and routes to appropriate handler
5. **Backend Processing**: Backend transforms STAC to ISO 19115 metadata
6. **Database Write**: Metadata is inserted into pycsw PostgreSQL database
7. **Cache Seeding**: (Optional) Item is enqueued to `seed_queue` for cache generation
8. **Catalogue Availability**: Metadata becomes queryable via CSW/OpenSearch interfaces

---

## Manual Data Registration

For testing or manual data ingestion, items can be directly pushed to Redis queues using `kubectl` and `redis-cli`.

### Manual Registration Script

The repository includes convenience scripts for manual registration:

```bash
#!/usr/bin/env bash

kubectl -n rm exec --stdin --tty data-access-redis-master-0 -- \
  redis-cli lpush register_queue \
    EODATA/Sentinel-2/MSI/L2A/2020/09/02/S2B_MSIL2A_20200902T090559_N0214_R050_T34SFH_20200902T113910.SAFE/ \
    EODATA/Sentinel-2/MSI/L2A/2020/09/02/S2B_MSIL2A_20200902T090559_N0214_R050_T35SLB_20200902T113910.SAFE/ \
    EODATA/Sentinel-2/MSI/L2A/2020/09/02/S2B_MSIL2A_20200902T090559_N0214_R050_T34SGJ_20200902T113910.SAFE/
```

This executes `redis-cli lpush` inside the Redis pod to add CloudFerro EODATA paths directly to the `register_queue`.

Sources: [system/clusters/data/register-S2-L2A-data.sh:9](), [system/clusters/data/register-S2-L1C-data.sh:9]()

### Manual Registration Process

To manually register data:

1. **Identify Redis pod**: `data-access-redis-master-0` in namespace `rm`
2. **Execute redis-cli**: Use `kubectl exec` to run commands inside the pod
3. **Push to queue**: Use `lpush register_queue <path>` with CloudFerro EODATA paths
4. **Monitor processing**: Watch registrar logs for processing confirmation
5. **Verify registration**: Query Resource Catalogue to confirm metadata is available

The registrar will automatically consume these queue entries and process them through the standard pipeline.

---

## Environment Variables and Startup

The Data Access service uses several environment variables to control registration behavior:

| Variable | Value | Purpose |
|----------|-------|---------|
| `REGISTRAR_REPLACE` | `"true"` | Enable replacement of existing entries |
| `CPL_VSIL_CURL_ALLOWED_EXTENSIONS` | `.TIF,.TIFF,.tif,.tiff,.xml,.jp2,.jpg,.jpeg,.png,.nc` | GDAL virtual file system extensions |
| `AWS_ENDPOINT_URL_S3` | `https://minio.develop.eoepca.org` | S3 endpoint for data access |
| `AWS_HTTPS` | `"FALSE"` | Disable HTTPS for MinIO connections |

Additionally, startup scripts can be configured:

```yaml
env:
  startup_scripts:
    - /registrar_pycsw/registrar_pycsw/initialize-collections.sh
```

This initialization script registers default collections at service startup.

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:27-33]()

---

## Resource Requirements

The harvester and registrar components are configured with resource limits to ensure stable operation:

```yaml
harvester:
  resources:
    requests:
      cpu: 100m
      memory: 100Mi

registrar:
  replicaCount: 1
  resources:
    requests:
      cpu: 100m
      memory: 100Mi
```

Sources: [system/clusters/creodias/resource-management/hr-data-access.yaml:953-956](), [system/clusters/creodias/resource-management/hr-data-access.yaml:879-886]()

---

## Integration with Resource Catalogue

The registered metadata becomes queryable through the Resource Catalogue service's CSW and OpenSearch interfaces. The catalogue uses the pycsw database populated by the registrar backends.

Example test query demonstrating registered data retrieval:

```python
# Query for Sentinel-2 data
filter_list = [PropertyIsEqualTo('apiso:ParentIdentifier', 'S2MSI1C')]
csw.getrecords2(constraints=filter_list, outputschema='http://www.isotc211.org/2005/gmd')
```

The Resource Catalogue federates queries across both global and workspace-specific catalogues, enabling users to discover platform data and their own registered results.

Sources: [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py:74-98]()

---

## Summary

The Data Registration and Harvesting subsystem provides automated and manual mechanisms for ingesting EO data metadata into the EOEPCA platform. Key components include:

- **Harvester**: Polls external OpenSearch endpoints and STAC catalogs
- **Redis Queues**: Decouple harvesting from registration with message queues
- **Registrar**: Routes and processes different metadata types via specialized backends
- **pycsw Database**: Stores ISO 19115 metadata for CSW/OpenSearch queries

This architecture enables continuous ingestion of satellite data from CloudFerro/CreoDIAS while also supporting user workspace data registration through STAC catalogs.