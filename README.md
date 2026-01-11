# CI/CD Demo

A simple Flask web application demonstrating a complete CI/CD pipeline with automated testing, Docker containerization, and deployment.

## What This Project Does

This repository contains a minimal Flask application that serves as a learning project for understanding Continuous Integration and Continuous Deployment (CI/CD) workflows. The application includes:

- **Flask Web Application**: A simple Python web app with two endpoints:
  - `/` - Home route returning a greeting
  - `/health` - Health check endpoint for monitoring

- **Docker Containerization**: The app is containerized using Docker with:
  - Multi-stage optimization
  - Non-root user for security
  - Production-ready Gunicorn server

- **Automated CI/CD Pipeline**: GitHub Actions workflow that:
  - Builds Docker images on every push to `main`
  - Pushes images to Docker Hub
  - Automatically deploys to a server via SSH
  - Performs health checks after deployment

## Features

- ✅ Automated testing (pytest)
- ✅ Docker containerization
- ✅ CI/CD with GitHub Actions
- ✅ Automated deployment to production server
- ✅ Health check monitoring
- ✅ Docker image caching for faster builds

## Project Structure

```
ci-cd-demo/
├── app.py                 # Flask application
├── requirements.txt       # Python dependencies
├── Dockerfile            # Docker image configuration
├── .dockerignore         # Files excluded from Docker build
├── .github/
│   └── workflows/
│       └── deploy.yml    # GitHub Actions CI/CD pipeline
└── README.md             # This file
```

## Local Development

### Prerequisites

- Python 3.9+
- Docker (optional, for containerized runs)

### Setup

1. Clone the repository:
```bash
git clone <repository-url>
cd ci-cd-demo
```

2. Create a virtual environment:
```bash
python -m venv env
source env/bin/activate  # On Windows: env\Scripts\activate
```

3. Install dependencies:
```bash
pip install -r requirements.txt
```

4. Run the application:
```bash
python app.py
```

The app will be available at `http://localhost:5000`

### Running Tests

```bash
pip install pytest pytest-cov
pytest
```

## Docker

### Build the Image

```bash
docker build -t silverlake:latest .
```

### Run the Container

```bash
docker run -d -p 5000:5000 --name silverlake silverlake:latest
```

### Push to Docker Hub

```bash
docker tag silverlake:latest <your-username>/silverlake:latest
docker push <your-username>/silverlake:latest
```

## CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/deploy.yml`) automatically:

1. **Builds** the Docker image when code is pushed to `main`
2. **Pushes** the image to Docker Hub with tags:
   - `latest` - Always points to the most recent build
   - `{git-sha}` - Specific commit SHA for versioning
3. **Deploys** to the production server via SSH:
   - Pulls the latest image
   - Stops and removes the old container
   - Starts a new container
   - Performs a health check
   - Cleans up unused images

### Required Secrets

Configure these secrets in your GitHub repository settings:

- `DOCKERHUB_USERNAME` - Your Docker Hub username
- `DOCKERHUB_TOKEN` - Docker Hub access token
- `SERVER_HOST` - Production server hostname/IP
- `SERVER_USER` - SSH username for deployment
- `SERVER_SSH_KEY` - Private SSH key for server access

## Technologies Used

- **Flask** - Python web framework
- **Gunicorn** - Production WSGI server
- **Docker** - Containerization
- **GitHub Actions** - CI/CD automation
- **pytest** - Testing framework

## Learning Objectives

This project demonstrates:

- Setting up a CI/CD pipeline from scratch
- Docker containerization best practices
- Automated deployment workflows
- Health check monitoring
- GitHub Actions configuration
- Testing integration in CI/CD

## License

This is a demo/learning project. Feel free to use it as a reference for your own projects.
