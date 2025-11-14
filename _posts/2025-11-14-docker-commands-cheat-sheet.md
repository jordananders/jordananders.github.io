---
layout: default
title:  "Docker Commands Cheat Sheet"
date:   2025-11-14 16:00:00
categories: Docker Development DevOps
---

I use Docker every day, but I still forget the exact syntax for certain commands. This is my personal cheat sheet—the commands I actually use, with real-world examples.

## Container Basics

### Run a Container

```bash
# Run container in foreground
docker run nginx

# Run in background (detached)
docker run -d nginx

# Run with name
docker run -d --name my-nginx nginx

# Run with port mapping
docker run -d -p 8080:80 nginx
# Access at http://localhost:8080

# Run with environment variables
docker run -d -e DATABASE_URL=postgres://localhost/db myapp

# Run with volume mount
docker run -d -v /host/path:/container/path nginx

# Run and remove after exit
docker run --rm alpine echo "Hello"

# Run interactively with shell
docker run -it ubuntu /bin/bash

# Run with resource limits
docker run -d --memory="512m" --cpus="1.0" nginx
```

### List Containers

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List with custom format
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"

# List container IDs only
docker ps -q
```

### Stop/Start/Restart Containers

```bash
# Stop container
docker stop container_name

# Stop all running containers
docker stop $(docker ps -q)

# Start stopped container
docker start container_name

# Restart container
docker restart container_name

# Kill container (force stop)
docker kill container_name
```

### Remove Containers

```bash
# Remove container
docker rm container_name

# Remove running container (force)
docker rm -f container_name

# Remove all stopped containers
docker container prune

# Remove all containers
docker rm $(docker ps -a -q)
```

## Container Inspection & Logs

### View Logs

```bash
# View container logs
docker logs container_name

# Follow logs (like tail -f)
docker logs -f container_name

# View last 100 lines
docker logs --tail 100 container_name

# View logs with timestamps
docker logs -t container_name

# View logs since specific time
docker logs --since 2025-01-01T00:00:00 container_name
```

### Execute Commands in Running Container

```bash
# Execute command
docker exec container_name ls -la

# Interactive shell
docker exec -it container_name /bin/bash

# Execute as specific user
docker exec -u root container_name whoami

# Execute with environment variable
docker exec -e MY_VAR=value container_name env
```

### Inspect Container

```bash
# View full container details
docker inspect container_name

# Get specific field (IP address)
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Get all environment variables
docker inspect --format='{{.Config.Env}}' container_name
```

### Container Stats

```bash
# View resource usage (CPU, memory)
docker stats

# View stats for specific container
docker stats container_name

# Stats with no streaming (one-time)
docker stats --no-stream
```

## Images

### Pull/Push Images

```bash
# Pull image from Docker Hub
docker pull nginx

# Pull specific version
docker pull nginx:1.21

# Pull from specific registry
docker pull gcr.io/my-project/my-image

# Tag image
docker tag nginx:latest mynginx:v1

# Push to registry
docker push myusername/myimage:tag

# Push to private registry
docker push registry.example.com/myimage:tag
```

### List Images

```bash
# List all images
docker images

# List with digests
docker images --digests

# List images by repository
docker images nginx

# List image IDs only
docker images -q
```

### Build Images

```bash
# Build from Dockerfile in current directory
docker build -t myapp:latest .

# Build with specific Dockerfile
docker build -t myapp:latest -f Dockerfile.prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp .

# Build without cache
docker build --no-cache -t myapp .

# Build and tag multiple tags
docker build -t myapp:latest -t myapp:1.0 .
```

### Remove Images

```bash
# Remove image
docker rmi image_name

# Remove image by ID
docker rmi abc123

# Remove all unused images
docker image prune

# Remove all images
docker rmi $(docker images -q)

# Remove dangling images (untagged)
docker image prune -a
```

## Dockerfile Examples

### Basic Node.js App

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Run application
CMD ["node", "server.js"]
```

### Multi-Stage Build (Smaller Final Image)

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Python App with Dependencies

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy requirements first (layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Run as non-root user
RUN useradd -m appuser
USER appuser

EXPOSE 8000
CMD ["python", "app.py"]
```

## Docker Compose

### Basic docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    environment:
      - NGINX_HOST=example.com
    networks:
      - mynetwork

  database:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: example
      POSTGRES_DB: mydb
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - mynetwork

networks:
  mynetwork:

volumes:
  db-data:
```

### Docker Compose Commands

```bash
# Start services
docker compose up

# Start in background
docker compose up -d

# Start specific service
docker compose up web

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# View logs
docker compose logs

# Follow logs
docker compose logs -f

# View logs for specific service
docker compose logs web

# List services
docker compose ps

# Execute command in service
docker compose exec web /bin/bash

# Rebuild services
docker compose up --build

# Scale service
docker compose up --scale web=3
```

## Volumes

### Manage Volumes

```bash
# Create volume
docker volume create my-volume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-volume

# Remove volume
docker volume rm my-volume

# Remove all unused volumes
docker volume prune

# Use volume with container
docker run -d -v my-volume:/data nginx
```

### Bind Mounts vs Volumes

```bash
# Bind mount (host path to container)
docker run -d -v /host/path:/container/path nginx

# Named volume
docker run -d -v my-volume:/container/path nginx

# Anonymous volume
docker run -d -v /container/path nginx

# Read-only mount
docker run -d -v /host/path:/container/path:ro nginx
```

## Networks

### Manage Networks

```bash
# Create network
docker network create my-network

# List networks
docker network ls

# Inspect network
docker network inspect my-network

# Remove network
docker network rm my-network

# Connect container to network
docker network connect my-network container_name

# Disconnect from network
docker network disconnect my-network container_name

# Run container with specific network
docker run -d --network my-network nginx
```

### Network Types

```bash
# Bridge network (default)
docker network create --driver bridge my-bridge

# Host network (container uses host network)
docker run -d --network host nginx

# None (no networking)
docker run -d --network none nginx

# Overlay network (Swarm mode)
docker network create --driver overlay my-overlay
```

## System & Cleanup

### System Information

```bash
# View Docker version
docker version

# View system information
docker info

# View disk usage
docker system df

# Detailed disk usage
docker system df -v
```

### Cleanup Commands

```bash
# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune

# Remove all unused volumes
docker volume prune

# Remove all unused networks
docker network prune

# Remove everything unused
docker system prune

# Remove everything including unused images
docker system prune -a

# Remove everything including volumes
docker system prune -a --volumes
```

## Common Patterns I Use

### Development Environment

```bash
# Run database for local development
docker run -d \
  --name dev-postgres \
  -e POSTGRES_PASSWORD=dev \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15

# Run Redis
docker run -d \
  --name dev-redis \
  -p 6379:6379 \
  redis:alpine

# Run Node app with hot reload
docker run -d \
  --name dev-app \
  -p 3000:3000 \
  -v $(pwd):/app \
  -v /app/node_modules \
  -e NODE_ENV=development \
  node:18 \
  npm run dev
```

### Quick Test Environment

```bash
# Spin up temporary test database
docker run --rm \
  -e POSTGRES_PASSWORD=test \
  -e POSTGRES_DB=testdb \
  -p 5433:5432 \
  postgres:15

# When done, container is automatically removed (--rm)
```

### Debugging

```bash
# Check why container exited
docker logs container_name

# View container processes
docker top container_name

# View container filesystem changes
docker diff container_name

# Export container filesystem
docker export container_name > container.tar

# Copy files from container
docker cp container_name:/path/to/file ./local/path

# Copy files to container
docker cp ./local/file container_name:/path/in/container
```

## Troubleshooting

### Container Won't Start

```bash
# Check logs for errors
docker logs container_name

# Try running interactively to see errors
docker run -it image_name /bin/sh

# Check if port is already in use
netstat -tuln | grep PORT

# Inspect container configuration
docker inspect container_name
```

### Can't Connect to Container

```bash
# Check container IP
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Check port mappings
docker port container_name

# Check if container is running
docker ps

# Test network connectivity
docker exec container_name ping other_container_name
```

### Out of Disk Space

```bash
# Check disk usage
docker system df

# Remove unused containers
docker container prune

# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune

# Nuclear option: remove everything
docker system prune -a --volumes
```

## Docker Best Practices

### 1. Use .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.vscode
.idea
*.log
```

### 2. Optimize Layer Caching

```dockerfile
# BAD: Changes to code invalidate dependency installation
COPY . .
RUN npm install

# GOOD: Dependencies cached separately
COPY package*.json ./
RUN npm install
COPY . .
```

### 3. Use Specific Image Tags

```dockerfile
# BAD: Tag can change
FROM node:latest

# GOOD: Specific, reproducible version
FROM node:18.17.1-alpine
```

### 4. Run as Non-Root User

```dockerfile
# Create and use non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

USER appuser
```

### 5. Multi-Stage Builds for Smaller Images

```dockerfile
# Build stage (large)
FROM node:18 AS builder
# ... build steps ...

# Production stage (small)
FROM node:18-alpine
COPY --from=builder /app/dist ./dist
```

## Quick Reference

### Most Common Commands

```bash
# Run container
docker run -d -p 8080:80 --name web nginx

# View logs
docker logs -f web

# Shell into container
docker exec -it web /bin/bash

# Stop and remove
docker stop web && docker rm web

# View running containers
docker ps

# View images
docker images

# Clean up everything
docker system prune -a
```

### Docker Compose Essentials

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Logs
docker compose logs -f

# Rebuild
docker compose up --build

# Shell into service
docker compose exec web /bin/bash
```

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

*Got a Docker command I should add? [Let me know](mailto:jordan@jordananderson.us).*
