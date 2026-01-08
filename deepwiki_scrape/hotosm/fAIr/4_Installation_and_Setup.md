# Installation and Setup

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/backend_build.yml](.github/workflows/backend_build.yml)
- [.gitignore](.gitignore)
- [Readme.md](Readme.md)
- [backend/Dockerfile](backend/Dockerfile)
- [backend/Dockerfile_CPU](backend/Dockerfile_CPU)
- [backend/README.md](backend/README.md)
- [backend/aiproject/__init__.py](backend/aiproject/__init__.py)
- [backend/aiproject/asgi.py](backend/aiproject/asgi.py)
- [backend/aiproject/celery.py](backend/aiproject/celery.py)
- [backend/aiproject/utils.py](backend/aiproject/utils.py)
- [backend/aiproject/wsgi.py](backend/aiproject/wsgi.py)
- [backend/api-requirements.txt](backend/api-requirements.txt)
- [backend/docker_sample_env](backend/docker_sample_env)
- [backend/sample_env](backend/sample_env)
- [backend/tests/__init__.py](backend/tests/__init__.py)
- [backend/tests/test_endpoints.py](backend/tests/test_endpoints.py)
- [backend/tests/test_views.py](backend/tests/test_views.py)
- [docker-compose-cpu.yml](docker-compose-cpu.yml)
- [docker-compose.yml](docker-compose.yml)
- [docs/Docker-installation.md](docs/Docker-installation.md)
- [frontend/.env.sample](frontend/.env.sample)
- [frontend/Dockerfile.frontend](frontend/Dockerfile.frontend)
- [frontend/src/config/env.ts](frontend/src/config/env.ts)
- [frontend/src/config/index.ts](frontend/src/config/index.ts)
- [run_migrations.sh](run_migrations.sh)
- [setup-ramp.sh](setup-ramp.sh)

</details>



This document provides comprehensive instructions for setting up the fAIr AI-assisted mapping system in a development environment. It covers both Docker-based and native installation methods, along with the necessary configuration steps. For information about the overall architecture, refer to [System Architecture](#1.1).

## System Components

Before diving into installation steps, it's important to understand the main components of the fAIr system:

```mermaid
graph TD
    subgraph "User Interface"
        Frontend["React Frontend\n(Port 3000)"]
    end
    
    subgraph "Application Servers"
        BackendAPI["Django Backend API\n(Port 8000)"]
        WorkerProcess["Celery Worker\n(AI Training)"]
        FlowerDashboard["Flower Dashboard\n(Port 5500)"]
    end
    
    subgraph "Data Services"
        PostgreSQL[(PostgreSQL/PostGIS\n(Port 5432/5434))]
        Redis[(Redis\n(Port 6379))]
    end
    
    subgraph "External Dependencies"
        RAMP["RAMP Base Model"]
        OSM["OpenStreetMap API"]
    end
    
    Frontend --> BackendAPI
    BackendAPI --> PostgreSQL
    BackendAPI --> Redis
    BackendAPI --> OSM
    Redis --> WorkerProcess
    WorkerProcess --> RAMP
    FlowerDashboard --> Redis
```

Sources: [Readme.md:80-93](), [docker-compose.yml]()

## Prerequisites

### Hardware Requirements

- **Recommended**: NVIDIA GPU with appropriate drivers for model training
- **CPU-only**: Possible but not recommended for production use

### Software Requirements

- Git
- Docker and Docker Compose (for Docker installation)
- Python 3.8+ (for native installation)
- PostgreSQL with PostGIS extension
- Redis server
- Node.js and npm/pnpm (for frontend development)

### External Accounts

- OpenStreetMap account (for authentication)

Sources: [Readme.md:22-24](), [docs/Docker-installation.md:20-27](), [backend/README.md:7-13]()

## Docker Installation (Recommended)

Docker provides the easiest way to set up the development environment with all necessary components.

### 1. Clone the Repository

```bash
git clone https://github.com/hotosm/fAIr.git
cd fAIr
```

### 2. Set Up RAMP Base Model

The RAMP (Rapide Assessment Mapping Protocol) base model is required for training AI models.

```mermaid
graph LR
    subgraph "RAMP Setup Process"
        A["Create RAMP Directory"] --> B["Download Base Model"]
        B --> C["Clone RAMP Code"]
        C --> D["Extract Checkpoint"]
        D --> E["Set Environment Variables"]
    end
```

Execute the following commands:

```bash
mkdir ramp
gdown --fuzzy https://drive.google.com/file/d/1YQsY61S_rGfJ_f6kLQq4ouYE2l3iRe1k/view
git clone https://github.com/kshitijrajsharma/ramp-code-fAIr.git ramp-code
unzip checkpoint.tf.zip -d ramp-code/ramp
export RAMP_HOME=/path/to/ramp
export TRAINING_WORKSPACE=/path/to/fAIr/trainings
```

Alternatively, you can use the provided setup script:

```bash
bash setup-ramp.sh
source .env
```

Sources: [docs/Docker-installation.md:31-64](), [setup-ramp.sh]()

### 3. Register OAuth Application with OpenStreetMap

1. Log in to your OpenStreetMap account at https://www.openstreetmap.org/
2. Go to "My Settings" > "OAuth2 Applications"
3. Register a new application:
   - Name: "fAIr Dev Local"
   - Redirect URI: http://127.0.0.1:3000/authenticate/
   - Permissions: "Read user preferences"
4. Save the generated `OSM_CLIENT_ID` and `OSM_CLIENT_SECRET` for later use

Sources: [docs/Docker-installation.md:65-74]()

### 4. Configure Environment Variables

Create backend environment file:

```bash
cd backend
cp docker_sample_env .env
```

Edit the `.env` file to add your OSM credentials and set a random string for `OSM_SECRET_KEY`.

Create frontend environment file:

```bash
cd ../frontend
cp .env.sample .env
```

Sources: [docs/Docker-installation.md:75-92](), [backend/docker_sample_env](), [frontend/.env.sample]()

### 5. Build and Start Docker Containers

```bash
docker compose build
docker compose up
```

For CPU-only systems:

```bash
docker compose -f docker-compose-cpu.yml build
docker compose -f docker-compose-cpu.yml up
```

Sources: [docs/Docker-installation.md:96-102](), [docker-compose.yml](), [docker-compose-cpu.yml]()

### 6. Run Database Migrations

In a new terminal window:

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

Sources: [docs/Docker-installation.md:104-125](), [run_migrations.sh]()

### 7. Access the Application

- Frontend: http://127.0.0.1:3000
- Backend API: http://localhost:8000/api/v1/
- Flower Dashboard: http://localhost:5500

**Note**: Use `127.0.0.1:3000` instead of `localhost:3000` to ensure login functionality works correctly.

Sources: [docs/Docker-installation.md:134-136]()

## Native Installation

If you prefer not to use Docker, you can install and configure each component separately.

### 1. Backend Setup

#### Install Python and Virtual Environment

```bash
sudo apt-get install python3 python3-pip
sudo apt install python3-virtualenv
virtualenv env
source ./env/bin/activate
```

#### Install GDAL and Dependencies

```bash
sudo apt-get update
sudo add-apt-repository ppa:ubuntugis/ppa
sudo apt-get update
sudo apt-get install gdal-bin libgdal-dev python3-gdal
sudo apt-get install python3-psycopg2
sudo apt install redis
pip install numpy==1.23.5
pip install GDAL==$(gdal-config --version) --global-option=build_ext --global-option="-I/usr/include/gdal"
```

#### Set Up RAMP Base Model

Follow the same RAMP setup steps as in the Docker installation section.

#### Install Backend Dependencies

```bash
cd backend
pip install -r requirements.txt
pip install -r api-requirements.txt
```

#### Configure Environment Variables

```bash
cp sample_env .env
```

Edit the `.env` file to set the following variables:

```
DATABASE_URL=postgis://admin:password@localhost:5432/ai
RAMP_HOME=/path/to/ramp
TRAINING_WORKSPACE=/path/to/training/workspace
OSM_CLIENT_ID=your_client_id
OSM_CLIENT_SECRET=your_client_secret
OSM_SECRET_KEY=your_secret_key
```

#### Set Up PostgreSQL with PostGIS

```bash
sudo apt-get install postgresql postgresql-contrib postgis
sudo -u postgres createuser -P admin
sudo -u postgres createdb -O admin ai
sudo -u postgres psql -d ai -c "CREATE EXTENSION postgis;"
```

#### Run Migrations

```bash
python manage.py makemigrations
python manage.py makemigrations login
python manage.py makemigrations core
python manage.py migrate
```

#### Start Celery Workers

```bash
celery -A aiproject worker --loglevel=debug -Q ramp_training,yolo_training
```

#### Start Flower Dashboard (Optional)

```bash
celery -A aiproject --broker=redis://localhost:6379/ flower
```

#### Start Django Server

```bash
python manage.py runserver
```

Sources: [backend/README.md](), [backend/sample_env](), [backend/api-requirements.txt]()

### 2. Frontend Setup

#### Install Node.js and Dependencies

```bash
cd frontend
npm install
```

#### Configure Environment Variables

```bash
cp .env.sample .env
```

Edit the `.env` file to point to your backend API:

```
VITE_BASE_API_URL=http://localhost:8000/api/v1/
```

#### Start Development Server

```bash
npm run dev
```

Sources: [frontend/.env.sample](), [frontend/src/config/env.ts](), [frontend/src/config/index.ts]()

## Environment Variables Reference

### Backend Environment Variables

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `DEBUG` | Enables debug mode | `True` |
| `SECRET_KEY` | Django secret key | `random_string` |
| `DATABASE_URL` | PostgreSQL connection URL | `postgis://admin:password@localhost:5432/ai` |
| `CORS_ALLOWED_ORIGINS` | Allowed CORS origins | `http://localhost:3000` |
| `OSM_CLIENT_ID` | OSM OAuth client ID | `your_client_id` |
| `OSM_CLIENT_SECRET` | OSM OAuth client secret | `your_client_secret` |
| `OSM_SECRET_KEY` | Secret key for OSM integration | `random_string` |
| `RAMP_HOME` | Path to RAMP directory | `/path/to/ramp` |
| `TRAINING_WORKSPACE` | Path to training workspace | `/path/to/trainings` |
| `CELERY_BROKER_URL` | Redis URL for Celery | `redis://localhost:6379/0` |

Sources: [backend/sample_env](), [backend/docker_sample_env]()

### Frontend Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `VITE_BASE_API_URL` | Backend API URL | `http://localhost:8000/api/v1/` |
| `VITE_FAIR_PREDICTOR_API_URL` | Model prediction API URL | `https://predictor-dev.fair.hotosm.org/predict/` |
| `VITE_MAX_TRAINING_AREA_SIZE` | Maximum training area size (sq m) | `5000000` |
| `VITE_MIN_TRAINING_AREA_SIZE` | Minimum training area size (sq m) | `5797` |
| `VITE_MAX_ZOOM_LEVEL` | Maximum map zoom level | `22` |
| `VITE_MIN_ZOOM_LEVEL_FOR_START_MAPPING_PREDICTION` | Min zoom for predictions | `19` |

Sources: [frontend/.env.sample](), [frontend/src/config/env.ts](), [frontend/src/config/index.ts:46-51]()

## Verification and Testing

### Backend Testing

To verify that the backend is working properly, run the included tests:

```bash
cd backend
python manage.py test tests
```

Sources: [backend/README.md:180-182](), [backend/tests/test_endpoints.py](), [backend/tests/test_views.py]()

### API Documentation

The API documentation is available at the Swagger UI endpoint:

```
http://localhost:8000/api/v1/docs/
```

### Testing Authentication

To test the authentication flow:

1. Visit `http://localhost:8000/api/v1/auth/login/` to get the login URL
2. Navigate to the login URL and authenticate with OSM
3. After successful authentication, check if you can access `http://localhost:8000/api/v1/auth/me/`

Sources: [backend/README.md:146-155]()

## Troubleshooting

### GPU Issues

If you encounter GPU-related errors, ensure:

1. NVIDIA drivers are properly installed
2. NVIDIA Container Toolkit is installed (for Docker)
3. GPU is correctly detected by running `nvidia-smi`

### Database Connection Issues

If you encounter database connection errors:

1. Check that PostgreSQL is running
2. Verify the `DATABASE_URL` is correct
3. Ensure the PostGIS extension is installed in the database

### Authorization Issues

If OpenStreetMap authentication is not working:

1. Verify that your OAuth credentials are correct
2. Ensure the redirect URI exactly matches what you registered with OSM
3. Use `127.0.0.1` instead of `localhost` in the URLs

Sources: [docs/Docker-installation.md:20-27](), [backend/README.md:146-155]()

## System Testing

After completing the installation, you should be able to:

1. Log in with your OSM account
2. Create a model
3. Define a training area
4. Fetch OSM data
5. Start model training
6. View predictions on the map

If any of these steps fail, check the relevant logs:

- Backend logs: `docker logs api`
- Worker logs: `docker logs worker`
- Frontend logs: Browser console

Sources: [Readme.md:72-79]()