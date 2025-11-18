---
layout: default
title:  "Docker Best Practices for ML Applications"
date:   2025-11-18 12:00:00
categories: DevOps Docker MachineLearning Containers
---

Building efficient Docker images for ML applications requires special considerations for model size, GPU support, and reproducibility.

## Base Image Selection

### NVIDIA CUDA Images

```dockerfile
# For GPU training
FROM nvidia/cuda:12.1-devel-ubuntu22.04 AS builder

# For GPU inference (smaller)
FROM nvidia/cuda:12.1-runtime-ubuntu22.04 AS runtime

# For CPU-only
FROM python:3.11-slim
```

### Multi-Stage for Size Optimization

```dockerfile
# Build stage with development tools
FROM nvidia/cuda:12.1-devel-ubuntu22.04 AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3-dev \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip3 install --no-cache-dir --prefix=/install -r requirements.txt

# Runtime stage
FROM nvidia/cuda:12.1-runtime-ubuntu22.04

# Copy only runtime dependencies
COPY --from=builder /install /usr/local

# Copy application
COPY --chown=1000:1000 src/ /app/src/
COPY --chown=1000:1000 models/ /app/models/

WORKDIR /app
USER 1000:1000

CMD ["python3", "src/serve.py"]
```

## Dependency Management

### Requirements with Pinned Versions

```dockerfile
# requirements.txt - production
torch==2.1.0
transformers==4.35.0
fastapi==0.104.1
uvicorn==0.24.0
numpy==1.24.3

# requirements-dev.txt - development only
pytest==7.4.3
black==23.11.0
mypy==1.7.0
```

### UV for Fast Dependency Resolution

```dockerfile
FROM python:3.11-slim

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Copy dependency files
COPY pyproject.toml uv.lock ./

# Install dependencies (cached layer)
RUN uv sync --frozen --no-dev

# Copy source code
COPY src/ src/

CMD ["uv", "run", "python", "src/main.py"]
```

### Poetry with Cache Mount

```dockerfile
FROM python:3.11-slim

RUN pip install poetry

WORKDIR /app

COPY pyproject.toml poetry.lock ./

# Use cache mount for faster builds
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=cache,target=/root/.cache/pypoetry \
    poetry config virtualenvs.create false && \
    poetry install --no-interaction --no-ansi --no-dev

COPY . .

CMD ["python", "main.py"]
```

## Model Handling

### Separate Model Layer

```dockerfile
FROM python:3.11-slim

# Base dependencies (rarely changes)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Model files (changes with model updates)
COPY models/ /app/models/

# Source code (changes frequently)
COPY src/ /app/src/

CMD ["python", "/app/src/serve.py"]
```

### Download Models at Build Time

```dockerfile
FROM python:3.11-slim

RUN pip install huggingface-hub

# Download model during build
ARG MODEL_NAME=bert-base-uncased
RUN python -c "from huggingface_hub import snapshot_download; \
    snapshot_download('${MODEL_NAME}', local_dir='/models/${MODEL_NAME}')"

COPY src/ /app/

ENV MODEL_PATH=/models/${MODEL_NAME}
CMD ["python", "/app/serve.py"]
```

### Runtime Model Loading

```dockerfile
FROM python:3.11-slim

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ /app/

# Download models at runtime
ENV HF_HOME=/models
VOLUME /models

CMD ["python", "/app/serve.py"]
```

## Build Optimization

### Layer Caching Strategy

```dockerfile
FROM python:3.11-slim

# System dependencies (rarely change) - Layer 1
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Python dependencies (change occasionally) - Layer 2
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Configuration (change sometimes) - Layer 3
COPY config/ /app/config/

# Models (change with model updates) - Layer 4
COPY models/ /app/models/

# Source code (change frequently) - Layer 5
COPY src/ /app/src/

WORKDIR /app
CMD ["python", "src/main.py"]
```

### BuildKit Features

```dockerfile
# syntax=docker/dockerfile:1.4

FROM python:3.11-slim

# Cache mounts for package managers
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y --no-install-recommends \
    build-essential

# Secret mounts for credentials
RUN --mount=type=secret,id=pip_conf,target=/etc/pip.conf \
    pip install --no-cache-dir -r requirements.txt

# Heredoc for inline scripts
RUN <<EOF
#!/bin/bash
set -e
echo "Setting up environment"
python -m compileall /app
EOF

COPY . /app
```

### .dockerignore

```
# .dockerignore
.git
.gitignore
.github
__pycache__
*.pyc
*.pyo
*.egg-info
dist
build
.pytest_cache
.mypy_cache
*.md
!README.md
tests/
notebooks/
data/raw/
*.log
.env*
Dockerfile*
docker-compose*
```

## GPU Support

### Runtime Configuration

```yaml
# docker-compose.yml
services:
  ml-inference:
    build: .
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
```

### NVIDIA Container Runtime

```dockerfile
FROM nvidia/cuda:12.1-runtime-ubuntu22.04

# Install Python and dependencies
RUN apt-get update && apt-get install -y python3 python3-pip

COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

# Verify GPU access
RUN python3 -c "import torch; print(torch.cuda.is_available())"

COPY . /app
CMD ["python3", "/app/serve.py"]
```

## Security

### Non-Root User

```dockerfile
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . /app

USER appuser
WORKDIR /app

CMD ["python", "main.py"]
```

### Read-Only Filesystem

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir -r requirements.txt

COPY . /app
WORKDIR /app

# Compile Python files
RUN python -m compileall .

USER 1000:1000

# Run with read-only root filesystem
CMD ["python", "main.py"]
```

```yaml
# docker-compose.yml
services:
  ml-app:
    build: .
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - model-cache:/app/cache
```

## Health Checks

```dockerfile
FROM python:3.11-slim

COPY . /app
WORKDIR /app

RUN pip install --no-cache-dir -r requirements.txt

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD python -c "import requests; r = requests.get('http://localhost:8080/health'); r.raise_for_status()"

EXPOSE 8080
CMD ["python", "serve.py"]
```

## Development vs Production

### Development Image

```dockerfile
# Dockerfile.dev
FROM python:3.11

RUN pip install poetry

WORKDIR /app

COPY pyproject.toml poetry.lock ./
RUN poetry install  # Include dev dependencies

COPY . .

# Hot reload for development
CMD ["poetry", "run", "uvicorn", "main:app", "--reload", "--host", "0.0.0.0"]
```

### Production Image

```dockerfile
# Dockerfile.prod
FROM python:3.11-slim AS builder

RUN pip install poetry

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --without-hashes > requirements.txt

FROM python:3.11-slim

COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ /app/

USER 1000:1000
CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "app.main:app"]
```

## Testing in Docker

### Test Stage

```dockerfile
# Multi-stage with test
FROM python:3.11-slim AS base
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS test
COPY requirements-dev.txt .
RUN pip install --no-cache-dir -r requirements-dev.txt
COPY . .
RUN pytest tests/

FROM base AS production
COPY src/ /app/
CMD ["python", "/app/main.py"]
```

### Build and Test Pipeline

```bash
#!/bin/bash
# build-and-test.sh

# Build test stage
docker build --target test -t ml-app:test .

# Run tests
docker run --rm ml-app:test pytest tests/ -v

# Build production if tests pass
docker build --target production -t ml-app:prod .
```

## Compose for ML Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  ml-training:
    build:
      context: .
      dockerfile: Dockerfile.train
    volumes:
      - ./data:/data
      - ./models:/models
      - ./src:/app/src
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - MLFLOW_TRACKING_URI=http://mlflow:5000
    depends_on:
      - mlflow

  ml-inference:
    build:
      context: .
      dockerfile: Dockerfile.serve
    ports:
      - "8080:8080"
    volumes:
      - ./models:/models:ro
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    ports:
      - "5000:5000"
    volumes:
      - mlflow-data:/mlflow
    command: mlflow server --host 0.0.0.0

volumes:
  mlflow-data:
```

## Best Practices Checklist

- [ ] Use multi-stage builds to reduce image size
- [ ] Pin all dependency versions
- [ ] Order layers from least to most frequently changed
- [ ] Run as non-root user
- [ ] Include health checks
- [ ] Use .dockerignore
- [ ] Leverage BuildKit cache mounts
- [ ] Scan images for vulnerabilities
- [ ] Sign images with cosign
- [ ] Use specific base image tags (not latest)

## Resources

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)
- [BuildKit Documentation](https://docs.docker.com/build/buildkit/)
- [Hadolint Dockerfile Linter](https://github.com/hadolint/hadolint)

---

*Questions about Docker for ML? [Let me know](mailto:jordan@jordananderson.us).*
