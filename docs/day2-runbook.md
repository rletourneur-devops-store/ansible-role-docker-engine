# Docker Engine - Day 2+ Runbook

Operational guide for managing and troubleshooting Docker Engine.

---

## 1. Configuration Management

### 1.1 Modifying daemon.json

```bash
# Backup current configuration
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.backup

# Edit configuration
sudo nano /etc/docker/daemon.json

# Validate JSON syntax
cat /etc/docker/daemon.json | jq .

# Reload Docker
sudo systemctl reload docker
```

### 1.2 Adding Custom DNS Servers

```yaml
# In playbook
vars:
  docker_dns_servers:
    - "8.8.8.8"
    - "1.1.1.1"
```

### 1.3 Configuring Proxy

```yaml
# In playbook
vars:
  docker_http_proxy: "http://proxy.company.com:8080"
  docker_https_proxy: "http://proxy.company.com:8080"
  docker_no_proxy: "localhost,127.0.0.1,.local"
```

---

## 2. Storage Management

### 2.1 Checking Disk Usage

```bash
# Overall Docker disk usage
docker system df

# Detailed usage
docker system df -v

# Check data root location
docker info | grep "Docker Root Dir"

# Check filesystem usage
df -h /var/lib/docker  # Or custom data root
```

### 2.2 Cleanup Operations

```bash
# Remove unused containers, networks, images
docker system prune

# Remove everything (including volumes)
docker system prune -a --volumes

# Remove old images (older than 24h)
docker image prune -a --filter 'until=24h'

# Remove stopped containers
docker container prune

# Remove unused volumes
docker volume prune
```

### 2.3 Moving Data Root

To move Docker's data directory:

```yaml
# In playbook
vars:
  docker_data_root: "/mnt/new-location/docker"
```

Then re-run the playbook. Docker will migrate data automatically.

---

## 3. Troubleshooting

### 3.1 Docker Service Won't Start

```bash
# Check service status
sudo systemctl status docker

# Check logs
sudo journalctl -u docker -n 100 --no-pager

# Validate daemon.json
sudo cat /etc/docker/daemon.json | jq .

# Start in debug mode
sudo dockerd --debug

# Reset to defaults
sudo mv /etc/docker/daemon.json /etc/docker/daemon.json.backup
sudo systemctl restart docker
```

### 3.2 Containers Can't Reach Internet

```bash
# Check DNS configuration
docker run --rm alpine nslookup google.com

# Check if IP forwarding is enabled
sysctl net.ipv4.ip_forward

# Enable if disabled
sudo sysctl -w net.ipv4.ip_forward=1

# Check Docker network
docker network inspect bridge

# Restart Docker networking
sudo systemctl restart docker
```

### 3.3 Permission Denied Errors

```bash
# Check user groups
groups $USER

# Add user to docker group (if needed)
sudo usermod -aG docker $USER

# Log out and back in

# Verify
docker ps
```

### 3.4 Disk Space Issues

```bash
# Check disk usage
df -h

# Clean up Docker
docker system prune -a --volumes

# Find large containers
docker ps -as

# Find large images
docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -h

# Remove old images
docker image prune -a --filter 'until=72h'
```

---

## 4. Performance Optimization

### 4.1 Monitoring Containers

```bash
# Real-time stats
docker stats

# Stats for specific container
docker stats <container_name>

# One-time snapshot
docker stats --no-stream

# Formatted output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### 4.2 Setting Resource Limits

```bash
# Run with memory limit
docker run -m 512m myimage

# Run with CPU limit
docker run --cpus=".5" myimage

# Combined limits
docker run -m 512m --cpus=".5" myimage
```

### 4.3 Optimizing Images

```bash
# Use multi-stage builds
# Use Alpine base images
# Minimize layers
# Use .dockerignore

# Check image layers
docker history myimage

# Analyze image size
docker images myimage
```

---

## 5. Network Management

### 5.1 Network Troubleshooting

```bash
# List networks
docker network ls

# Inspect network
docker network inspect bridge

# Test connectivity between containers
docker run --rm --network=mynetwork alpine ping other-container

# Check port bindings
docker port <container>

# View iptables rules
sudo iptables -L -n | grep DOCKER
```

### 5.2 Creating Custom Networks

```bash
# Create bridge network
docker network create mynetwork

# Run containers on custom network
docker run -d --network mynetwork --name web nginx
docker run -d --network mynetwork --name db mysql

# Connect existing container to network
docker network connect mynetwork <container>
```

---

##6. Security Best Practices

### 6.1 Regular Updates

```bash
# Update Docker
sudo apt update
sudo apt upgrade docker-ce docker-ce-cli containerd.io

# Or re-run Ansible role
ansible-playbook -i inventory playbook.yml
```

### 6.2 Scanning Images

```bash
# Scan image for vulnerabilities
docker scan myimage:tag

# Use only trusted images
docker pull nginx  # Official images

# Verify image signatures
docker trust inspect nginx
```

### 6.3 Running Containers Securely

```bash
# Run as non-root user
docker run --user 1000:1000 myimage

# Drop capabilities
docker run --cap-drop ALL myimage

# Read-only filesystem
docker run --read-only myimage

# No new privileges
docker run --security-opt="no-new-privileges:true" myimage
```

---

## 7. Backup and Recovery

### 7.1 Backing Up Volumes

```bash
# List volumes
docker volume ls

# Backup volume
docker run --rm \
  -v myvolume:/source:ro \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/myvolume-$(date +%Y%m%d).tar.gz /source

# Restore volume
docker run --rm \
  -v myvolume:/target \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/myvolume-YYYYMMDD.tar.gz -C /target --strip-components=1
```

### 7.2 Backing Up Containers

```bash
# Commit container to image
docker commit mycontainer myimage:backup

# Export image
docker save myimage:backup | gzip > myimage-backup.tar.gz

# Import on another system
gunzip -c myimage-backup.tar.gz | docker load
```

### 7.3 Disaster Recovery

```bash
# If Docker won't start:
# 1. Backup daemon.json and data
sudo cp -r /var/lib/docker /var/lib/docker.backup

# 2. Reinstall Docker (via role or manually)

# 3. Restore configuration
sudo cp /var/lib/docker.backup/daemon.json /etc/docker/

# 4. Restart
sudo systemctl restart docker
```

---

## 8. Monitoring and Logging

### 8.1 Container Logs

```bash
# View logs
docker logs <container>

# Follow logs
docker logs -f <container>

# Last N lines
docker logs --tail 100 <container>

# With timestamps
docker logs -t <container>

# Since timestamp
docker logs --since 2023-01-01T00:00:00 <container>
```

### 8.2 Log Rotation

Docker automatically rotates logs. Check configuration:

```bash
# View log configuration
cat /etc/docker/daemon.json | jq '.["log-driver","log-opts"]'
```

### 8.3 Prometheus Metrics

If enabled:

```bash
# Check metrics
curl http://localhost:9323/metrics

# Key metrics to monitor
curl http://localhost:9323/metrics | grep -E "container_|engine_daemon"
```

---

## 9. Common Scenarios

### 9.1 High Memory Usage

```bash
# Find memory-heavy containers
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}" | sort -k 2 -h

# Restart container with limits
docker update --memory 512m <container>

# Or recreate with limit
docker run -m 512m myimage
```

### 9.2 Slow Container Performance

```bash
# Check I/O stats
docker stats

# Check host resources
top
iostat -x 1

# Check storage driver
docker info | grep "Storage Driver"

# Consider overlay2 if using devicemapper
```

### 9.3 Networking Issues

```bash
# Restart Docker networking
sudo systemctl restart docker

# Check firewall rules
sudo ufw status
sudo iptables -L -n

# Reset Docker networks
docker network prune

# Recreate default networks
sudo systemctl restart docker
```

---

## 10. Maintenance Checklist

**Weekly**:
- [ ] Check disk usage: `docker system df`
- [ ] Review container logs for errors
- [ ] Check resource usage: `docker stats`

**Monthly**:
- [ ] Clean up unused resources: `docker system prune -a`
- [ ] Update Docker: re-run Ansible role
- [ ] Review and backup important volumes
- [ ] Check for image updates

**As Needed**:
- [ ] Update daemon.json configuration
- [ ] Adjust automatic cleanup schedule
- [ ] Review and update security settings

---

## 11. Getting Help

```bash
# Docker documentation
docker --help
docker <command> --help

# Check Docker version
docker --version
docker version

# System information
docker info

# Check what changed
docker diff <container>

# Inspect any object
docker inspect <container|image|volume|network>
```

For more help, review the [Day 1 Guide](day1-guide.md) or the official Docker documentation.
