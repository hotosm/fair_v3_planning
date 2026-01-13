# Resource Catalogue

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



The Resource Catalogue provides standards-based metadata discovery and query capabilities for Earth Observation data, applications, and services in the EOEPCA platform. It implements OGC Catalogue Service for the Web (CSW) 3.0 and OpenSearch interfaces using pycsw, storing ISO 19115 metadata in PostgreSQL. Both a global catalogue and per-workspace catalogues are deployed to support multi-tenant data discovery.

For information about data visualization and access services, see [Data Access Services](#5.1). For details on how metadata enters the catalogue, see [Data Registration and Harvesting](#5.4). For workspace provisioning including catalogue instantiation, see [Multi-Tenant Workspaces](#5.5).

---

## Architecture Overview

The Resource Catalogue is built on pycsw, a Python implementation of OGC CSW. It serves as the central metadata repository for discovering EO products, collections, applications, ADES services, and other catalogued resources. The architecture supports both a global platform-wide catalogue and isolated per-workspace catalogues.

```mermaid
graph TB
    subgraph "External Clients"
        CSWClient["CSW Client<br/>(owslib.csw)"]
        OSClient["OpenSearch Client<br/>(pyops)"]
        Browser["Web Browser"]
    end
    
    subgraph "Resource Catalogue Deployment"
        Ingress["Nginx Ingress<br/>resource-catalogue.develop.eoepca.org"]
        PyCSW["pycsw Server<br/>Port 8000"]
        PYCSWDB[("PostgreSQL<br/>resource-catalogue-db<br/>database: pycsw")]
    end
    
    subgraph "Data Sources"
        Registrar["Registrar Service<br/>Multiple Backend Routes"]
        DataAccess["Data Access Renderer<br/>Metadata Generation"]
        Harvester["Harvester<br/>OpenSearch Polling"]
    end
    
    subgraph "Query Endpoints"
        CSWEndpoint["/csw<br/>OGC CSW 3.0"]
        OSEndpoint["/opensearch<br/>OpenSearch 1.1"]
        Collections["/collections<br/>STAC Collections"]
    end
    
    CSWClient -->|GetCapabilities<br/>GetRecords<br/>GetRecordById| Ingress
    OSClient -->|OpenSearch Query| Ingress
    Browser -->|HTTP GET| Ingress
    
    Ingress --> PyCSW
    PyCSW --> CSWEndpoint
    PyCSW --> OSEndpoint
    PyCSW --> Collections
    
    PyCSW -->|Query/Insert| PYCSWDB
    
    Registrar -->|ItemBackend<br/>CollectionBackend<br/>ADESBackend<br/>CWLBackend| PYCSWDB
    DataAccess -->|Product Metadata| Registrar
    Harvester -->|Harvested Records| Registrar
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:1-82]()
- [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py:1-156]()
- [test/acceptance/03__ResourceCatalogue/Resource_catalogue.robot:1-96]()

---

## Technology Stack

### pycsw Core

The Resource Catalogue uses pycsw, an OGC-compliant CSW server implementation written in Python. pycsw provides:

- **OGC CSW 3.0** interface for structured metadata queries
- **OpenSearch 1.1** interface for REST-based discovery
- **INSPIRE** profile support for European spatial data infrastructure compliance
- **Transactional CSW** for programmatic metadata insertion

The pycsw deployment is configured through the HelmRelease values structure.

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:38-82]()

### PostgreSQL Backend

Metadata records are stored in PostgreSQL using the ISO 19115 schema. The database deployment includes performance tuning parameters:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `shared_buffers` | 2GB | Memory for caching data |
| `effective_cache_size` | 6GB | Query planner estimate |
| `maintenance_work_mem` | 512MB | Index maintenance |
| `work_mem` | 4MB | Per-operation memory |
| `checkpoint_completion_target` | 0.9 | Checkpoint spreading |

The database is deployed with persistent storage via a 5Gi volume.

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:19-31]()

---

## Deployment Patterns

### Global Resource Catalogue

The platform-wide catalogue is deployed in the `rm` namespace and is accessible at `https://resource-catalogue.develop.eoepca.org`. It contains metadata for:

- **EO Products**: Sentinel-2, Landsat-8, Sentinel-1, Sentinel-3 scenes
- **Collections**: Product type groupings (S2MSI2A, L8MSI1TP, etc.)
- **Applications**: CWL workflow packages
- **ADES Services**: Processing service endpoints
- **Other Catalogues**: Federated catalogue references

Configuration for the global catalogue:

```yaml
pycsw:
  config:
    server:
      url: https://resource-catalogue.develop.eoepca.org/
    manager:
      transactions: "true"
      allowed_ips: "*"
    metadata:
      identification_title: EOEPCA Resource Catalogue
      identification_abstract: Based on pycsw, a Python OGC CSW server implementation
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:38-82]()

### Per-Workspace Catalogues

Each user workspace receives its own isolated Resource Catalogue instance deployed to the user's namespace. These workspace catalogues:

- Store user-specific STAC items generated by ADES processing jobs
- Reference the global catalogue via federated search
- Use the same pycsw image but with workspace-specific configuration
- Persist data on managed NFS storage

The workspace catalogue template uses placeholders for dynamic values:

```yaml
global:
  namespace: "{{ workspace_name }}"
pycsw:
  config:
    server:
      url: "https://resource-catalogue.{{ workspace_name }}.develop.eoepca.org"
      federatedcatalogues: https://resource-catalogue.develop.eoepca.org/collections/S2MSI2A
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68]()

```mermaid
graph TB
    subgraph "Global Namespace: rm"
        GlobalCat["resource-catalogue<br/>HelmRelease"]
        GlobalDB[("resource-catalogue-db<br/>PostgreSQL<br/>PVC: 5Gi")]
        GlobalCat --> GlobalDB
    end
    
    subgraph "Workspace: eric-workspace"
        WSCat1["rm-resource-catalogue<br/>HelmRelease<br/>(from template)"]
        WSDB1[("resource-catalogue-db<br/>PostgreSQL<br/>PVC: managed-nfs-storage-retain")]
        WSCat1 --> WSDB1
        WSCat1 -.->|federatedcatalogues| GlobalCat
    end
    
    subgraph "Workspace: alice-workspace"
        WSCat2["rm-resource-catalogue<br/>HelmRelease<br/>(from template)"]
        WSDB2[("resource-catalogue-db<br/>PostgreSQL<br/>PVC: managed-nfs-storage-retain")]
        WSCat2 --> WSDB2
        WSCat2 -.->|federatedcatalogues| GlobalCat
    end
    
    WorkspaceAPI["Workspace API<br/>HelmRelease Orchestrator"]
    Template["template-hr-resource-catalogue.yaml<br/>ConfigMap"]
    
    WorkspaceAPI -->|Instantiate with<br/>workspace_name| Template
    Template -->|Generate| WSCat1
    Template -->|Generate| WSCat2
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:1-68]()
- [system/clusters/creodias/resource-management/hr-workspace-api.yaml:46]()

---

## Query Interfaces

### OGC CSW Interface

The CSW endpoint supports standard OGC operations accessed at `/csw`:

| Operation | Purpose | Example Usage |
|-----------|---------|---------------|
| `GetCapabilities` | Retrieve service metadata | Discover supported operations and constraints |
| `GetRecords` | Query metadata records | Search with filters, sorting, paging |
| `GetRecordById` | Fetch specific record | Retrieve by identifier |
| `DescribeRecord` | Get record schema | Understand metadata structure |
| `GetDomain` | Query property values | Discover valid property ranges |
| `Transaction` | Insert/update/delete | Programmatic metadata management |

The CSW interface uses ISO 19115 output schema by default and supports filtering with OGC Filter Encoding:

```python
# Example CSW query using owslib
from owslib.csw import CatalogueServiceWeb
from owslib.fes import PropertyIsEqualTo, BBox, And

csw = CatalogueServiceWeb('https://resource-catalogue.develop.eoepca.org/csw')

# Spatial and temporal filters
bbox_filter = BBox([37, 13.9, 37.9, 15.1])
collection_filter = PropertyIsEqualTo('apiso:ParentIdentifier', 'S2MSI2A')

csw.getrecords2(
    constraints=[And([bbox_filter, collection_filter])],
    outputschema='http://www.isotc211.org/2005/gmd',
    maxrecords=10
)
```

**Sources:**
- [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py:74-98]()
- [test/acceptance/03__ResourceCatalogue/Resource_catalogue.robot:51-73]()

### OpenSearch Interface

The OpenSearch endpoint at `/opensearch` provides a REST-based alternative for metadata discovery. It supports:

- **Description document**: XML describing search parameters at `/opensearch`
- **Query interface**: URL-based search with query parameters
- **STAC extensions**: EO-specific search parameters

OpenSearch parameters include:

| Parameter | Type | Description |
|-----------|------|-------------|
| `eo:parentIdentifier` | string | Collection/product type |
| `eo:cloudCover` | integer | Maximum cloud coverage percentage |
| `bbox` | bbox | Geographic bounding box |
| `start` | datetime | Temporal range start |
| `end` | datetime | Temporal range end |

Example OpenSearch queries:

```
# All records
GET /opensearch/?mode=opensearch&service=CSW&version=3.0.0&request=GetRecords&elementsetname=full&resulttype=results&typenames=csw:Record

# Filter by collection
GET /opensearch/?mode=opensearch&service=CSW&version=3.0.0&request=GetRecords&elementsetname=full&resulttype=results&typenames=csw:Record&eo:parentIdentifier=S2MSI2A

# Filter by cloud cover
GET /opensearch/?eo:cloudCover=]20
```

**Sources:**
- [test/acceptance/03__ResourceCatalogue/Resource_catalogue.robot:85-96]()
- [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py:139-156]()

---

## Integration with Registrar Service

Metadata enters the Resource Catalogue through the Registrar Service, which implements multiple backend routes for different record types. The registrar consumes messages from Redis queues and writes to the PostgreSQL database using specialized backend classes.

```mermaid
graph LR
    subgraph "Redis Queues"
        ItemQ["register_queue<br/>STAC Items"]
        CollQ["register_collection_queue<br/>STAC Collections"]
        ADESQ["register_ades_queue<br/>ADES Services"]
        AppQ["register_application_queue<br/>CWL Applications"]
        CatQ["register_catalogue_queue<br/>Catalogue Services"]
        JSONQ["register_json_queue<br/>Generic JSON"]
        XMLQ["register_xml_queue<br/>Generic XML"]
    end
    
    subgraph "Registrar Backends"
        ItemBE["registrar_pycsw.backend.ItemBackend"]
        CollBE["registrar_pycsw.backend.CollectionBackend"]
        ADESBE["registrar_pycsw.backend.ADESBackend"]
        CWLBE["registrar_pycsw.backend.CWLBackend"]
        CatBE["registrar_pycsw.backend.CatalogueBackend"]
        JSONBE["registrar_pycsw.backend.JSONBackend"]
        XMLBE["registrar_pycsw.backend.XMLBackend"]
    end
    
    subgraph "Database"
        PYCSWDB[("PostgreSQL<br/>pycsw database<br/>ISO 19115 schema")]
    end
    
    ItemQ --> ItemBE
    CollQ --> CollBE
    ADESQ --> ADESBE
    AppQ --> CWLBE
    CatQ --> CatBE
    JSONQ --> JSONBE
    XMLQ --> XMLBE
    
    ItemBE --> PYCSWDB
    CollBE --> PYCSWDB
    ADESBE --> PYCSWDB
    CWLBE --> PYCSWDB
    CatBE --> PYCSWDB
    JSONBE --> PYCSWDB
    XMLBE --> PYCSWDB
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:878-948]()
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-data-access.yaml:84-164]()

### Backend Route Configuration

Each backend route is configured in the Data Access registrar component with specific parameters:

#### ItemBackend

Handles STAC Item registration from EO products:

```yaml
routes:
  items:
    path: registrar.route.stac.ItemRoute
    queue: register_queue
    replace: true
    backends:
      - path: registrar_pycsw.backend.ItemBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
          ows_url: https://data-access.develop.eoepca.org/ows
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:888-892]()

#### CollectionBackend

Manages STAC Collection metadata:

```yaml
routes:
  collections:
    path: registrar.route.stac.Collection
    queue: register_collection_queue
    replace: true
    backends:
      - path: registrar_pycsw.backend.CollectionBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:895-903]()

#### ADESBackend

Registers ADES service endpoints:

```yaml
routes:
  ades:
    path: registrar.route.json.JSONRoute
    queue: register_ades_queue
    replace: true
    backends:
      - path: registrar_pycsw.backend.ADESBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:904-912]()

#### CWLBackend

Catalogs CWL application packages:

```yaml
routes:
  application:
    path: registrar.route.json.JSONRoute
    queue: register_application_queue
    replace: true
    backends:
      - path: registrar_pycsw.backend.CWLBackend
        kwargs:
          repository_database_uri: postgresql://postgres:mypass@resource-catalogue-db/pycsw
```

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:913-921]()

---

## Server Configuration

### Service Metadata

The pycsw server configuration defines service-level metadata:

| Configuration Key | Purpose | Example Value |
|-------------------|---------|---------------|
| `server.url` | Service endpoint URL | `https://resource-catalogue.develop.eoepca.org/` |
| `manager.transactions` | Enable CSW-T operations | `"true"` |
| `manager.allowed_ips` | Transaction access control | `"*"` |
| `metadata.identification_title` | Service title | `EOEPCA Resource Catalogue` |
| `metadata.provider_name` | Organization name | `EOEPCA` |
| `metadata.provider_url` | Organization website | `https://eoepca.org/` |

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:44-82]()

### INSPIRE Profile

The catalogue supports the INSPIRE metadata profile for European spatial data:

```yaml
inspire:
  enabled: "true"
  languages_supported: eng,gre
  default_language: eng
  date: YYYY-MM-DD
  gemet_keywords: Utility and governmental services
  conformity_service: notEvaluated
  contact_name: Organization Name
  contact_email: Email Address
  temp_extent: YYYY-MM-DD/YYYY-MM-DD
```

When enabled, the catalogue validates and serves metadata according to INSPIRE technical guidelines.

**Sources:**
- [system/clusters/creodias/resource-management/hr-resource-catalogue.yaml:72-82]()

---

## Federated Search

Workspace catalogues implement federated search to query both local user data and global platform resources. The `federatedcatalogues` configuration parameter specifies remote catalogue endpoints:

```yaml
pycsw:
  config:
    server:
      federatedcatalogues: https://resource-catalogue.develop.eoepca.org/collections/S2MSI2A
```

This enables a workspace catalogue to:

1. Query local user-generated STAC items
2. Automatically search the global catalogue for platform datasets
3. Merge results from multiple catalogue sources
4. Provide unified discovery across personal and shared resources

```mermaid
sequenceDiagram
    participant User
    participant WSCat as "Workspace Catalogue<br/>(eric-workspace)"
    participant GlobalCat as "Global Catalogue<br/>(rm namespace)"
    
    User->>WSCat: CSW GetRecords<br/>(eo:parentIdentifier=S2MSI2A)
    
    WSCat->>WSCat: Query local PostgreSQL<br/>(user processing outputs)
    
    WSCat->>GlobalCat: Federated query<br/>federatedcatalogues endpoint
    GlobalCat-->>WSCat: Platform S2MSI2A records
    
    WSCat->>WSCat: Merge local + federated results
    WSCat-->>User: Combined result set
```

**Sources:**
- [system/clusters/creodias/resource-management/rm-workspace-charts/template-hr-resource-catalogue.yaml:33-34]()

---

## Operational Management

### Resource Registration

Data products are registered into the catalogue by pushing their identifiers to Redis queues. The `register-S2-L2A-data.sh` script demonstrates bulk registration:

```bash
kubectl -n rm exec --stdin --tty data-access-redis-master-0 -- \
  redis-cli lpush register_queue \
  EODATA/Sentinel-2/MSI/L2A/2020/09/02/S2B_MSIL2A_20200902T090559_N0214_R050_T34SFH_20200902T113910.SAFE/ \
  EODATA/Sentinel-2/MSI/L2A/2020/09/02/S2B_MSIL2A_20200902T090559_N0214_R050_T35SLB_20200902T113910.SAFE/
```

The registrar service consumes from `register_queue` and invokes `ItemBackend` to create ISO 19115 records in PostgreSQL.

**Sources:**
- [system/clusters/data/register-S2-L2A-data.sh:1-10]()
- [system/clusters/data/register-S2-L1C-data.sh:1-10]()

### Policy Management

Although the global Resource Catalogue in the reference deployment does not have ingress protection enabled, workspace catalogues can be protected by Resource Guards (PEPs). The `unregister-resource.sh` script includes catalogue-specific resource cleanup, indicating support for protected deployments:

```bash
# resource-catalogue-pep
echo -n "Delete resource ${resourceId} from resource-catalogue-pep..."
kubectl -n rm exec -it svc/resource-catalogue-pep -c resource-catalogue-pep -- \
  management_tools remove -r ${resourceId}
```

**Sources:**
- [bin/unregister-resource.sh:26-29]()

### Testing and Validation

The acceptance test suite validates catalogue functionality using `owslib.csw` for CSW queries and `pyops` for OpenSearch:

```python
# CSW operations validation
csw = CatalogueServiceWeb('https://resource-catalogue.develop.eoepca.org/csw')
operations = [op.name for op in csw.operations]
# Expects: ['GetCapabilities', 'GetRecords', 'GetRecordById', ...]

# Record queries with filters
bbox_query = BBox([37, 13.9, 37.9, 15.1])
begin = PropertyIsGreaterThanOrEqualTo(propertyname='apiso:TempExtent_begin', 
                                        literal='2021-04-02 00:00')
csw.getrecords2(constraints=[And([bbox_query, begin])])

# OpenSearch queries
client = pyops.Client(description_xml_url='https://resource-catalogue.develop.eoepca.org/opensearch')
results = client.search(params={"{eo:cloudCover?}": {"value": "]20"}})
```

**Sources:**
- [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py:17-98]()
- [test/acceptance/03__ResourceCatalogue/Resource_catalogue.robot:20-96]()

---

## Data Model

### ISO 19115 Metadata Records

The catalogue stores metadata following ISO 19115 geographic information standards. Key record properties include:

| Property | ISO Path | Purpose |
|----------|----------|---------|
| Identifier | `apiso:Identifier` | Unique record ID |
| Title | `apiso:Title` | Human-readable name |
| Abstract | `apiso:Abstract` | Description |
| Parent Identifier | `apiso:ParentIdentifier` | Collection reference |
| Temporal Extent | `apiso:TempExtent_begin/end` | Time range |
| Bounding Box | `apiso:BoundingBox` | Geographic extent |
| Type | `apiso:Type` | Resource type (dataset, service, etc.) |
| References | Links | Related resources (WMS, data download, etc.) |

**Sources:**
- [test/acceptance/03__ResourceCatalogue/CatalogueServiceWeb.py:79-98]()

### STAC Integration

While pycsw natively supports ISO 19115, the registrar backends translate STAC metadata into ISO records. The backends extract:

- **STAC Item** → ISO 19115 Dataset record
- **STAC Collection** → ISO 19115 Series/Collection record  
- **STAC assets** → ISO 19115 Distribution information (links to data files, WMS services)

This allows the catalogue to serve both ISO-compliant CSW clients and STAC-aware applications through appropriate output schemas.

**Sources:**
- [system/clusters/creodias/resource-management/hr-data-access.yaml:888-903]()
- [test/acceptance/02__Processing/01__ADES/data/app-deploy-body-atom.json:9]()