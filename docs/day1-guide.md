# Docker Engine - Day 1 Guide

Quick start guide after deploying the `devops_store.docker_engine` role.

---

## 1. Post-Deployment Verification

### 1.1 Verify Docker Installation

```bash
# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Verify Docker is running
sudo docker ps

# Check Docker info
sudo docker info
```

### 1.2 Verify Docker Configuration

```bash
# Check daemon configuration
cat /etc/docker/daemon.json | jq .

# Check Docker service configuration
systemctl cat docker | grep -A 10 Environment

# Verify data root
docker info | grep "Docker Root Dir"
```

### 1.3 Verify User Groups

```bash
# Check if user is in docker group
groups $USER

# Test Docker without sudo
docker ps

# If needed, log out and back in for group changes to take effect
```

---

## 2. Basic Docker Commands

### 2.1 Container Management

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# Run a test container
docker run hello-world

# Run interactive container
docker run -it ubuntu bash

# Stop a container
docker stop <container_id>

# Remove a container
docker rm <container_id>
```

### 2.2 Image Management

```bash
# List images
docker images

# Pull an image
docker pull nginx

# Remove an image
docker rmi <image_id>

# Build an image
docker build -t myapp:latest .
```

### 2.3 Network and Volume

```bash
# List networks
docker network ls

# List volumes
docker volume ls

# Inspect a network
docker network inspect bridge

# Create a volume
docker volume create mydata
```

---

## 3. Health Checks

### 3.1 Quick Health Check

```bash
# Run all checks
docker --version && \
sudo systemctl is-active docker && \
docker ps && \
docker info | grep -E "Server Version|Storage Driver|Docker Root Dir"
```

### 3.2 Verify Proxy Configuration (if configured)

```bash
# Check proxy settings
systemctl show docker --property=Environment

# Test pulling image through proxy
docker pull hello-world
```

### 3.3 Check Resource Usage

```bash
# Disk usage
docker system df

# Detailed disk usage
docker system df -v

# Container resource usage
docker stats --no-stream
```

---

## 4. Common First-Day Tasks

### 4.1 Running Your First Application

```bash
# Run Nginx
docker run -d -p 80:80 --name mynginx nginx

# Verify it's running
curl http://localhost

# Check logs
docker logs mynginx

# Stop and remove
docker stop mynginx
docker rm mynginx
```

### 4.2 Using Docker Compose

```bash
# Create docker-compose.yml
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  web:
    image: nginx
    ports:
      - "80:80"
EOF

# Start services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs

# Stop services
docker compose down
```

### 4.3 Managing Persistent Data

```bash
# Create a volume
docker volume create mydata

# Run container with volume
docker run -d -v mydata:/data nginx

# Inspect volume
docker volume inspect mydata

# Backup volume
docker run --rm -v mydata:/source -v $(pwd):/backup ubuntu tar czf /backup/mydata.tar.gz /source
```

---

## 5. Verifying Special Configurations

### 5.1 Custom Data Root

If you configured a custom data root:

```bash
# Verify data root location
docker info | grep "Docker Root Dir"

# Check disk usage at custom location
df -h /mnt/data  # Or your custom path

# List contents
sudo ls -la /mnt/data/docker
```

### 5.2 Custom DNS

If you configured custom DNS:

```bash
# Check daemon.json
cat /etc/docker/daemon.json | jq '.dns'

# Test DNS resolution in container
docker run --rm alpine nslookup google.com
```

### 5.3 Insecure Registries

If you configured insecure registries:

```bash
# Check configuration
cat /etc/docker/daemon.json | jq '.["insecure-registries"]'

# Test pull from insecure registry
docker pull your-registry:5000/image:tag
```

### 5.4 Prometheus Metrics (if enabled)

If metrics are enabled:

```bash
# Check metrics port
curl http://localhost:9323/metrics

# Verify metrics in daemon.json
cat /etc/docker/daemon.json | jq '.["metrics-addr"]'
```

---

## 6. Automatic Cleanup Verification

### 6.1 Verify Cron Job

```bash
# Check if prune job is configured
sudo crontab -l | grep docker

# Manual cleanup test
docker system prune -a --force --filter 'until=24h' --dry-run
```

---

## 7. Python Docker SDK Verification

```bash
# Check if Python Docker SDK is installed
python3 -c "import docker; print(docker.__version__)"

# Test basic functionality
python3 <<EOF
import docker
client = docker.from_env()
print(client.version())
EOF
```

---

## 8. Tips and Best Practices

1. **Always use docker compose** for multi-container apps
2. **Name your containers** with `--name` for easier management
3. **Use volumes** for persistent data, not bind mounts in production
4. **Tag your images** properly (avoid `latest` in production)
5. **Clean up regularly** with `docker system prune`
6. **Monitor resources** with `docker stats`
7. **Check logs** with `docker logs -f <container>`
8. **Use .dockerignore** to reduce image sizes
9. **Run as non-root** inside containers when possible
10. **Keep Docker updated** by re-running the role

---

## 9. Next Steps

- Review [`day2-runbook.md`](day2-runbook.md) for troubleshooting and advanced usage
- Set up Docker Compose for your applications
- Configure container monitoring
- Plan backup strategy for volumes
- Review security best practices
