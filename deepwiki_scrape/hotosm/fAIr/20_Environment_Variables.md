# Environment Variables

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/backend_build.yml](.github/workflows/backend_build.yml)
- [backend/README.md](backend/README.md)
- [backend/aiproject/settings.py](backend/aiproject/settings.py)
- [backend/aiproject/urls.py](backend/aiproject/urls.py)
- [backend/api-requirements.txt](backend/api-requirements.txt)
- [backend/core/models.py](backend/core/models.py)
- [backend/core/serializers.py](backend/core/serializers.py)
- [backend/core/tasks.py](backend/core/tasks.py)
- [backend/core/urls.py](backend/core/urls.py)
- [backend/core/utils.py](backend/core/utils.py)
- [backend/core/views.py](backend/core/views.py)
- [backend/docker_sample_env](backend/docker_sample_env)
- [backend/requirements.txt](backend/requirements.txt)
- [backend/sample_env](backend/sample_env)
- [backend/tests/__init__.py](backend/tests/__init__.py)
- [backend/tests/test_endpoints.py](backend/tests/test_endpoints.py)
- [backend/tests/test_views.py](backend/tests/test_views.py)
- [frontend/.env.sample](frontend/.env.sample)
- [frontend/src/config/env.ts](frontend/src/config/env.ts)
- [frontend/src/config/index.ts](frontend/src/config/index.ts)

</details>



This document provides a comprehensive guide to environment variables in the fAIr system. Environment variables control configuration settings across both backend and frontend components, allowing for customization without code changes. This page focuses specifically on the environment variables themselves and how they're used throughout the system.

For information about the overall system architecture, please refer to [System Architecture](#1.1).

## Environment Variable Loading Flow

The fAIr system handles environment variables differently in the backend versus the frontend. The following diagram illustrates how environment variables are loaded and used across the system:

```mermaid
graph TD
    subgraph "Backend (Django)"
        EnvFile[".env File"] -->|"environ.Env.read_env()"| EnvObject["Environment Object (env)"]
        EnvObject -->|"env('VARIABLE', default=value)"| SettingsModule["Settings (settings.py)"]
        SettingsModule --> BackendComponents["Django Components"]
        BackendComponents --> API["API Endpoints"]
        BackendComponents --> CeleryWorker["Celery Workers"]
        BackendComponents --> Models["ML Models (RAMP/YOLO)"]
    end
    
    subgraph "Frontend (React)"
        FrontendEnvFile[".env File"] -->|"import.meta.env"| FrontendConfig["Config (env.ts)"]
        FrontendConfig -->|"parseEnv* functions"| FrontendConstants["Constants (index.ts)"]
        FrontendConstants --> FrontendComponents["React Components"]
    end
    
    API <--> FrontendComponents
```

Sources: [backend/aiproject/settings.py:22-28](), [frontend/src/config/env.ts:1-92](), [frontend/src/config/index.ts:9-37]()

### Backend Environment Variables

In the backend, environment variables are loaded using the `django-environ` package from a `.env` file in the project root. The Django settings module reads these variables and applies them to various configurations:

```python
# From backend/aiproject/settings.py
import environ
env = environ.Env()
environ.Env.read_env(os.path.join(BASE_DIR, ".env"))
DEBUG = env("DEBUG", default=False)  # Example of loading with default value
```

### Frontend Environment Variables

In the frontend, environment variables are accessed using Vite's `import.meta.env`. These variables must be prefixed with `VITE_` to be accessible in the frontend code:

```typescript
// From frontend/src/config/env.ts
export const ENVS = {
  BASE_API_URL: import.meta.env.VITE_BASE_API_URL,
  // Additional environment variables
}
```

The frontend also includes helper functions to safely parse environment variables with default values:

```typescript
// From frontend/src/config/index.ts
export const parseIntEnv = (
  value: string | undefined,
  defaultValue: number,
): number =>
  value !== undefined && !isNaN(parseInt(value, 10))
    ? parseInt(value, 10)
    : defaultValue;
```

## Environment Variable Categories

The following diagram shows how different categories of environment variables relate to system components:

```mermaid
graph TD
    subgraph "Environment Variable Categories"
        CoreSettings["Django Core Settings
        (DEBUG, SECRET_KEY, etc.)"]
        
        OsmSettings["OSM Integration
        (OSM_CLIENT_ID, OSM_SECRET_KEY, etc.)"]
        
        S3Settings["AWS S3 Configuration
        (BUCKET_NAME, AWS_REGION, etc.)"]
        
        TrainingSettings["Model Training
        (EPOCHS_LIMIT, RAMP_HOME, etc.)"]
        
        CelerySettings["Celery and Tasks
        (CELERY_BROKER_URL, LOG_PATH, etc.)"]
        
        EmailSettings["Email Configuration
        (EMAIL_HOST, EMAIL_PORT, etc.)"]
        
        FrontendSettings["Frontend Settings
        (VITE_BASE_API_URL, VITE_MAX_ZOOM_LEVEL, etc.)"]
    end
    
    subgraph "System Components"
        DjangoCore["Django Core"]
        AuthSystem["Authentication"]
        StorageSystem["S3 Storage"]
        ModelTraining["Model Training"]
        TaskQueue["Celery Task Queue"]
        NotificationSystem["Notifications"]
        FrontendApp["React Frontend"]
    end
    
    CoreSettings --> DjangoCore
    OsmSettings --> AuthSystem
    S3Settings --> StorageSystem
    TrainingSettings --> ModelTraining
    CelerySettings --> TaskQueue
    EmailSettings --> NotificationSystem
    FrontendSettings --> FrontendApp
```

## Core Django Settings

These environment variables configure the core Django settings:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `DEBUG` | Enables/disables debug mode | `False` | `True` |
| `SECRET_KEY` | Secret key for cryptographic signing | `"default_secret_key"` | `"your-secret-key"` |
| `HOSTNAME` | Hostname where the application is running | `"127.0.0.1"` | `"fair.hotosm.org"` |
| `FRONTEND_URL` | URL of the frontend application | `"https://fair.hotosm.org"` | `"http://localhost:3000"` |
| `DATABASE_URL` | Database connection URL | `"postgis://admin:password@localhost:5432/ai"` | `"postgis://user:pass@host:port/dbname"` |
| `DEFAULT_PAGINATION_SIZE` | Default number of items per page in API responses | `50` | `100` |

Sources: [backend/aiproject/settings.py:29-38](), [backend/aiproject/settings.py:122-133]()

## OpenStreetMap (OSM) Integration

These environment variables configure OSM authentication and integration:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `OSM_CLIENT_ID` | OAuth2 client ID for OSM authentication | Required | `"your-client-id"` |
| `OSM_CLIENT_SECRET` | OAuth2 client secret for OSM authentication | Required | `"your-client-secret"` |
| `OSM_URL` | Base URL for OSM API | `"https://www.openstreetmap.org"` | `"https://www.openstreetmap.org"` |
| `OSM_SCOPE` | OAuth2 scope for OSM authentication | `"read_prefs"` | `"read_prefs"` |
| `OSM_LOGIN_REDIRECT_URI` | Redirect URI for OSM authentication | `"http://127.0.0.1:8000/api/v1/auth/callback/"` | `"https://your-app.com/api/v1/auth/callback/"` |
| `OSM_SECRET_KEY` | Secret key for OSM authentication | Required | `"your-secret-key"` |
| `EXPORT_TOOL_API_URL` | URL for the OSM export tool API | `"https://api-prod.raw-data.hotosm.org/v1"` | `"https://api-prod.raw-data.hotosm.org/v1"` |

Sources: [backend/aiproject/settings.py:49-56](), [backend/core/views.py:85-86]()

## AWS S3 Configuration

These environment variables configure AWS S3 storage for model artifacts:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `BUCKET_NAME` | Name of the S3 bucket | `"fair-dev"` | `"your-bucket-name"` |
| `PARENT_BUCKET_FOLDER` | Parent folder in the S3 bucket | `"dev"` | `"prod"` |
| `AWS_REGION` | AWS region for the S3 bucket | `"us-east-1"` | `"us-west-2"` |
| `AWS_ACCESS_KEY_ID` | AWS access key ID | None | `"your-access-key-id"` |
| `AWS_SECRET_ACCESS_KEY` | AWS secret access key | None | `"your-secret-access-key"` |
| `PRESIGNED_URL_EXPIRY` | Expiry time (in seconds) for S3 presigned URLs | `3600` | `86400` |

Sources: [backend/aiproject/settings.py:60-67](), [backend/core/utils.py:34-44](), [backend/core/utils.py:398-420]()

## Model Training Configuration

These environment variables configure model training parameters and limits:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `EPOCHS_LIMIT` | Maximum number of epochs for training | `20` | `30` |
| `BATCH_SIZE_LIMIT` | Maximum batch size for training | `8` | `16` |
| `YOLO_EPOCHS_LIMIT` | Maximum number of epochs for YOLO training | `200` | `300` |
| `YOLO_BATCH_SIZE_LIMIT` | Maximum batch size for YOLO training | `8` | `16` |
| `RAMP_EPOCHS_LIMIT` | Maximum number of epochs for RAMP training | `40` | `50` |
| `RAMP_BATCH_SIZE_LIMIT` | Maximum batch size for RAMP training | `8` | `16` |
| `TRAINING_WORKSPACE_DOWNLOAD_LIMIT` | Download limit for training workspace | `200` | `500` |
| `RAMP_HOME` | Path to RAMP home directory | Required | `"/path/to/ramp"` |
| `YOLO_HOME` | Path to YOLO home directory | `os.getcwd()` | `"/path/to/yolo"` |
| `TRAINING_WORKSPACE` | Path to training workspace | `os.path.join(os.getcwd(), "training")` | `"/path/to/training"` |
| `ENABLE_PREDICTION_API` | Enables/disables prediction API | `False` | `True` |

Sources: [backend/aiproject/settings.py:71-86](), [backend/aiproject/settings.py:243-254](), [backend/core/views.py:166-186](), [backend/core/tasks.py:381-464]()

## Celery and Task Queue Configuration

These environment variables configure Celery and task queues for asynchronous processing:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `CELERY_BROKER_URL` | URL for Celery broker | `"redis://127.0.0.1:6379/0"` | `"redis://redis:6379/0"` |
| `CELERY_RESULT_BACKEND` | URL for Celery result backend | `"redis://127.0.0.1:6379/0"` | `"redis://redis:6379/0"` |
| `LOG_PATH` | Path for log files | `os.path.join(os.getcwd(), "log")` | `"/var/log/fair"` |
| `LOG_LINE_STREAM_TRUNCATE_VALUE` | Number of log lines to display in API responses | `10` | `20` |

Sources: [backend/aiproject/settings.py:217-232](), [backend/aiproject/settings.py:257](), [backend/core/tasks.py:43-82](), [backend/core/views.py:696]()

## Email Configuration

These environment variables configure email notifications:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `EMAIL_HOST` | SMTP host for sending emails | `"smtp.gmail.com"` | `"smtp.mailgun.org"` |
| `EMAIL_PORT` | SMTP port for sending emails | `587` | `465` |
| `EMAIL_USE_TLS` | Uses TLS for SMTP connection | `True` | `False` |
| `EMAIL_USE_SSL` | Uses SSL for SMTP connection | `False` | `True` |
| `EMAIL_HOST_USER` | Username for SMTP authentication | `"example-email@example.com"` | `"your-email@gmail.com"` |
| `EMAIL_HOST_PASSWORD` | Password for SMTP authentication | `"example-email-password"` | `"your-password"` |
| `DEFAULT_FROM_EMAIL` | Default sender email address | `"no-reply@example.com"` | `"no-reply@your-domain.com"` |
| `FAIL_EMAIL_SILENTLY` | Silently fails when email sending fails | `"True"` | `"False"` |

Sources: [backend/aiproject/settings.py:263-271](), [backend/core/utils.py:463-510]()

## CORS Configuration

These environment variables configure Cross-Origin Resource Sharing (CORS):

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `CORS_ORIGIN_ALLOW_ALL` | Allows all origins (in debug mode only) | `True` (in debug mode), `False` (in production) | `True` |
| `CORS_ALLOWED_ORIGINS` | Comma-separated list of allowed origins | `"http://127.0.0.1:8000"` | `"https://app.example.com,https://admin.example.com"` |

Sources: [backend/aiproject/settings.py:276-302]()

## Frontend Environment Variables

The frontend uses Vite environment variables, which must be prefixed with `VITE_`. Here are some key frontend environment variables:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `VITE_BASE_API_URL` | URL of the backend API | `"http://localhost:8000/api/v1/"` | `"https://api.fair.hotosm.org/api/v1/"` |
| `VITE_MATOMO_ID` | Matomo analytics ID | `"0"` | `"123"` |
| `VITE_MATOMO_APP_DOMAIN` | Matomo application domain | `"fair.hotosm.org"` | `"your-domain.com"` |
| `VITE_MAX_TRAINING_AREA_SIZE` | Maximum size of training areas (sq meters) | `5000000` | `10000000` |
| `VITE_MIN_TRAINING_AREA_SIZE` | Minimum size of training areas (sq meters) | `5797` | `1000` |
| `VITE_MAX_ZOOM_LEVEL` | Maximum zoom level for the map | `21` | `22` |
| `VITE_MIN_ZOOM_LEVEL_FOR_START_MAPPING_PREDICTION` | Minimum zoom level for prediction | `19` | `18` |
| `VITE_FAIR_PREDICTOR_API_URL` | URL of the fAIr predictor API | `"https://predictor-dev.fair.hotosm.org/predict/"` | `"https://predictor.fair.hotosm.org/predict/"` |

Sources: [frontend/src/config/env.ts:1-92](), [frontend/src/config/index.ts:47-468](), [frontend/.env.sample:1-212]()

## Environment Variables in the Model Training Process

The following diagram illustrates how environment variables are used in the model training process, a key workflow in the fAIr system:

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend
    participant CeleryWorker
    participant ModelTraining
    participant S3Storage

    User->>Frontend: Create model & training
    Frontend->>Backend: POST /api/v1/training/
    Note over Backend: Uses EPOCHS_LIMIT, BATCH_SIZE_LIMIT
    Backend->>CeleryWorker: Enqueue train_model task
    Note over CeleryWorker: Uses CELERY_BROKER_URL
    CeleryWorker->>ModelTraining: Run training
    Note over ModelTraining: Uses RAMP_HOME, YOLO_HOME, TRAINING_WORKSPACE
    ModelTraining->>S3Storage: Upload model artifacts
    Note over S3Storage: Uses BUCKET_NAME, AWS_ACCESS_KEY_ID
    S3Storage-->>ModelTraining: Upload success
    ModelTraining-->>CeleryWorker: Training complete
    CeleryWorker-->>Backend: Update training status
    Backend->>NotificationSystem: Send notification
    Note over NotificationSystem: Uses EMAIL_* variables
    Backend-->>Frontend: Return training status
```

Sources: [backend/core/tasks.py:380-467](), [backend/core/utils.py:397-460](), [backend/core/views.py:124-232]()

## Environment File Templates

### Backend `.env` Template

```
DEBUG=True
SECRET_KEY=your-secret-key
DATABASE_URL=postgis://admin:password@localhost:5432/ai
EXPORT_TOOL_API_URL=https://api-prod.raw-data.hotosm.org/v1
CORS_ALLOWED_ORIGINS=http://localhost:3000
HOSTNAME=
OSM_CLIENT_ID=your-osm-client-id
OSM_CLIENT_SECRET=your-osm-client-secret
OSM_URL=https://www.openstreetmap.org
OSM_SCOPE=read_prefs
OSM_LOGIN_REDIRECT_URI=http://127.0.0.1:8000/api/v1/auth/callback/
OSM_SECRET_KEY=your-osm-secret-key
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
RAMP_HOME=/path/to/ramp
TRAINING_WORKSPACE=/path/to/training
LOG_PATH=/path/to/log
BUCKET_NAME=fair-dev
PARENT_BUCKET_FOLDER=dev
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your-aws-access-key-id
AWS_SECRET_ACCESS_KEY=your-aws-secret-access-key
ENABLE_PREDICTION_API=False
```

Sources: [backend/sample_env:1-22]()

### Frontend `.env` Template

```
VITE_BASE_API_URL=http://localhost:8000/api/v1/
VITE_MATOMO_ID=0
VITE_MATOMO_APP_DOMAIN=fair.hotosm.org
VITE_MAX_TRAINING_AREA_SIZE=5000000
VITE_MIN_TRAINING_AREA_SIZE=5797
VITE_MAX_ZOOM_LEVEL=21
VITE_MIN_ZOOM_LEVEL_FOR_START_MAPPING_PREDICTION=19
VITE_FAIR_PREDICTOR_API_URL=https://predictor-dev.fair.hotosm.org/predict/
VITE_OSM_DATABASE_STATUS_API_URL=https://api-prod.raw-data.hotosm.org/v1/status/
VITE_OAM_TITILER_ENDPOINT=https://titiler.hotosm.org/
VITE_OAM_S3_BUCKET_URL=https://oin-hotosm-temp.s3.us-east-1.amazonaws.com/
```

Sources: [frontend/.env.sample:1-212]()

## Importance of Environment Variables

Environment variables are crucial for the fAIr system for several reasons:

1. **Configuration without code changes**: System behavior can be modified without modifying the codebase
2. **Environment-specific settings**: Different settings can be used across development, testing, and production environments
3. **Security**: Sensitive information like API keys and passwords are kept out of the codebase
4. **Deployment flexibility**: The system can be deployed in different environments with appropriate configurations
5. **Feature toggling**: Features like the prediction API can be enabled or disabled using environment variables

Proper configuration of environment variables is essential for the correct functioning of the fAIr system across all environments.