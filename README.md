# Airflow ML Training Pipeline

This repository contains an Apache Airflow setup that orchestrates ML model training on AWS EC2 instances. The pipeline automatically creates EC2 instances, runs MLflow training jobs, and cleans up resources after completion.

## Overview

The Airflow DAG (`github_ec2_ml_training`) automates the following workflow:

1. **Wait for GitHub CI** - Polls GitHub Actions to ensure CI passes before proceeding
2. **Create EC2 Instance** - Provisions an EC2 instance with Docker and MLflow pre-installed
3. **Check EC2 Status** - Waits for the instance to pass all status checks
4. **Get Public IP** - Retrieves the public IP address of the instance
5. **Run Training via SSH** - Connects via SSH and executes MLflow training from a GitHub repository
6. **Terminate EC2 Instance** - Cleans up the EC2 instance (runs even if training fails)

## Architecture

The project uses Docker Compose to run a complete Airflow stack:

- **PostgreSQL** - Database for Airflow metadata
- **Redis** - Message broker for Celery executor
- **Airflow API Server** - REST API and web UI (port 8080)
- **Airflow Scheduler** - Schedules and triggers DAG runs
- **Airflow DAG Processor** - Processes DAG files
- **Airflow Worker** - Executes tasks using Celery
- **Airflow Triggerer** - Handles deferred tasks and triggers
- **Ngrok** (optional) - Provides public tunnel to Airflow UI

## Prerequisites

- Docker and Docker Compose installed
- AWS account with EC2 permissions
- GitHub Personal Access Token (PAT) with repo access
- SSH key pair for EC2 access

## Setup

### 1. Environment Variables

Create a `.env` file in the root directory with the following variables:

```bash
# GitHub Configuration
GITHUB_REPO=your-org/your-repo          # GitHub repository (e.g., username/repo-name)
GITHUB_PAT=your_github_personal_token    # GitHub Personal Access Token

# AWS Configuration
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_DEFAULT_REGION=eu-west-3             # AWS region (default: eu-west-3)
KEY_PAIR_NAME=your-key-pair-name         # EC2 key pair name
AMI_ID=ami-00ac45f3035ff009e             # AMI ID (default provided)
SECURITY_GROUP_ID=sg-xxxxxxxxx           # Security group ID
INSTANCE_TYPE=t3.micro                    # EC2 instance type (default: t3.micro)

# SSH Configuration
SSH_PRIVATE_KEY_CONTENT="-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----"           # Full SSH private key content

# MLflow Configuration
MLFLOW_TRACKING_URI=your_mlflow_uri      # MLflow tracking server URI
MLFLOW_EXPERIMENT_NAME=california_housing # MLflow experiment name (default: california_housing)

# Airflow Configuration (optional)
AIRFLOW_UID=50000                         # User ID for Airflow containers
_AIRFLOW_WWW_USER_USERNAME=airflow       # Airflow web UI username (default: airflow)
_AIRFLOW_WWW_USER_PASSWORD=airflow       # Airflow web UI password (default: airflow)

# Ngrok Configuration (optional)
NGROK_AUTHTOKEN=your_ngrok_token         # Required if using ngrok service
```

### 2. Initialize Airflow

Before starting the services, set the Airflow user ID (Linux/Mac):

```bash
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

### 3. Launch Airflow with Docker Compose

Start all services:

```bash
docker-compose up -d
```

This will start:
- PostgreSQL database
- Redis broker
- All Airflow services (scheduler, webserver, worker, etc.)
- Ngrok tunnel (if configured)

### 4. Access Airflow Web UI

Once the services are running, access the Airflow web UI at:

- **Local**: http://localhost:8080
- **Via Ngrok**: Check http://localhost:4040 for the ngrok web interface to get the public URL

Default credentials:
- Username: `airflow`
- Password: `airflow`

### 5. Monitor Services

Check service status:

```bash
docker-compose ps
```

View logs:

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f airflow-scheduler
docker-compose logs -f airflow-worker
```

### 6. Stop Services

Stop all services:

```bash
docker-compose down
```

Stop and remove volumes (⚠️ this will delete the database):

```bash
docker-compose down -v
```

## DAG Execution

1. **Enable the DAG**: In the Airflow UI, toggle the `github_ec2_ml_training` DAG to enable it
2. **Trigger a Run**: Click the "Play" button to trigger a manual DAG run
3. **Monitor Progress**: Watch the task execution in the Graph or Tree view
4. **View Logs**: Click on any task to view detailed execution logs

## Project Structure

```
.
├── dags/
│   └── ml_training_pipeline.py    # Main DAG definition
├── config/
│   └── airflow.cfg                # Airflow configuration
├── logs/                          # Airflow task logs
├── plugins/                       # Custom Airflow plugins
├── docker-compose.yaml            # Docker Compose configuration
├── requirements.txt               # Python dependencies
└── .env                           # Environment variables (create this)
```

## Troubleshooting

### Services won't start
- Ensure Docker has enough resources (4GB RAM, 2 CPUs minimum)
- Check that ports 8080, 5432, 6379 are not already in use
- Verify `.env` file exists and contains required variables

### DAG not appearing
- Check `docker-compose logs -f airflow-scheduler` for DAG parsing errors
- Verify the DAG file is in the `dags/` directory
- Ensure Python syntax is correct

### EC2 tasks failing
- Verify AWS credentials are correct
- Check that the security group allows SSH (port 22)
- Ensure the key pair exists in the specified region
- Verify SSH_PRIVATE_KEY_CONTENT matches your EC2 key pair

### MLflow training failures
- Check SSH connection logs in the task output
- Verify MLFLOW_TRACKING_URI is accessible from EC2
- Ensure the GitHub repository contains a valid MLflow project

## Additional Resources

- [Airflow Documentation](https://airflow.apache.org/docs/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [MLflow Documentation](https://www.mlflow.org/docs/latest/index.html)
