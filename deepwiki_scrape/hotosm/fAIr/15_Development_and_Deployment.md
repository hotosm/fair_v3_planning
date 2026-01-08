# Development and Deployment

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/backend_build.yml](.github/workflows/backend_build.yml)
- [.github/workflows/docker_build.yml](.github/workflows/docker_build.yml)
- [.github/workflows/docker_publish_image.yml](.github/workflows/docker_publish_image.yml)
- [.github/workflows/frontend_build.yml](.github/workflows/frontend_build.yml)
- [.github/workflows/frontend_build_push.yml](.github/workflows/frontend_build_push.yml)
- [Readme.md](Readme.md)
- [backend/Dockerfile](backend/Dockerfile)
- [backend/Dockerfile_CPU](backend/Dockerfile_CPU)
- [backend/README.md](backend/README.md)
- [backend/api-requirements.txt](backend/api-requirements.txt)
- [backend/docker_sample_env](backend/docker_sample_env)
- [backend/sample_env](backend/sample_env)
- [backend/tests/__init__.py](backend/tests/__init__.py)
- [backend/tests/test_endpoints.py](backend/tests/test_endpoints.py)
- [backend/tests/test_views.py](backend/tests/test_views.py)
- [docker-compose-cpu.yml](docker-compose-cpu.yml)
- [docker-compose.yml](docker-compose.yml)
- [docs/Docker-installation.md](docs/Docker-installation.md)
- [frontend/Dockerfile.frontend](frontend/Dockerfile.frontend)
- [run_migrations.sh](run_migrations.sh)
- [setup-ramp.sh](setup-ramp.sh)

</details>



This document provides an overview of the development workflow and deployment processes for the fAIr AI-assisted mapping system. It covers local development setup, CI/CD pipelines, Docker containerization, and production deployment methods. For installation instructions, see [Installation and Setup](#1.3).

## Development Environment Setup

The fAIr system provides multiple development environment options, with Docker being the recommended approach for consistency across development environments.

### Docker-Based Development

Docker Compose configuration provides a complete development environment with all required services:

```mermaid
graph TD
    subgraph "Docker Environment"
        Frontend["Frontend Container<br>(React)"]
        API["API Container<br>(Django)"]
        Worker["Worker Container<br>(Celery)"]
        Flower["Flower Dashboard<br>(Worker Monitoring)"]
        Redis["Redis Container<br>(Queue & Cache)"]
        PostgreSQL["PostgreSQL/PostGIS<br>(Database)"]
    end

    Frontend --> API
    API --> PostgreSQL
    API --> Redis
    Worker --> Redis
    Worker --> PostgreSQL
    Flower --> Redis
    
    subgraph "Host System"
        RAMP_HOME["RAMP_HOME<br>(AI Model Base)"]
        TRAINING_WORKSPACE["TRAINING_WORKSPACE<br>(Training Data)"]
    end
    
    API --> RAMP_HOME
    Worker --> RAMP_HOME
    API --> TRAINING_WORKSPACE
    Worker --> TRAINING_WORKSPACE
```

To set up the Docker-based development environment:

1. Clone the repository and navigate to the project directory
2. Install prerequisites:
   - Docker and Docker Compose
   - NVIDIA Container Toolkit (for GPU support)
3. Set up RAMP base model:
   ```bash
   bash setup-ramp.sh
   ```
4. Configure environment variables:
   - Create `.env` file in the backend directory using `docker_sample_env` as a template
   - Create `.env` file in the frontend directory using `.env_sample` as a template
5. Build and start the containers:
   ```bash
   docker compose build
   docker compose up
   ```
6. Run database migrations:
   ```bash
   bash run_migrations.sh
   ```

The development environment will be available at:
- Frontend: http://127.0.0.1:3000
- Backend API: http://localhost:8000
- Flower dashboard: http://localhost:5500

Sources: [docs/Docker-installation.md](), [docker-compose.yml](), [backend/docker_sample_env](), [run_migrations.sh](), [setup-ramp.sh]()

### GPU vs CPU Configuration

The system supports both GPU and CPU configurations:

- For GPU support (recommended for model training):
  ```bash
  docker compose up
  ```

- For CPU-only environments:
  ```bash
  docker compose -f docker-compose-cpu.yml up
  ```

Note that model training performance will be significantly degraded without GPU acceleration.

Sources: [docker-compose.yml](), [docker-compose-cpu.yml](), [backend/Dockerfile](), [backend/Dockerfile_CPU]()

### Local Development Environment

For direct local development without Docker:

1. Install Python 3.8+, pip, and virtualenv
2. Set up a virtual environment:
   ```bash
   virtualenv env
   source ./env/bin/activate
   ```
3. Set up RAMP dependencies and basemodel:
   - Clone RAMP code repository
   - Download basemodel checkpoint
   - Install GDAL and other dependencies
4. Install required packages:
   ```bash
   pip install -r backend/requirements.txt
   pip install -r backend/api-requirements.txt
   ```
5. Configure environment variables in `.env` file
6. Start backend services:
   ```bash
   cd backend
   python manage.py runserver
   celery -A aiproject worker --loglevel=debug -Q ramp_training,yolo_training
   python manage.py qcluster
   ```
7. Start frontend development server:
   ```bash
   cd frontend
   npm install
   npm run dev
   ```

Sources: [backend/README.md](), [backend/api-requirements.txt](), [backend/sample_env]()

## CI/CD Workflows

The project uses GitHub Actions for continuous integration and deployment.

### CI/CD Pipeline Architecture

```mermaid
flowchart TD
    subgraph "Source Control"
        GitRepo["GitHub Repository"]
    end
    
    subgraph "Continuous Integration"
        BackendBuild["Backend Build Workflow<br>backend_build.yml"]
        FrontendBuild["Frontend Build Workflow<br>frontend_build.yml"]
        DockerBuild["Docker Build Workflow<br>docker_build.yml"]
    end
    
    subgraph "Continuous Deployment"
        DockerPublish["Docker Publish Workflow<br>docker_publish_image.yml"]
        FrontendDeploy["Frontend Deploy Workflow<br>frontend_build_push.yml"]
    end
    
    subgraph "Deployment Targets"
        GHCR["GitHub Container Registry"]
        S3Bucket["AWS S3 Bucket"]
    end
    
    GitRepo -- "Push/PR to master/develop<br>(backend/** changes)" --> BackendBuild
    GitRepo -- "Push/PR to master/develop<br>(frontend/** changes)" --> FrontendBuild
    GitRepo -- "Push/PR to master/develop<br>(backend/** changes)" --> DockerBuild
    
    GitRepo -- "Push to master/develop" --> DockerPublish
    GitRepo -- "Release created" --> FrontendDeploy
    
    DockerPublish --> GHCR
    FrontendDeploy --> S3Bucket
```

Sources: [.github/workflows/backend_build.yml](), [.github/workflows/frontend_build.yml](), [.github/workflows/docker_build.yml](), [.github/workflows/docker_publish_image.yml](), [.github/workflows/frontend_build_push.yml]()

### Backend Build Workflow

The backend build workflow:
1. Sets up a test environment with PostgreSQL and Redis
2. Installs dependencies including RAMP and GDAL
3. Runs database migrations
4. Executes backend tests

This workflow runs on pushes and pull requests to master/develop branches when backend files are changed.

Sources: [.github/workflows/backend_build.yml]()

### Frontend Build Workflow

The frontend build workflow:
1. Sets up Node.js environment (testing against multiple Node versions)
2. Installs dependencies with pnpm
3. Builds the frontend application

This workflow runs on pushes and pull requests to master/develop branches when frontend files are changed.

Sources: [.github/workflows/frontend_build.yml]()

### Docker Build and Publish

The Docker build workflow tests building the Docker images without publishing them.

The Docker publish workflow builds and publishes two container images to GitHub Container Registry:
1. API container (`Dockerfile.API`) - Backend API service
2. Worker container (`Dockerfile`) - Celery worker for asynchronous processing

This workflow runs on pushes to master/develop branches, and when releases are created.

Sources: [.github/workflows/docker_build.yml](), [.github/workflows/docker_publish_image.yml]()

### Frontend Deployment

Frontend deployment workflow:
1. Builds the frontend with production environment variables
2. Authenticates to AWS using OIDC
3. Syncs the built files to an S3 bucket for static hosting

This workflow runs when a release is created or manually triggered.

Sources: [.github/workflows/frontend_build_push.yml]()

## Docker Containerization

The system is containerized using Docker, with different containers for each component.

### Container Architecture

```mermaid
graph TD
    subgraph "Frontend Layer"
        Frontend["Frontend Container<br>React Application"]
    end
    
    subgraph "API Layer"
        API["API Container<br>Django REST API"]
    end
    
    subgraph "Worker Layer"
        Worker["Worker Container<br>Celery Worker"]
        Flower["Flower Dashboard<br>Worker Monitoring"]
    end
    
    subgraph "Data Layer"
        Redis["Redis<br>Message Broker"]
        PostgreSQL["PostgreSQL/PostGIS<br>Database"]
    end
    
    Frontend --> API
    API --> Redis
    API --> PostgreSQL
    Worker --> Redis
    Worker --> PostgreSQL
    Flower --> Redis
```

Sources: [docker-compose.yml](), [backend/Dockerfile](), [backend/Dockerfile.API](), [frontend/Dockerfile.frontend]()

### Container Configurations

#### API Container
- Base image: `tensorflow/tensorflow:2.9.2` (GPU or CPU variant)
- Includes Django, REST framework, and dependencies
- Handles HTTP requests and database operations
- Dockerfile: `backend/Dockerfile.API` (production) or `backend/Dockerfile`/`backend/Dockerfile_CPU` (development)

#### Worker Container
- Base image: `tensorflow/tensorflow:2.9.2-gpu` (for GPU support)
- Runs Celery worker for background tasks
- Handles model training and prediction
- Requires GPU capabilities for optimal performance
- Dockerfile: `backend/Dockerfile`

#### Frontend Container
- Base image: `node:22` (build) / `alpine:latest` (serving)
- Builds and serves React application
- Development mode runs with hot reloading
- Production mode builds static files
- Dockerfile: `frontend/Dockerfile.frontend`

Sources: [backend/Dockerfile](), [backend/Dockerfile_CPU](), [frontend/Dockerfile.frontend]()

### Development vs Production

Development containers:
- Include development dependencies
- Mount code volumes for live editing
- Run development servers with debugging
- Include Flower dashboard for Celery monitoring

Production containers:
- Optimized for performance and security
- API container runs with WSGI server
- Worker container optimized for background processing
- Frontend served as static files from S3

Sources: [docker-compose.yml](), [.github/workflows/docker_publish_image.yml]()

## Deployment Process

### Production Deployment Architecture

```mermaid
graph TD
    subgraph "User"
        Browser["Web Browser"]
    end
    
    subgraph "Frontend"
        S3["AWS S3 Bucket<br>Static Files"]
        CloudFront["CDN<br>(Optional)"]
    end
    
    subgraph "Backend Services"
        API["API Container<br>ghcr.io/hotosm/fair_api"]
        Worker["Worker Container<br>ghcr.io/hotosm/fair_worker"]
    end
    
    subgraph "Data Storage"
        Redis["Redis<br>Message Broker"]
        PostgreSQL["PostgreSQL/PostGIS<br>Database"]
        Storage["Object Storage<br>Model Files & Training Data"]
    end
    
    Browser --> S3
    Browser --> CloudFront
    CloudFront --> S3
    Browser --> API
    API --> Redis
    API --> PostgreSQL
    API --> Storage
    Worker --> Redis
    Worker --> PostgreSQL
    Worker --> Storage
```

Sources: [.github/workflows/frontend_build_push.yml](), [.github/workflows/docker_publish_image.yml]()

### Frontend Deployment

The frontend is deployed as static files to AWS S3:

1. GitHub Actions workflow builds the frontend with production environment variables
2. Built files are uploaded to an S3 bucket
3. S3 bucket is configured for static website hosting
4. Optionally, a CDN like CloudFront can be configured for improved performance

Required environment variables for frontend build:
- `VITE_BASE_API_URL`: Backend API URL
- `VITE_MATOMO_ID`: Analytics ID (optional)
- `VITE_MATOMO_APP_DOMAIN`: Analytics domain (optional)
- `VITE_OSM_HASHTAGS`: OSM hashtags for changesets
- `VITE_FAIR_PREDICTOR_API_URL`: Prediction API URL

Sources: [.github/workflows/frontend_build_push.yml]()

### Backend Deployment

Backend deployment uses Docker containers:

1. GitHub Actions workflow builds and publishes container images to GitHub Container Registry
2. Images are tagged based on Git branch/tag
3. API and Worker containers are deployed to target infrastructure
4. Containers connect to production database and Redis instances

Required environment variables for backend deployment:
- Database connection details
- Redis connection details
- OSM authentication credentials
- Storage configuration
- RAMP and training workspace paths

Sources: [.github/workflows/docker_publish_image.yml](), [backend/docker_sample_env](), [backend/sample_env]()

### Environment Configuration

Production environment should be configured with:

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL database connection string | `postgis://user:password@host:port/dbname` |
| `CELERY_BROKER_URL` | Redis connection for Celery broker | `redis://host:port/0` |
| `CELERY_RESULT_BACKEND` | Redis connection for Celery results | `redis://host:port/0` |
| `OSM_CLIENT_ID` | OpenStreetMap OAuth client ID | `your-osm-client-id` |
| `OSM_CLIENT_SECRET` | OpenStreetMap OAuth client secret | `your-osm-client-secret` |
| `OSM_LOGIN_REDIRECT_URI` | OpenStreetMap OAuth redirect URI | `https://your-domain/authenticate/` |
| `RAMP_HOME` | Path to RAMP base model | `/path/to/ramp` |
| `TRAINING_WORKSPACE` | Path to training workspace | `/path/to/training` |
| `CORS_ALLOWED_ORIGINS` | Allowed origins for CORS | `https://your-frontend-domain` |

Sources: [backend/sample_env](), [backend/docker_sample_env]()

## Testing

### Running Backend Tests

Backend tests can be run locally:

```bash
cd backend
python manage.py test tests
```

The test suite includes:
- API endpoint tests
- Model tests
- Authentication tests
- Task processing tests

Tests are automatically run as part of the CI pipeline.

Sources: [backend/tests/test_endpoints.py](), [backend/tests/test_views.py](), [.github/workflows/backend_build.yml]()