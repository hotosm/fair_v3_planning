# Configuration and Customization

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [backend/aiproject/settings.py](backend/aiproject/settings.py)
- [backend/aiproject/urls.py](backend/aiproject/urls.py)
- [backend/core/models.py](backend/core/models.py)
- [backend/core/serializers.py](backend/core/serializers.py)
- [backend/core/tasks.py](backend/core/tasks.py)
- [backend/core/urls.py](backend/core/urls.py)
- [backend/core/utils.py](backend/core/utils.py)
- [backend/core/views.py](backend/core/views.py)
- [backend/requirements.txt](backend/requirements.txt)
- [frontend/.env.sample](frontend/.env.sample)
- [frontend/src/components/ui/form/help-text/help-text.tsx](frontend/src/components/ui/form/help-text/help-text.tsx)
- [frontend/src/config/env.ts](frontend/src/config/env.ts)
- [frontend/src/config/index.ts](frontend/src/config/index.ts)
- [frontend/src/constants/ui-contents/models-content.ts](frontend/src/constants/ui-contents/models-content.ts)
- [frontend/src/constants/ui-contents/start-mapping-content.ts](frontend/src/constants/ui-contents/start-mapping-content.ts)
- [frontend/src/features/model-creation/components/progress-bar.tsx](frontend/src/features/model-creation/components/progress-bar.tsx)
- [frontend/src/features/model-creation/components/training-dataset/create-new.tsx](frontend/src/features/model-creation/components/training-dataset/create-new.tsx)
- [frontend/src/features/model-creation/components/training-dataset/select-existing.tsx](frontend/src/features/model-creation/components/training-dataset/select-existing.tsx)
- [frontend/src/features/start-mapping/components/map/legend-control.tsx](frontend/src/features/start-mapping/components/map/legend-control.tsx)
- [frontend/src/types/ui-contents.ts](frontend/src/types/ui-contents.ts)

</details>



This document explains how to configure and customize the fAIr AI-Assisted Mapping system. It covers environment variables, backend configuration options, frontend customization, and UI text modifications. For information about system architecture, see [System Architecture](#1.1), and for deployment instructions, see [Development and Deployment](#4).

## Configuration Overview

The fAIr system is highly configurable through a combination of environment variables and customizable UI content. This flexibility allows deployers to adapt the system to different environments, use cases, and branding requirements without modifying source code.

```mermaid
flowchart TD
    subgraph "Configuration Points"
        ENV["Environment Variables"]
        UI["UI Content Constants"]
        DB["Database Settings"]
        S3["S3 Storage Settings"]
        OSM["OSM Integration"]
        MLP["Model Training Parameters"]
        MAP["Map Visualization"]
        END["API Endpoints"]
        EMAIL["Email Notifications"]
    end

    subgraph "Configuration Files"
        BACKEND_ENV[".env (Backend)"]
        FRONTEND_ENV[".env (Frontend)"]
        SETTINGS["settings.py"]
        UI_CONTENT["UI Content Files"]
    end

    ENV --> BACKEND_ENV
    ENV --> FRONTEND_ENV
    BACKEND_ENV --> SETTINGS
    FRONTEND_ENV --> REACT["Frontend App"]
    
    UI --> UI_CONTENT
    UI_CONTENT --> REACT
    
    SETTINGS --> DB
    SETTINGS --> S3
    SETTINGS --> OSM
    SETTINGS --> MLP
    SETTINGS --> EMAIL
    
    REACT --> MAP
    REACT --> END
```

Sources: [backend/aiproject/settings.py:16-38](), [frontend/src/config/env.ts:1-92](), [frontend/src/config/index.ts:1-482]()

## Environment Variables Configuration

### Backend Environment Variables

The backend uses Django's settings module to load environment variables with default values through the `environ.Env()` utility.

Key backend environment variables include:

| Variable | Purpose | Default | 
|----------|---------|---------|
| `DEBUG` | Enable debug mode | `False` |
| `SECRET_KEY` | Django secret key | `"default_secret_key"` |
| `BUCKET_NAME` | S3 bucket for storing model data | `"fair-dev"` |
| `PARENT_BUCKET_FOLDER` | Path prefix in S3 bucket | `"dev"` |
| `AWS_REGION` | AWS region for S3 | `"us-east-1"` |
| `OSM_CLIENT_ID` | OpenStreetMap OAuth client ID | None |
| `OSM_CLIENT_SECRET` | OpenStreetMap OAuth client secret | None |
| `RAMP_EPOCHS_LIMIT` | Maximum epochs for RAMP model training | `40` |
| `YOLO_EPOCHS_LIMIT` | Maximum epochs for YOLO model training | `200` |
| `ENABLE_PREDICTION_API` | Enable the prediction API | `False` |
| `TRAINING_WORKSPACE` | Local directory for training data | Current directory + `/training` |
| `CELERY_BROKER_URL` | Redis URL for Celery | `"redis://127.0.0.1:6379/0"` |

These variables can be set in a `.env` file in the backend root directory or as system environment variables.

Sources: [backend/aiproject/settings.py:22-303]()

### Frontend Environment Variables

The frontend application uses environment variables via Vite's `import.meta.env` mechanism. These are loaded from `.env` files and exposed through a structured configuration system.

Key frontend environment variables include:

| Variable | Purpose | Default |
|----------|---------|---------|
| `VITE_BASE_API_URL` | Backend API endpoint | `"http://localhost:8000/api/v1/"` |
| `VITE_MAX_TRAINING_AREA_SIZE` | Maximum training area size (sq meters) | `5000000` |
| `VITE_MIN_TRAINING_AREA_SIZE` | Minimum training area size (sq meters) | `5797` |
| `VITE_MAX_ZOOM_LEVEL` | Maximum map zoom level | `21` |
| `VITE_MIN_ZOOM_LEVEL_FOR_START_MAPPING_PREDICTION` | Minimum zoom for predictions | `19` |
| `VITE_FAIR_PREDICTOR_API_URL` | Model prediction endpoint | `"https://predictor-dev.fair.hotosm.org/predict/"` |
| `VITE_TRAINING_AREAS_AOI_FILL_COLOR` | Training area fill color | `"#247DCACC"` |
| `VITE_OSM_HASHTAGS` | Hashtags for OSM edits | `"#HOT-fAIr"` |

The frontend provides a comprehensive `.env.sample` file that documents all available configuration options with detailed descriptions and default values.

Sources: [frontend/src/config/env.ts:1-92](), [frontend/.env.sample:1-212]()

## Backend Configuration

### Database Configuration

fAIr uses PostgreSQL with PostGIS extension for geospatial data storage. The database connection is configured through the `DATABASE_URL` environment variable or defaults to `postgis://admin:password@localhost:5432/ai`:

```python
DATABASES = {}
DATABASES["default"] = dj_database_url.config(
    default="postgis://admin:password@localhost:5432/ai", conn_max_age=500
)
```

Sources: [backend/aiproject/settings.py:163-167]()

### Model Training Configuration

The system allows customization of AI model training parameters through environment variables to control resource usage and model quality:

```mermaid
flowchart TD
    subgraph "Training Configuration Parameters"
        EPOCHS["Epoch Limits"]
        BATCH["Batch Size Limits"]
        WORKSPACE["Training Workspace"]
        BASE_MODEL["Base Model Selection"]
    end

    subgraph "Environment Variables"
        RAMP_EPOCHS["RAMP_EPOCHS_LIMIT"]
        YOLO_EPOCHS["YOLO_EPOCHS_LIMIT"]
        RAMP_BATCH["RAMP_BATCH_SIZE_LIMIT"]
        YOLO_BATCH["YOLO_BATCH_SIZE_LIMIT"]
        TRAINING_WORKSPACE_VAR["TRAINING_WORKSPACE"]
        RAMP_HOME["RAMP_HOME"]
        YOLO_HOME["YOLO_HOME"]
    end

    subgraph "Training Process"
        VALIDATOR["Training Validator"]
        TASK["train_model Task"]
        TRAINER["Trainer Class"]
    end

    RAMP_EPOCHS --> EPOCHS
    YOLO_EPOCHS --> EPOCHS
    RAMP_BATCH --> BATCH
    YOLO_BATCH --> BATCH
    TRAINING_WORKSPACE_VAR --> WORKSPACE
    RAMP_HOME --> BASE_MODEL
    YOLO_HOME --> BASE_MODEL

    EPOCHS --> VALIDATOR
    BATCH --> VALIDATOR
    WORKSPACE --> TASK
    BASE_MODEL --> TRAINER

    VALIDATOR --> TASK
    TASK --> TRAINER
```

The system enforces limits on training parameters to prevent resource exhaustion:

```python
# In TrainingSerializer.create method
if model.base_model == "RAMP":
    if epochs > settings.RAMP_EPOCHS_LIMIT:
        raise ValidationError(
            f"Epochs can't be greater than {settings.RAMP_EPOCHS_LIMIT} on this server"
        )
    if batch_size > settings.RAMP_BATCH_SIZE_LIMIT:
        raise ValidationError(
            f"Batch size can't be greater than {settings.RAMP_BATCH_SIZE_LIMIT} on this server"
        )
```

Sources: [backend/aiproject/settings.py:70-85](), [backend/core/views.py:167-186](), [backend/core/tasks.py:380-467]()

### S3 Storage Configuration

The system uses AWS S3 for storing model artifacts. This is configured through these environment variables:

```python
# S3
BUCKET_NAME = env("BUCKET_NAME", default="fair-dev")
PARENT_BUCKET_FOLDER = env(
    "PARENT_BUCKET_FOLDER", default="dev"
)  # use prod for production
AWS_REGION = env("AWS_REGION", default="us-east-1")
AWS_ACCESS_KEY_ID = env("AWS_ACCESS_KEY_ID", default=None)
AWS_SECRET_ACCESS_KEY = env("AWS_SECRET_ACCESS_KEY", default=None)
PRESIGNED_URL_EXPIRY = env("PRESIGNED_URL_EXPIRY", default=3600)
```

The S3Uploader utility in `core/utils.py` handles uploading model artifacts to S3 with the configured credentials.

Sources: [backend/aiproject/settings.py:59-67](), [backend/core/utils.py:397-460]()

### OpenStreetMap Integration

fAIr integrates with OpenStreetMap for authentication and data import. Configuration options include:

```python
OSM_CLIENT_ID = env("OSM_CLIENT_ID")
OSM_CLIENT_SECRET = env("OSM_CLIENT_SECRET")
OSM_URL = env("OSM_URL", default="https://www.openstreetmap.org")
OSM_SCOPE = env("OSM_SCOPE", default="read_prefs")
OSM_LOGIN_REDIRECT_URI = env(
    "OSM_LOGIN_REDIRECT_URI", default="http://127.0.0.1:8000/api/v1/auth/callback/"
)
OSM_SECRET_KEY = env("OSM_SECRET_KEY")
```

Sources: [backend/aiproject/settings.py:49-56]()

### Email Notification Configuration

The system can send email notifications for training events through SMTP:

```python
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = os.getenv("EMAIL_HOST", "smtp.gmail.com")
EMAIL_PORT = int(os.getenv("EMAIL_PORT", 587))
EMAIL_USE_TLS = os.getenv("EMAIL_USE_TLS", "True") == "True"
EMAIL_USE_SSL = os.getenv("EMAIL_USE_SSL", "False") == "True"
EMAIL_HOST_USER = os.getenv("EMAIL_HOST_USER", "example-email@example.com")
EMAIL_HOST_PASSWORD = os.getenv("EMAIL_HOST_PASSWORD", "example-email-password")
DEFAULT_FROM_EMAIL = os.getenv("DEFAULT_FROM_EMAIL", "no-reply@example.com")
```

Sources: [backend/aiproject/settings.py:263-271](), [backend/core/utils.py:463-487]()

## Frontend Configuration

### API Endpoints Configuration

The frontend configures API endpoints through environment variables, which are then processed through helper functions:

```typescript
/**
 * Helper function to safely parse environment variables as strings.
 */
export const parseStringEnv = (
  value: string | undefined,
  defaultValue: string,
): string => (value && value.trim() !== "" ? value.trim() : defaultValue);

/**
 * The backend api endpoint url.
 * Note: Ensure CORs is enabled in the backend and access is given to your port.
 */
export const BASE_API_URL: string = parseStringEnv(
  ENVS.BASE_API_URL,
  "http://localhost:8000/api/v1/",
);
```

This pattern is used for all API endpoints including OpenAerialMap, JOSM, and the fAIr predictor API.

Sources: [frontend/src/config/index.ts:33-90]()

### Map Visualization Configuration

The map visualization can be customized through various environment variables that control appearance and behavior:

```typescript
/**
 * The maximum zoom level for the map.
 * Model predictions require a max zoom of 22.
 * 21 is used here because 1 is already added to the 'currentZoom' in the useMapInstance() hook.
 */
export const MAX_ZOOM_LEVEL: number = parseIntEnv(ENVS.MAX_ZOOM_LEVEL, 21);

/**
 * The minimum zoom level for the map before the prediction components can be activated.
 */
export const MIN_ZOOM_LEVEL_FOR_START_MAPPING_PREDICTION: number = parseIntEnv(
  ENVS.MIN_ZOOM_LEVEL_FOR_START_MAPPING_PREDICTION,
  19,
);
```

Map colors and styling are also configurable:

```typescript
export const TRAINING_AREAS_AOI_FILL_COLOR: string = parseStringEnv(
  ENVS.TRAINING_AREAS_AOI_FILL_COLOR,
  "#247DCACC",
);
```

Sources: [frontend/src/config/index.ts:196-323]()

### Model Prediction Configuration

The frontend allows configuration of model prediction parameters:

```typescript
/**
 * The maximum tolerance for a prediction to be considered valid in the start mapping page.
 */
export const MAXIMUM_PREDICTION_TOLERANCE: number = parseIntEnv(
  ENVS.MAXIMUM_PREDICTION_TOLERANCE,
  10,
);

/**
 * The maximum area for a prediction to be considered valid in the start mapping page.
 */
export const MAXIMUM_PREDICTION_AREA: number = parseIntEnv(
  ENVS.MAXIMUM_PREDICTION_AREA,
  20,
);
```

These settings impact the quality and filtering of model predictions when users are mapping.

Sources: [frontend/src/config/index.ts:448-458]()

## UI Content Customization

The fAIr system has a sophisticated UI content system that allows customization of text, labels, and instructions throughout the application without modifying code. This is implemented through TypeScript constants that are strongly typed for reliability.

```mermaid
flowchart TD
    subgraph "UI Content System"
        subgraph "Content Definition Files"
            MODELS_CONTENT["models-content.ts"]
            START_MAPPING_CONTENT["start-mapping-content.ts"]
            SHARED_CONTENT["shared-content.ts"]
        end

        subgraph "Types"
            CONTENT_TYPES["ui-contents.ts Types"]
        end

        subgraph "Components"
            MODEL_CREATION["Model Creation Components"]
            START_MAPPING["Start Mapping Components"]
            SHARED_COMPONENTS["Shared Components"]
        end
    end

    CONTENT_TYPES --> MODELS_CONTENT
    CONTENT_TYPES --> START_MAPPING_CONTENT
    CONTENT_TYPES --> SHARED_CONTENT

    MODELS_CONTENT --> MODEL_CREATION
    START_MAPPING_CONTENT --> START_MAPPING
    SHARED_CONTENT --> SHARED_COMPONENTS
```

Sources: [frontend/src/constants/ui-contents/models-content.ts:1-334](), [frontend/src/constants/ui-contents/start-mapping-content.ts:1-105](), [frontend/src/types/ui-contents.ts:1-456]()

### Customizing UI Text

To customize text in the application:

1. Locate the relevant content file in `frontend/src/constants/ui-contents/`
2. Modify the text values while preserving the structure
3. Rebuild the frontend application

For example, to change the text for the model creation process, modify `models-content.ts`:

```typescript
export const MODELS_CONTENT: TModelsContent = {
  trainingArea: {
    // The retry button when the training area map fails to load
    retryButton: "retry", // Customizable text
    modalTitle: "Training Area", // Customizable text
    map: {
      loadingText: "Loading map...", // Customizable text
    },
  },
  // ...more content
}
```

The types in `ui-contents.ts` ensure that all required text fields are present.

Sources: [frontend/src/constants/ui-contents/models-content.ts:1-334](), [frontend/src/types/ui-contents.ts:1-305]()

### UI Structure and Configuration Points

The frontend UI has several key configuration points that affect the user experience:

```mermaid
classDiagram
    class UIConfigurationPoints {
        +ModelCreation
        +StartMapping
        +FormValidation
        +MapVisuals
    }
    
    class ModelCreation {
        +pageTitle: string
        +formLabels: object
        +helpText: string
        +validationMessages: string
    }
    
    class StartMapping {
        +pageTitle: function
        +buttonLabels: object
        +tooltips: object
        +messages: object
    }
    
    class FormValidation {
        +minLength: number
        +maxLength: number
        +pattern: regex
    }
    
    class MapVisuals {
        +colors: object
        +opacities: number
        +lineWidths: number
    }
    
    UIConfigurationPoints --> ModelCreation
    UIConfigurationPoints --> StartMapping
    UIConfigurationPoints --> FormValidation
    UIConfigurationPoints --> MapVisuals
```

Each component in the UI typically loads text from the content constants:

```typescript
// Example from training dataset component
<Input
  handleInput={(e) =>
    handleChange(MODEL_CREATION_FORM_NAME.DATASET_NAME, e.target.value)
  }
  value={formData.datasetName}
  toolTipContent={
    MODELS_CONTENT.modelCreation.trainingDataset.form.datasetName
      .toolTip
  }
  label={
    MODELS_CONTENT.modelCreation.trainingDataset.form.datasetName.label
  }
  labelWithTooltip
  placeholder={
    MODELS_CONTENT.modelCreation.trainingDataset.form.datasetName
      .placeholder
  }
/>
```

This approach allows complete customization of the application text while maintaining a consistent user experience.

Sources: [frontend/src/features/model-creation/components/training-dataset/create-new.tsx:91-122](), [frontend/src/components/ui/form/help-text/help-text.tsx:1-25]()

## Practical Examples

### Changing Model Training Limits

To adjust the maximum epochs and batch size for model training:

1. Set environment variables in backend `.env` file:
   ```
   RAMP_EPOCHS_LIMIT=60
   RAMP_BATCH_SIZE_LIMIT=16
   YOLO_EPOCHS_LIMIT=300
   YOLO_BATCH_SIZE_LIMIT=12
   ```

2. Restart the backend application to apply changes

### Customizing Frontend Map Colors

To change the appearance of training areas on the map:

1. Set environment variables in frontend `.env` file:
   ```
   VITE_TRAINING_AREAS_AOI_FILL_COLOR="#336699CC"
   VITE_TRAINING_AREAS_AOI_OUTLINE_COLOR="#336699CC"
   VITE_TRAINING_AREAS_AOI_FILL_OPACITY=0.5
   ```

2. Rebuild the frontend application

### Changing UI Content

To update instructional text for the model creation workflow:

1. Modify `frontend/src/constants/ui-contents/models-content.ts`:
   ```typescript
   modelCreation: {
     modelDetails: {
       pageTitle: "Create Custom AI Model",
       form: {
         modelName: {
           label: "Model Name",
           helpText: "Enter a descriptive name for your model (10-40 characters).",
           // ...
         },
       },
       // ...
     },
     // ...
   }
   ```

2. Rebuild the frontend application

Sources: [frontend/src/constants/ui-contents/models-content.ts:22-59]()

## Conclusion

The fAIr system offers extensive configuration options through environment variables and UI content customization. This allows deployers to adapt the system to different environments, branding requirements, and use cases without modifying source code. Backend configuration focuses on system operations and model training parameters, while frontend configuration enables customization of user interface appearance, behavior, and content.