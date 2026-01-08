# Backend System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/backend_build.yml](.github/workflows/backend_build.yml)
- [backend/.gitignore](backend/.gitignore)
- [backend/.pre-commit-config.yaml](backend/.pre-commit-config.yaml)
- [backend/README.md](backend/README.md)
- [backend/aiproject/settings.py](backend/aiproject/settings.py)
- [backend/aiproject/urls.py](backend/aiproject/urls.py)
- [backend/api-requirements.txt](backend/api-requirements.txt)
- [backend/core/admin.py](backend/core/admin.py)
- [backend/core/models.py](backend/core/models.py)
- [backend/core/serializers.py](backend/core/serializers.py)
- [backend/core/tasks.py](backend/core/tasks.py)
- [backend/core/urls.py](backend/core/urls.py)
- [backend/core/utils.py](backend/core/utils.py)
- [backend/core/views.py](backend/core/views.py)
- [backend/docker_sample_env](backend/docker_sample_env)
- [backend/login/admin.py](backend/login/admin.py)
- [backend/login/authentication.py](backend/login/authentication.py)
- [backend/login/permissions.py](backend/login/permissions.py)
- [backend/login/views.py](backend/login/views.py)
- [backend/pdm.lock](backend/pdm.lock)
- [backend/pyproject.toml](backend/pyproject.toml)
- [backend/requirements.txt](backend/requirements.txt)
- [backend/sample_env](backend/sample_env)
- [backend/tests/__init__.py](backend/tests/__init__.py)
- [backend/tests/factories.py](backend/tests/factories.py)
- [backend/tests/test_endpoints.py](backend/tests/test_endpoints.py)
- [backend/tests/test_views.py](backend/tests/test_views.py)

</details>



The Backend System of fAIr provides a comprehensive infrastructure for managing AI-assisted mapping tasks through a robust RESTful API, data persistence, and asynchronous processing capabilities. It serves as the foundation for model training, prediction generation, and data management within the fAIr platform.

For information about frontend interactions with this backend, see [Frontend System](#3). For details on setting up the backend for development, see [Installation and Setup](#1.3).

## Architecture Overview

The backend follows a layered architecture with clear separation of concerns:

```mermaid
graph TD
    subgraph "API Layer"
        DatasetViewSet["DatasetViewSet"]
        ModelViewSet["ModelViewSet"]
        TrainingViewSet["TrainingViewSet"]
        AOIViewSet["AOIViewSet"]
        LabelViewSet["LabelViewSet"]
        FeedbackViewset["FeedbackViewset"]
        PredictionView["PredictionView"]
    end
    
    subgraph "Data Layer"
        Dataset[("Dataset")]
        Model[("Model")]
        Training[("Training")]
        AOI[("AOI")]
        Label[("Label")]
        Feedback[("Feedback")]
    end
    
    subgraph "Processing Layer"
        TrainModelTask["train_model Task"]
        Trainer["Trainer Class"]
        ProcessRawData["process_rawdata"]
    end
    
    subgraph "Storage Layer"
        S3["S3 Storage"]
        Redis["Redis Queue"]
        PostgreSQL["PostgreSQL/PostGIS"]
    end
    
    subgraph "External Services"
        OSM["OpenStreetMap Auth"]
        RawDataAPI["Raw Data API"]
    end
    
    DatasetViewSet --> Dataset
    ModelViewSet --> Model
    TrainingViewSet --> Training
    TrainingViewSet --> TrainModelTask
    AOIViewSet --> AOI
    LabelViewSet --> Label
    FeedbackViewset --> Feedback
    
    Dataset --> PostgreSQL
    Model --> PostgreSQL
    Training --> PostgreSQL
    AOI --> PostgreSQL
    Label --> PostgreSQL
    Feedback --> PostgreSQL
    
    PredictionView --> Model
    PredictionView --> Training
    
    TrainModelTask --> Trainer
    Trainer --> RAMP["RAMP Training"]
    Trainer --> YOLO["YOLO Training"]
    
    AOIViewSet --> ProcessRawData
    ProcessRawData --> RawDataAPI
    
    TrainModelTask --> S3
    TrainModelTask --> Redis
    
    OSM --> DatasetViewSet
    OSM --> ModelViewSet
    OSM --> TrainingViewSet
    OSM --> AOIViewSet
    OSM --> LabelViewSet
    OSM --> FeedbackViewset
    OSM --> PredictionView
```

Sources: [backend/core/views.py:1-986](), [backend/core/tasks.py:1-468](), [backend/core/models.py:1-213](), [backend/aiproject/settings.py:1-303]()

## Core Components

### API Endpoints

The backend provides RESTful API endpoints implemented as Django REST Framework ViewSets in `core/views.py`. These endpoints handle client requests for managing datasets, models, training jobs, and predictions.

| Endpoint | ViewSet/View | Description |
|----------|--------------|-------------|
| `/dataset/` | `DatasetViewSet` | Manage training datasets |
| `/model/` | `ModelViewSet` | Manage AI models |
| `/training/` | `TrainingViewSet` | Manage model training jobs |
| `/aoi/` | `AOIViewSet` | Manage areas of interest for training |
| `/label/` | `LabelViewSet` | Manage training labels |
| `/feedback/` | `FeedbackViewset` | Manage user feedback on predictions |
| `/prediction/` | `PredictionView` | Generate predictions using trained models |
| `/auth/` | Auth views in `login/views.py` | Handle OSM authentication |

Each ViewSet provides standard CRUD operations for its resource, with additional specialized endpoints for specific actions like fetching OSM data or publishing trained models.

```mermaid
classDiagram
    class DatasetViewSet {
        +queryset: Dataset.objects.all()
        +serializer_class: DatasetSerializer
        +list()
        +retrieve()
        +create()
        +update()
        +destroy()
    }
    
    class ModelViewSet {
        +queryset: Model.objects.all()
        +serializer_class: ModelSerializer
        +list()
        +retrieve()
        +create()
        +update()
        +destroy()
    }
    
    class TrainingViewSet {
        +queryset: Training.objects.all()
        +serializer_class: TrainingSerializer
        +list()
        +retrieve()
        +create()
        +destroy()
    }
    
    class PredictionView {
        +post()
    }
    
    class AOIViewSet {
        +queryset: AOI.objects.all()
        +serializer_class: AOISerializer
        +list()
        +retrieve()
        +create()
        +update()
        +destroy()
    }
    
    class LabelViewSet {
        +queryset: Label.objects.all()
        +serializer_class: LabelSerializer
        +list()
        +retrieve()
        +create()
        +update()
        +destroy()
    }
    
    class RawdataApiAOIView {
        +post()
    }
    
    AOIViewSet --> RawdataApiAOIView
```

Sources: [backend/core/views.py:93-105](), [backend/core/views.py:324-347](), [backend/core/views.py:244-276](), [backend/core/views.py:786-886](), [backend/core/views.py:389-398](), [backend/core/views.py:400-433](), [backend/core/views.py:578-595]()

### Data Models

The core data models are defined in `core/models.py` using Django's ORM with GeoDjango extensions for spatial data.

```mermaid
erDiagram
    Dataset {
        int id PK
        string name
        int user FK
        string source_imagery
        int status
        float[] offset
    }
    
    AOI {
        int id PK
        int dataset FK
        geometry geom
        int label_status
        datetime label_fetched
        int user FK
    }
    
    Label {
        int id PK
        int aoi FK
        geometry geom
        bigint osm_id
        json tags
    }
    
    Model {
        int id PK
        int dataset FK
        string name
        string base_model
        int status
        int published_training
        int user FK
    }
    
    Training {
        int id PK
        int model FK
        string status
        int epochs
        int batch_size
        int[] zoom_level
        float accuracy
        string task_id
        int user FK
    }
    
    Feedback {
        int id PK
        int training FK
        geometry geom
        string feedback_type
        int user FK
    }
    
    Dataset ||--o{ AOI : "contains"
    AOI ||--o{ Label : "contains"
    Dataset ||--o{ Model : "has"
    Model ||--o{ Training : "undergoes"
    Training ||--o{ Feedback : "receives"
```

Sources: [backend/core/models.py:11-36](), [backend/core/models.py:40-52](), [backend/core/models.py:55-61](), [backend/core/models.py:64-86](), [backend/core/models.py:89-116](), [backend/core/models.py:119-135]()

### Asynchronous Processing

Celery is used for asynchronous processing of computationally intensive tasks like model training. The key components are:

- `train_model` task in `core/tasks.py`: Main task for model training
- `Trainer` class in `core/tasks.py`: Handles model-specific training logic
- Redis: Message broker for Celery tasks

The `train_model` task is triggered when a user creates a new Training instance through the API:

```mermaid
sequenceDiagram
    participant Client
    participant TrainingViewSet
    participant TrainingSerializer
    participant Redis
    participant CeleryWorker
    participant Trainer
    participant S3
    
    Client->>TrainingViewSet: POST /api/v1/training/
    TrainingViewSet->>TrainingSerializer: create()
    TrainingSerializer->>TrainingSerializer: validate()
    TrainingSerializer->>Redis: train_model.apply_async()
    TrainingSerializer->>Client: Return Training ID
    
    Redis->>CeleryWorker: task
    CeleryWorker->>CeleryWorker: prepare_data()
    CeleryWorker->>Trainer: run()
    
    alt RAMP Model
        Trainer->>Trainer: _train_ramp()
    else YOLO Model
        Trainer->>Trainer: _train_yolo()
    end
    
    Trainer->>CeleryWorker: Return results
    CeleryWorker->>S3: upload_to_s3()
    CeleryWorker->>CeleryWorker: finalize()
    CeleryWorker->>CeleryWorker: Update Training status
```

Sources: [backend/core/tasks.py:380-467](), [backend/core/tasks.py:99-290](), [backend/core/views.py:124-232]()

### Authentication

The backend uses OpenStreetMap OAuth 2.0 for authentication, implemented in the `login` app:

- `OsmAuthentication` class in `login/authentication.py`: Custom authentication backend
- `IsOsmAuthenticated` permission class in `login/permissions.py`: Custom permission class
- Auth views in `login/views.py`: Handle login, callback, and user data endpoints

The authentication process works as follows:

1. User is redirected to OSM for login
2. After successful authentication, OSM redirects back with an authorization code
3. The backend exchanges the code for an access token
4. API requests include the access token in the `access-token` header
5. The `OsmAuthentication` class validates the token and identifies the user

Sources: [backend/login/authentication.py:10-54](), [backend/login/permissions.py:6-60](), [backend/login/views.py:23-159]()

## Key Processes

### Model Training Flow

The model training process is a key feature of the backend. It handles the following steps:

```mermaid
flowchart TD
    Start([Start]) --> CreateTraining["TrainingSerializer.create()
    - Validate parameters
    - Create Training record
    (serializers.py)"]
    
    CreateTraining --> EnqueueTask["train_model.apply_async()
    - Submit task to Celery
    (serializers.py)"]
    
    EnqueueTask --> PrepareData["prepare_data()
    - Get AOIs and Labels
    - Download imagery
    (tasks.py)"]
    
    PrepareData --> TrainModel["Trainer.run()
    - Select model type (RAMP/YOLO)
    - Train model
    - Evaluate performance
    (tasks.py)"]
    
    TrainModel --> UploadModel["upload_to_s3()
    - Store model artifacts in S3
    (tasks.py)"]
    
    UploadModel --> Finalize["finalize()
    - Update Training record
    - Set status to FINISHED
    - Record accuracy
    (tasks.py)"]
    
    Finalize --> End([End])
```

Sources: [backend/core/tasks.py:380-467](), [backend/core/tasks.py:302-351](), [backend/core/tasks.py:99-290](), [backend/core/serializers.py:124-242]()

### Prediction Flow

When enabled, the prediction API generates predictions for new areas:

```mermaid
flowchart TD
    Start([Start]) --> ReceiveRequest["PredictionView.post()
    - Receive bbox, model_id, params
    (views.py)"]
    
    ReceiveRequest --> ValidateParams["PredictionParamSerializer
    - Validate input parameters
    (serializers.py)"]
    
    ValidateParams --> LoadModel["Load published model
    - Get Training by model.published_training
    (views.py)"]
    
    LoadModel --> GetImagery["Download imagery for bbox
    (predictor module)"]
    
    GetImagery --> RunModel["predict()
    - Run model inference
    (predictor module)"]
    
    RunModel --> FormatResponse["Format predictions as GeoJSON
    (views.py)"]
    
    FormatResponse --> End([End])
```

Sources: [backend/core/views.py:786-886](), [backend/core/serializers.py:360-449]()

### OSM Data Collection Flow

The backend provides functionality to fetch OSM data for training:

```mermaid
flowchart TD
    Start([Start]) --> ReceiveRequest["RawdataApiAOIView.post()
    - Receive aoi_id
    (views.py)"]
    
    ReceiveRequest --> EnqueueTask["process_rawdata_task()
    - Asynchronous task
    (views.py)"]
    
    EnqueueTask --> UpdateStatus1["Update AOI label_status to RUNNING
    (views.py)"]
    
    UpdateStatus1 --> FetchData["request_rawdata()
    - Call Raw Data API with geometry
    (utils.py)"]
    
    FetchData --> ProcessData["process_rawdata()
    - Convert GeoJSON to Labels
    (utils.py)"]
    
    ProcessData --> UpdateStatus2["Update AOI label_status to DOWNLOADED
    (views.py)"]
    
    UpdateStatus2 --> End([End])
```

Sources: [backend/core/views.py:578-610](), [backend/core/utils.py:222-247](), [backend/core/utils.py:250-281]()

## Configuration

The backend configuration is managed through environment variables, which can be set in a `.env` file or in the deployment environment. The configuration is defined in `aiproject/settings.py`.

Key configuration options include:

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgis://admin:password@localhost:5432/ai` |
| `CELERY_BROKER_URL` | Redis connection for Celery | `redis://127.0.0.1:6379/0` |
| `RAMP_HOME` | Path to RAMP model code | - |
| `TRAINING_WORKSPACE` | Path for training data | - |
| `BUCKET_NAME` | S3 bucket for model storage | `fair-dev` |
| `OSM_CLIENT_ID` | OSM OAuth client ID | - |
| `OSM_CLIENT_SECRET` | OSM OAuth client secret | - |
| `RAMP_EPOCHS_LIMIT` | Maximum epochs for RAMP training | 40 |
| `YOLO_EPOCHS_LIMIT` | Maximum epochs for YOLO training | 200 |
| `ENABLE_PREDICTION_API` | Enable prediction API | False |

Sources: [backend/aiproject/settings.py:30-86](), [backend/aiproject/settings.py:217-222](), [backend/aiproject/settings.py:243-254]()

## Technologies and Dependencies

The backend relies on several key technologies and libraries:

| Technology | Purpose |
|------------|---------|
| Django 4.1.4 | Web framework |
| Django REST Framework | API development |
| PostgreSQL/PostGIS | Spatial database |
| Celery | Asynchronous task processing |
| Redis | Message broker for Celery |
| GeoDjango | Spatial data handling |
| S3/boto3 | Model artifact storage |
| RAMP/YOLO | AI model architectures |

The complete list of dependencies can be found in the project's `requirements.txt` and `api-requirements.txt` files.

Sources: [backend/requirements.txt:1-7](), [backend/api-requirements.txt:1-28]()

## Development and Deployment

### Local Development

For local development, you'll need:

1. PostgreSQL with PostGIS extension
2. Redis server
3. Python 3.8+ environment
4. RAMP/YOLO model code repositories

Once set up, you can run:

```bash
# Create virtual environment and install dependencies
pip install -r requirements.txt

# Apply database migrations
python manage.py migrate

# Start development server
python manage.py runserver

# Start Celery worker
celery -A aiproject worker --loglevel=debug -Q ramp_training,yolo_training
```

### Containerized Deployment

The project includes Docker configuration for containerized deployment:

```bash
# Build and start containers
docker-compose up -d --build
```

Sources: [backend/README.md:1-196](), [backend/.github/workflows/backend_build.yml:1-139]()

## Conclusion

The fAIr Backend System provides a robust foundation for AI-assisted mapping with a well-structured architecture. The combination of Django REST Framework for the API, Celery for asynchronous processing, and PostgreSQL/PostGIS for spatial data storage creates a powerful platform for managing model training, prediction, and data management.

The system is designed to be scalable and maintainable, with clear separation of concerns and modular components. The integration with OpenStreetMap provides authentication and data access, while S3 storage ensures that trained models are persisted and accessible.

For detailed information about the API endpoints, see [API Endpoints](#2.1). For information on the data models, see [Data Models](#2.2). For details on the asynchronous processing system, see [Asynchronous Processing](#2.3).