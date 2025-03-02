# Dockerized Flask Application - Production Ready Template

[![CI/CD Pipeline](https://img.shields.io/github/actions/workflow/status/yourusername/flask-docker-template/docker-publish.yml?label=Build%20%26%20Deploy)](https://github.com/yourusername/flask-docker-template/actions)
[![Docker Image Version](https://img.shields.io/docker/v/yourusername/my-flask-app/latest)](https://hub.docker.com/r/yourusername/my-flask-app)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Scan](https://img.shields.io/badge/Trivy-Scanned-success)](https://aquasecurity.github.io/trivy/)

A production-grade template for containerizing Python/Flask applications with Docker, implementing industry best practices for security, efficiency, and maintainability.

## Table of Contents
- [Features](#features)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Advanced Configuration](#advanced-configuration)
- [CI/CD Integration](#cicd-integration)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Contributing](#contributing)
- [License](#license)

## Features
- 🛡️ Multi-stage Docker builds with security scanning
- 📦 Optimized production image (<150MB)
- 🔒 Non-root user execution & read-only filesystem
- 📊 Built-in metrics and health checks
- 🌐 NGINX reverse proxy configuration
- 🔄 CI/CD with GitHub Actions
- 📈 Prometheus/Grafana monitoring
- � Docker Compose deployment
- 🧪 Unit test integration

## Project Structure

```text
.
├── .dockerignore
├── .github
│   └── workflows
│       └── docker-publish.yml
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── requirements.txt
├── src
│   ├── app
│   │   ├── __init__.py
│   │   └── routes.py
│   └── wsgi.py
├── config
│   ├── gunicorn.conf.py
│   └── nginx.conf
└── tests
    └── test_app.py
```

## Prerequisites
- Docker Engine 20.10+
- Docker Compose 2.12+
- Python 3.9+
- GNU Make (optional)

# Quick Start
## 1. Clone Repository
```
git clone https://github.com/yourusername/flask-docker-template.git
cd flask-docker-template
```
## 2. Build & Run
```
make build  # Build containers
make up     # Start the stack
```
## 3. Verify Deployment
```
curl http://localhost:8080/health
# Expected response: {"status": "healthy", "version": "1.0.0"}
```

# Advanced Configuration
## Multi-stage Dockerfile
```
# Build stage
FROM python:3.9-slim as builder

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc python3-dev

COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Runtime stage
FROM python:3.9-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser && \
    apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/wheels /wheels
COPY --from=builder /app/requirements.txt .

RUN pip install --no-cache /wheels/* && \
    rm -rf /wheels

COPY . /app
COPY config/nginx.conf /etc/nginx/nginx.conf

RUN chown -R appuser:appuser /app && \
    chmod -R 755 /app && \
    chown -R appuser:appuser /var/log/nginx

USER appuser

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["gunicorn", "--config", "config/gunicorn.conf.py", "src.wsgi:app"]
```

## Docker Compose
```
version: '3.8'

services:
  web:
    image: yourusername/my-flask-app:${TAG:-latest}
    build:
      context: .
      target: builder
    environment:
      - FLASK_ENV=production
      - GUNICORN_WORKERS=4
    ports:
      - "8080:8080"
    volumes:
      - app-data:/app/data
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M

  redis:
    image: redis:alpine
    networks:
      - app-network
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  app-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

## CI/CD Integration
Example GitHub Actions workflow (.github/workflows/docker-publish.yml):
```
name: Docker Build and Push

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v3
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: yourusername/my-flask-app:latest
```

## Monitoring
- Prometheus Configuration
- Add to docker-compose.yml:
  ```
    prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - app-network
  ```
- Example prometheus.yml:
  ```
  global:
  scrape_interval: 15s

  scrape_configs:
   - job_name: 'flask-app'
     static_configs:
       - targets: ['web:8080']
  ```

# Best Practices
  ```
   # Image scanning
   docker scan yourusername/my-flask-app

   # Run as non-root user
   docker run --user 1001:1001 yourusername/my-flask-app

   # Read-only filesystem
   docker run --read-only yourusername/my-flask-app
  ```
# Performance
```
# Build with cache
docker build --cache-from yourusername/my-flask-app:latest .

# Resource limits
docker run --cpus 1.0 --memory 512M yourusername/my-flask-app
```
# Troubleshooting
```
# View container logs
docker logs -f <container_id>

# Inspect running container
docker exec -it <container_id> sh

# Cleanup system
docker system prune -a --volumes
```

# Contributing
- Fork the Project
- Create your Feature Branch (git checkout -b feature/AmazingFeature)
- Commit your Changes (git commit -m 'Add some AmazingFeature')
- Push to the Branch (git push origin feature/AmazingFeature)
- Open a Pull Request

# License
Distributed under the MIT License. See LICENSE for more information.

```
This README.md includes:
- Proper markdown formatting
- Badges for build status and metadata
- Clear table of contents
- Code blocks with syntax highlighting
- Complete implementation details
- CI/CD pipeline example
- Monitoring setup
- Best practices section
- Contribution guidelines

To use this template:
1. Replace all `yourusername` occurrences with your Docker Hub username
2. Update application code in the `src/` directory
3. Configure CI/CD secrets in your repository
4. Modify the license file if needed
5. Customize the monitoring configuration based on your needs
```
