# Development Environment Setup

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.gitignore](.gitignore)
- [Readme.md](Readme.md)
- [backend/Dockerfile](backend/Dockerfile)
- [backend/Dockerfile_CPU](backend/Dockerfile_CPU)
- [backend/aiproject/__init__.py](backend/aiproject/__init__.py)
- [backend/aiproject/asgi.py](backend/aiproject/asgi.py)
- [backend/aiproject/celery.py](backend/aiproject/celery.py)
- [backend/aiproject/utils.py](backend/aiproject/utils.py)
- [backend/aiproject/wsgi.py](backend/aiproject/wsgi.py)
- [docker-compose-cpu.yml](docker-compose-cpu.yml)
- [docker-compose.yml](docker-compose.yml)
- [docs/Docker-installation.md](docs/Docker-installation.md)
- [frontend/.gitignore](frontend/.gitignore)
- [frontend/Dockerfile.frontend](frontend/Dockerfile.frontend)
- [frontend/README.md](frontend/README.md)
- [frontend/eslint.config.js](frontend/eslint.config.js)
- [frontend/index.html](frontend/index.html)
- [frontend/src/components/ui/animated-beam/animated-beam.tsx](frontend/src/components/ui/animated-beam/animated-beam.tsx)
- [frontend/src/components/ui/banner/banner.tsx](frontend/src/components/ui/banner/banner.tsx)
- [frontend/src/styles/index.css](frontend/src/styles/index.css)
- [run_migrations.sh](run_migrations.sh)
- [setup-ramp.sh](setup-ramp.sh)

</details>



This page provides comprehensive instructions for setting up a local development environment for the fAIr AI-assisted mapping system. This guide focuses specifically on configuring your local machine for developing and testing the fAIr application. For deployment instructions, see [Docker Containerization](#4.3) and for continuous integration workflows, see [CI/CD Workflows](#4.2).

## Prerequisites

Before setting up the fAIr development environment, ensure you have the following:

- 64-bit operating system (Linux, macOS, or Windows)
- At least 8GB RAM (16GB recommended)
- At least 20GB free disk space
- Git installed
- Docker and Docker Compose installed
- For optimal performance: NVIDIA GPU with appropriate drivers

## Development Environment Architecture

The fAIr development environment consists of several interconnected components running in Docker containers:

```mermaid
graph TD
    subgraph "Docker Environment"
        Frontend["Frontend Container<br>React/TypeScript"]
        BackendAPI["Backend API Container<br>Django"]
        Worker["Worker Container<br>Celery"]
        PostgreSQL["PostgreSQL/PostGIS<br>Database"]
        Redis["Redis<br>Message Broker"]
        Flower["Flower<br>Task Monitor"]
    end
    
    subgraph "External Components"
        RAMP["RAMP Home<br>AI Model Framework"]
        TrainingWS["Training Workspace<br>Model Training Data"]
    end
    
    subgraph "External Services"
        OSM["OpenStreetMap API<br>Auth & Data"]
        OAM["OpenAerialMap<br>Imagery"]
    end
    
    Frontend -->|"HTTP Requests"| BackendAPI
    BackendAPI -->|"Enqueue Tasks"| Redis
    Worker -->|"Process Tasks"| Redis
    BackendAPI -->|"Store Data"| PostgreSQL
    Worker -->|"Read/Write Data"| PostgreSQL
    Flower -->|"Monitor"| Redis
    
    BackendAPI -->|"Mount Volume"| RAMP
    Worker -->|"Mount Volume"| RAMP
    BackendAPI -->|"Mount Volume"| TrainingWS
    Worker -->|"Mount Volume"| TrainingWS
    
    BackendAPI -->|"Authentication"| OSM
    BackendAPI -->|"Fetch Imagery"| OAM
```

Sources: [docker-compose.yml:1-82](), [docker-compose-cpu.yml:1-76](), [Readme.md:72-84]()

## Setup Process

### 1. Clone the Repository

```bash
git clone https://github.com/hotosm/fAIr.git
cd fAIr
```

### 2. RAMP Setup

fAIr requires the RAMP (Remote sensing Analytics for Mapping in Python) framework, which includes baseline models. You can set it up automatically using the provided script:

```bash
bash setup-ramp.sh
```

Alternatively, follow these manual steps:

1. Create a directory for RAMP:
   ```bash
   mkdir ramp
   ```

2. Download the baseline model checkpoint:
   ```bash
   pip install gdown
   gdown --fuzzy https://drive.google.com/file/d/1YQsY61S_rGfJ_f6kLQq4ouYE2l3iRe1k/view
   ```

3. Clone the RAMP code:
   ```bash
   git clone https://github.com/kshitijrajsharma/ramp-code-fAIr.git ramp-code
   ```

4. Unzip the baseline model:
   ```bash
   unzip checkpoint.tf.zip -d ramp-code/ramp
   ```

5. Set environment variables:
   ```bash
   export RAMP_HOME=/absolute/path/to/ramp
   export TRAINING_WORKSPACE=/absolute/path/to/fAIr/trainings
   ```

Sources: [setup-ramp.sh:1-29](), [docs/Docker-installation.md:29-64]()

### 3. OpenStreetMap OAuth Setup

fAIr uses OpenStreetMap for authentication:

1. Go to [OpenStreetMap](https://www.openstreetmap.org/) and log in
2. Navigate to your profile → My Settings → OAuth2 Applications
3. Register a new application:
   - Name: "fAIr Dev Local"
   - Permissions: "Read user preferences"
   - Redirect URI: `http://127.0.0.1:3000/authenticate/`
4. Save the provided `OSM_CLIENT_ID` and `OSM_CLIENT_SECRET` for the next step

Sources: [docs/Docker-installation.md:66-73]()

### 4. Environment Configuration

#### Backend Environment

```bash
cd backend
cp docker_sample_env .env
```

Edit the `.env` file to include your OSM credentials and a secret key:

```
OSM_CLIENT_ID=your_client_id_here
OSM_CLIENT_SECRET=your_client_secret_here
OSM_SECRET_KEY=generate_a_random_key_here
```

#### Frontend Environment

```bash
cd frontend
cp .env.sample .env
```

For local development, the default values should work without modification.

Sources: [docs/Docker-installation.md:75-92](), [.gitignore:12-36]()

### 5. Docker Containers Setup

fAIr offers two Docker Compose configurations:

#### GPU Setup (Recommended)

For systems with an NVIDIA GPU:

```bash
# Verify GPU is recognized
nvidia-smi

# Build and start containers
docker compose build
docker compose up
```

Sources: [docker-compose.yml:1-82](), [backend/Dockerfile:1-39]()

#### CPU-only Setup

For systems without a compatible GPU:

```bash
docker compose -f docker-compose-cpu.yml build
docker compose -f docker-compose-cpu.yml up
```

Sources: [docker-compose-cpu.yml:1-76](), [backend/Dockerfile_CPU:1-41]()

### 6. Database Initialization

After starting the containers, run migrations to initialize the database:

```bash
bash run_migrations.sh
```

Or manually:

```bash
docker exec -it api bash -c "python manage.py makemigrations"
docker exec -it api bash -c "python manage.py makemigrations login"
docker exec -it api bash -c "python manage.py makemigrations core"
docker exec -it api bash -c "python manage.py migrate"
```

Sources: [run_migrations.sh:1-7](), [docs/Docker-installation.md:104-125]()

## Container Structure and Service Mapping

```mermaid
graph TD
    subgraph "Docker Service Architecture"
        Frontend["frontend<br>Port: 3000"]
        BackendAPI["backend-api<br>Port: 8000"]
        BackendWorker["backend-worker"]
        Postgres["postgres<br>Port: 5434"]
        Redis["redis<br>Port: 6379"]
        Flower["worker-dashboard<br>Port: 5500"]
    end
    
    subgraph "Volume Mounts"
        FrontendSrc["./frontend → /app"]
        BackendSrc["./backend → /app"]
        RampHome["RAMP_HOME → /RAMP_HOME"]
        TrainingWS["TRAINING_WORKSPACE → /TRAINING_WORKSPACE"]
        PostgresData["./postgres-data → /var/lib/postgresql/data"]
    end
    
    FrontendSrc -->|"mounted to"| Frontend
    BackendSrc -->|"mounted to"| BackendAPI
    BackendSrc -->|"mounted to"| BackendWorker
    RampHome -->|"mounted to"| BackendAPI
    RampHome -->|"mounted to"| BackendWorker
    TrainingWS -->|"mounted to"| BackendAPI
    TrainingWS -->|"mounted to"| BackendWorker
    PostgresData -->|"mounted to"| Postgres
    
    Frontend -->|"HTTP requests"| BackendAPI
    BackendAPI -->|"enqueue tasks"| Redis
    BackendWorker -->|"process tasks"| Redis
    BackendAPI -->|"DB access"| Postgres
    BackendWorker -->|"DB access"| Postgres
    Flower -->|"monitor tasks"| Redis
```

Sources: [docker-compose.yml:1-82](), [docker-compose-cpu.yml:1-76]()

## Frontend Development

The frontend is a React application built with TypeScript, Vite, and various UI libraries:

### Key Frontend Facts

- Port: 3000
- Development server: Runs with hot-reloading enabled
- Source location: `/frontend` directory

If you prefer to run the frontend outside Docker:

```bash
cd frontend
pnpm install
pnpm dev
```

The frontend will be available at `http://127.0.0.1:5173`.

Sources: [frontend/README.md:1-146](), [frontend/index.html:1-26]()

## Backend Development

The backend is a Django application with several key components:

### Key Backend Facts

- Port: 8000
- API: RESTful endpoints for frontend communication
- Database: PostgreSQL with PostGIS extension
- Asynchronous task queue: Celery with Redis broker

### Working with Celery Tasks

The backend uses Celery for asynchronous tasks like model training:

- Tasks are defined in the backend code
- Redis serves as the message broker
- Flower dashboard (port 5500) monitors task execution

If you make changes to task code, restart the worker:

```bash
docker compose restart backend-worker
```

Sources: [backend/aiproject/celery.py:1-30](), [backend/aiproject/__init__.py:1-6]()

## Accessing the Applications

After setup, access the applications at:

| Component | URL | Description |
|-----------|-----|-------------|
| Frontend | http://127.0.0.1:3000 | React application UI |
| Backend API | http://localhost:8000 | Django REST API |
| Flower Dashboard | http://localhost:5500 | Celery task monitor |

**Note**: Use `127.0.0.1:3000` rather than `localhost:3000` to ensure login functionality works correctly.

Sources: [docs/Docker-installation.md:135-136]()

## Troubleshooting

### Common Issues

1. **Docker containers fail to start**
   - Check if the required ports are already in use
   - Ensure Docker has sufficient resources allocated

2. **GPU not detected**
   - Run `nvidia-smi` to verify your GPU is recognized
   - Install NVIDIA Container Toolkit if missing

3. **Database migration errors**
   - Try removing the postgres-data directory and restarting containers
   - Run migrations manually step by step

4. **Frontend authentication issues**
   - Ensure you're using `127.0.0.1:3000` instead of `localhost:3000`
   - Check that OSM credentials are correctly configured

5. **Model training failures**
   - Verify RAMP environment is correctly set up
   - Check worker logs for specific errors:
     ```bash
     docker compose logs backend-worker
     ```

Sources: [docs/Docker-installation.md:20-28](), [.gitignore:1-55]()

## Custom TileServer Integration

To use local tiles for development:

1. Set up a local tile server using [titiler](https://github.com/developmentseed/titiler), [gdal2tiles](https://gdal.org/programs/gdal2tiles.html), or nginx
2. Modify `docker-compose.yml` to add `network_mode: "host"` to API and Worker services
3. Update backend `.env` to use localhost URLs:
   ```
   DATABASE_URL=postgis://postgres:admin@localhost:5434/ai
   CELERY_BROKER_URL="redis://localhost:6379/0"
   CELERY_RESULT_BACKEND="redis://localhost:6379/0"
   ```

Sources: [docs/Docker-installation.md:139-190]()

## Next Steps

After setting up your development environment, you can:

1. Create your first model using the web interface
2. Explore the codebase to understand the system architecture
3. Run tests to ensure everything is working properly

For more information on the CI/CD workflows, see [CI/CD Workflows](#4.2).