---
name: docker-swarm
description: Manages Docker Swarm orchestration for service deployment, stack operations, node management, scaling, secrets handling, network configuration, and distributed container orchestration. Use when user mentions Docker Swarm, stacks, services, swarm nodes, container orchestration, or compose files.
---

# Docker, Docker Compose & Docker Swarm Integration

## Instructions

Use this skill to interact with Docker standalone, Docker Compose applications, and Docker Swarm clusters. Docker Swarm is the container orchestration platform for the Home Lab, running production and development workloads. Docker Compose is used for single-host multi-container applications and local development.
**Always prefer using AWX Job Templates for deployment operations over manual docker/compose commands.**

### Configuration

The preferred method of interaction is SSH to Swarm manager nodes or using AWX job templates for orchestration tasks.

**Smart Access Pattern:**
Before running commands, determine if the operation should be executed via AWX (deployments, configuration changes) or direct SSH (read-only verification, troubleshooting).

#### Bjzy Labs defaults

- **Docker Swarm Clusters**:
  - **Production**: 3-node manager cluster (Huey, Dewey, Louie)
  - **Development**: 3-node manager cluster (devHuey, devDewey, devLouie)
- **AWX Integration**:
  - Job Template ID 68: "Docker Swarm - Generic" (playbook: `docker-swarm-manage.yml`)
  - Workflow ID 65: "Docker Swarm - Full Stack Deployment"
- **Vault lookup (for credentials if needed)**:
  - Path: `kvProd_v2/Docker/` (various secrets)

### Environment and Guardrails (Bjzy Labs)

- **Managed Environments**:
  - **Production Cluster**:
    - **Inventory ID**: 15 (Docker Swarm - Production)
    - **Nodes**: Huey (192.168.60.81), Dewey (192.168.60.82), Louie (192.168.60.83)
    - **Hostnames**: huey.bjzy.me, dewey.bjzy.me, louie.bjzy.me
    - **Network**: 192.168.60.0/24 (smDSwitch-ProdLAN)
    - **Swarm Node IDs**:
      - Huey: `8mj25u1rmr9tzdbp5oj179zlx` (Manager - Reachable)
      - Dewey: `ma7q2zjjolchc5bs8olga68kv` (Manager - Reachable)
      - Louie: `ynor96m8zqq427wj5u9mx7dok` (Manager - Leader)
  - **Development Cluster**:
    - **Inventory ID**: 16 (Docker Swarm - Development)
    - **Nodes**: devHuey (192.168.50.81), devDewey (192.168.50.82), devLouie (192.168.50.83)
    - **Hostnames**: devhuey.bjzy.me, devdewey.bjzy.me, devlouie.bjzy.me
    - **Network**: 192.168.50.0/24 (smDSwitch-DevLAN)
    - **Swarm Node IDs**:
      - devHuey: `stdt4v7vltry18zkmxi5fo62i` (Manager - Reachable)
      - devDewey: `i4wayj3rtpsfy5pl102vzop84` (Manager - Leader)
      - devLouie: `raw7wsjdvhddbvvliu52bp3ps` (Manager - Reachable)

- **Execution Rules**:
  - The agent must **NEVER** run `docker stack deploy` manually; use AWX job templates.
  - The agent must **NEVER** modify Docker daemon configuration directly; changes must be made in the `homelab-playbooks` repo and deployed via AWX.
  - **SSH is allowed for read-only verification and troubleshooting**.
  - **Hostname safety**: `devHuey` != `Huey` (dev vs prod are different clusters).

- **Overlay Networks** (both clusters):
  - `ingress` - Default overlay network for published ports
  - `traefik-public` - Public-facing overlay for Traefik services
  - `management` - Encrypted overlay for admin services
  - `monitoring` - Encrypted overlay for monitoring stack (Grafana Alloy, etc.)
  - `backend` - Encrypted overlay for application backends

- **Storage**:
  - Local storage: `/var/lib/docker` (20GB LVM partition per node)
  - Shared storage: MicroCeph cluster (CephFS mounts for persistent data)
  - Docker volumes: Mix of local and CephFS-backed volumes

### Standard Operating Procedure (SOP)

When asked to "Deploy a service," "Check Swarm status," or "Troubleshoot containers":

1. **Determine Environment**: Identify if this is Production (Inventory 15) or Development (Inventory 16).
2. **Choose Method**:
   - **Deployments/Changes**: Use AWX Job Template 68 or Workflow 65.
   - **Read-Only Checks**: SSH directly to any manager node.
3. **Verify Access**: For SSH operations, use `ssh ansible@<hostname>` (e.g., `ssh ansible@huey.bjzy.me`).
4. **Execute & Monitor**: Run the operation and verify results.
5. **Document**: If issues are found, consider enriching Keep alerts or updating documentation.

## Examples

### 1. Check Swarm Cluster Status (Read-Only)

Use this to verify cluster health, node status, and service distribution.

- **Method:** SSH to any manager node
- **Command Pattern:**

```bash
# Check node status (run on any manager)
ssh ansible@huey.bjzy.me "docker node ls"

# Check service status
ssh ansible@huey.bjzy.me "docker service ls"

# Check specific service details
ssh ansible@huey.bjzy.me "docker service ps <service-name>"

# Check cluster info
ssh ansible@huey.bjzy.me "docker info --format '{{.Swarm.NodeID}}\t{{.Swarm.ControlAvailable}}\t{{.Swarm.Managers}}'"
```

### 2. Inspect Service Logs (Troubleshooting)

View logs from a specific service across all replicas.

- **Method:** SSH to any manager node
- **Command Pattern:**

```bash
# View service logs (last 100 lines)
ssh ansible@huey.bjzy.me "docker service logs --tail 100 <service-name>"

# Follow service logs in real-time
ssh ansible@huey.bjzy.me "docker service logs -f <service-name>"

# View logs from specific task/container
ssh ansible@huey.bjzy.me "docker logs <container-id>"
```

### 3. Check Container Resource Usage (Read-Only)

Monitor resource consumption across the cluster.

- **Method:** SSH to specific node
- **Command Pattern:**

```bash
# Check resource usage on a specific node
ssh ansible@dewey.bjzy.me "docker stats --no-stream"

# Check disk usage
ssh ansible@louie.bjzy.me "docker system df"

# Check image storage
ssh ansible@huey.bjzy.me "docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'"
```

### 4. Deploy Service via AWX (Preferred Method)

Use AWX to deploy or update services in the cluster.

- **Method:** AWX CLI
- **Command Pattern:**

```bash
# Launch Docker Swarm management job
awx job_template launch 68 \
  --extra_vars '{
    "operation": "deploy-service",
    "service_name": "my-app",
    "environment": "production"
  }' \
  --inventory 15 \
  --monitor

# Launch full stack deployment workflow
awx workflow_job_template launch 65 \
  --inventory 16 \
  --monitor
```

### 5. Check Network Configuration (Read-Only)

Verify overlay networks and connectivity.

- **Method:** SSH to any manager node
- **Command Pattern:**

```bash
# List all networks
ssh ansible@huey.bjzy.me "docker network ls"

# Inspect specific network
ssh ansible@huey.bjzy.me "docker network inspect traefik-public"

# Check which services are on a network
ssh ansible@huey.bjzy.me "docker network inspect backend --format '{{range .Containers}}{{.Name}} {{end}}'"
```

### 6. Verify Service Health (Read-Only)

Check health status of running services.

- **Method:** SSH to any manager node
- **Command Pattern:**

```bash
# Check service replicas and health
ssh ansible@dewey.bjzy.me "docker service ls --format 'table {{.Name}}\t{{.Replicas}}\t{{.Image}}'"

# Check for failed tasks
ssh ansible@dewey.bjzy.me "docker service ps <service-name> --filter 'desired-state=shutdown' --format 'table {{.Name}}\t{{.Node}}\t{{.Error}}'"

# Inspect service configuration
ssh ansible@dewey.bjzy.me "docker service inspect <service-name> --pretty"
```

### 7. Check Docker Daemon Status (Read-Only)

Verify Docker daemon health on individual nodes.

- **Method:** SSH to specific node
- **Command Pattern:**

```bash
# Check Docker service status
ssh ansible@louie.bjzy.me "systemctl status docker"

# Check Docker version
ssh ansible@huey.bjzy.me "docker version"

# Check Docker daemon configuration
ssh ansible@dewey.bjzy.me "docker info"
```

### 8. List Running Containers on Node (Read-Only)

View containers running on a specific node.

- **Method:** SSH to specific node
- **Command Pattern:**

```bash
# List all containers on a node
ssh ansible@devhuey.bjzy.me "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'"

# List containers with resource usage
ssh ansible@devlouie.bjzy.me "docker ps --format 'table {{.Names}}\t{{.Status}}' && docker stats --no-stream"
```

## Docker Compose Examples

Docker Compose is used for single-host multi-container applications. Common use cases include development environments, standalone services, and applications not requiring Swarm orchestration.

**IMPORTANT**: Use `docker compose` (with a space) - this is the modern Docker CLI plugin syntax. The legacy `docker-compose` (with a hyphen) command is deprecated and should NOT be used.

### 9. Check Docker Compose Application Status (Read-Only)

View status of Docker Compose applications.

- **Method:** SSH to host running compose application
- **Command Pattern:**

```bash
# List running compose services (from compose directory)
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps"

# View compose application logs
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose logs --tail 100"

# Check specific service logs
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose logs -f service-name"

# View compose configuration
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config"
```

### 10. Inspect Docker Compose Resources (Read-Only)

Check networks, volumes, and containers created by Compose.

- **Method:** SSH to host
- **Command Pattern:**

```bash
# List compose-managed containers
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps -a"

# View compose networks
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config --networks"

# View compose volumes
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config --volumes"

# Check resource usage of compose services
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps -q | xargs docker stats --no-stream"
```

### 11. Validate Docker Compose Configuration (Read-Only)

Verify compose file syntax and configuration.

- **Method:** SSH to host
- **Command Pattern:**

```bash
# Validate compose file syntax
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config --quiet"

# View resolved compose configuration
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config"

# Check for compose file issues
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config 2>&1 | grep -i error"
```

### 12. Docker Compose via AWX (Preferred for Deployments)

Use AWX to deploy or manage Docker Compose applications.

- **Method:** AWX CLI
- **Command Pattern:**

```bash
# Launch compose deployment via AWX
awx job_template launch <template-id> \
  --extra_vars '{
    "operation": "deploy",
    "compose_app": "myapp",
    "compose_path": "/opt/myapp"
  }' \
  --inventory 15 \
  --monitor

# Note: Specific job template IDs depend on your AWX configuration
```

### 13. Harbor Registry Operations (Read-Only)

Harbor is the container registry for the Home Lab, hosting AWX EE images, custom applications, and third-party images.

- **Method:** Harbor API (primary) + Docker CLI (registry operations)
- **Registry URL:** `harbor.bjzy.me`
- **Authentication:** Robot account tokens (from Vault)
- **Note:** Harbor CLI (`harbor-cli`) is not installed by default; use API + Docker CLI

**Harbor API Examples:**
```bash
# Check Harbor API health (no auth)
curl -k https://harbor.bjzy.me/api/v2/health

# List projects (requires token)
HARBOR_TOKEN=$(vault kv get -field=robot_token kvProd_v2/Harbor/Application-Prod)
curl -k -H "Authorization: Bearer $HARBOR_TOKEN" \
  https://harbor.bjzy.me/api/v2.0/projects

# List images in a project
curl -k -H "Authorization: Bearer $HARBOR_TOKEN" \
  https://harbor.bjzy.me/api/v2.0/projects/awx/repositories

# Check image manifest
curl -k -H "Authorization: Bearer $HARBOR_TOKEN" \
  https://harbor.bjzy.me/v2/awx/awx-ee/tags/latest
```

**Docker Registry Operations:**
```bash
# Login to Harbor (robot account)
docker login harbor.bjzy.me -u robot-<project> -p <token>

# Pull AWX EE image
docker pull harbor.bjzy.me/awx/awx-ee:latest

# Push custom image
docker tag myapp:latest harbor.bjzy.me/myapp/myapp:latest
docker push harbor.bjzy.me/myapp/myapp:latest

# List available images (requires Harbor API)
curl -k -H "Authorization: Bearer $HARBOR_TOKEN" \
  https://harbor.bjzy.me/api/v2.0/repositories
```

### 14. Docker Logging Configuration (Read-Only)

Verify Docker daemon logging configuration and Loki integration.

- **Method:** SSH to any node
- **Purpose:** Verify logs are flowing to Grafana Loki via Alloy

**Command Pattern:**
```bash
# Check Docker daemon logging driver
ssh ansible@huey.bjzy.me "docker info --format '{{.LoggingDriver}}'"

# Verify daemon.json logging configuration
ssh ansible@huey.bjzy.me "sudo cat /etc/docker/daemon.json | jq -r '.\"log-driver\" // \"json-file\"'"

# Check container log paths (for troubleshooting)
ssh ansible@huey.bjzy.me "docker inspect <container-id> --format '{{.LogPath}}'"

# Verify Alloy is collecting Docker logs
ssh ansible@huey.bjzy.me "sudo systemctl status alloy"

# Check Alloy Docker log collection config
ssh ansible@huey.bjzy.me "sudo cat /etc/alloy/config.alloy | grep -A 5 'docker.*logs'"
```

### 15. Emergency Stack Deploy Reference

**NOTE:** Stack deploy operations should normally use AWX Job Templates. This reference is for emergency scenarios when AWX is unavailable.

- **Documentation:** `docs/DOCKER_SWARM_DEPLOYMENT.md`
- **Playbook:** `playbooks/docker-swarm-manage.yml`
- **AWX Template:** Job Template 68 ("Docker Swarm - Generic")
- **Emergency Command Pattern:** `docker stack deploy -c docker-compose.yml stack-name`

**For AWX failures:**
1. Check AWX service status via `awx-control` skill
2. Consider manual stack deploy only if AWX is completely unavailable
3. Always document any manual deployments for later AWX reconciliation

## Troubleshooting

### Service Won't Start

```bash
# Check service logs for errors
ssh ansible@huey.bjzy.me "docker service logs <service-name>"

# Check task failures
ssh ansible@huey.bjzy.me "docker service ps <service-name> --no-trunc"

# Inspect service constraints and placement
ssh ansible@huey.bjzy.me "docker service inspect <service-name> --format '{{.Spec.TaskTemplate.Placement}}'"
```

### Node Shows as Down

```bash
# Check Docker service on the node
ssh ansible@dewey.bjzy.me "systemctl status docker"

# Check node status from manager
ssh ansible@huey.bjzy.me "docker node inspect dewey.bjzy.me --pretty"

# Check system logs
ssh ansible@dewey.bjzy.me "sudo journalctl -u docker.service -n 50"
```

### Network Connectivity Issues

```bash
# Check overlay network status
ssh ansible@huey.bjzy.me "docker network ls --filter driver=overlay"

# Inspect network for connected services
ssh ansible@huey.bjzy.me "docker network inspect backend"

# Test connectivity between services (if possible)
ssh ansible@huey.bjzy.me "docker exec <container-id> ping <other-service-name>"
```

### Storage Issues

```bash
# Check disk usage across nodes
ssh ansible@huey.bjzy.me "df -h /var/lib/docker"
ssh ansible@dewey.bjzy.me "df -h /var/lib/docker"
ssh ansible@louie.bjzy.me "df -h /var/lib/docker"

# Check Docker storage usage
ssh ansible@huey.bjzy.me "docker system df -v"

# Check for dangling images/volumes
ssh ansible@huey.bjzy.me "docker images -f dangling=true"
ssh ansible@huey.bjzy.me "docker volume ls -f dangling=true"
```

### Certificate/TLS Issues

```bash
# Check Docker daemon TLS configuration
ssh ansible@huey.bjzy.me "docker info --format '{{.SecurityOptions}}'"

# Verify certificates exist
ssh ansible@huey.bjzy.me "ls -la /etc/docker/certs.d/"

# Check service-specific TLS mounts
ssh ansible@huey.bjzy.me "docker service inspect <service-name> --format '{{.Spec.TaskTemplate.ContainerSpec.Mounts}}'"
```

### Docker Compose Application Issues

```bash
# Check compose service status
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps"

# View compose logs for errors
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose logs --tail 200"

# Check specific service that won't start
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose logs service-name"

# Validate compose file
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config"

# Check compose network connectivity
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose exec service-name ping other-service"

# View compose service resource usage
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps -q | xargs docker stats --no-stream"

# Check for port conflicts
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config | grep -A 2 ports"
```

## Quick Reference Commands

### Cluster Management (Read-Only)
```bash
# View all nodes
ssh ansible@huey.bjzy.me "docker node ls"

# View all services
ssh ansible@huey.bjzy.me "docker service ls"

# View all stacks
ssh ansible@huey.bjzy.me "docker stack ls"

# View cluster-wide events
ssh ansible@huey.bjzy.me "docker events --filter type=service --filter type=node"
```

### Service Inspection (Read-Only)
```bash
# Service details
ssh ansible@huey.bjzy.me "docker service inspect <service-name>"

# Service logs
ssh ansible@huey.bjzy.me "docker service logs <service-name>"

# Service tasks/replicas
ssh ansible@huey.bjzy.me "docker service ps <service-name>"
```

### Node-Specific Checks (Read-Only)
```bash
# Containers on node
ssh ansible@dewey.bjzy.me "docker ps"

# Resource usage on node
ssh ansible@dewey.bjzy.me "docker stats --no-stream"

# Images on node
ssh ansible@dewey.bjzy.me "docker images"
```

### Docker Compose Operations (Read-Only)
```bash
# View compose application status
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps"

# View compose logs
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose logs --tail 50"

# Validate compose configuration
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose config"

# Check compose resource usage
ssh ansible@huey.bjzy.me "cd /opt/myapp && docker compose ps -q | xargs docker stats --no-stream"
```

### Harbor Registry Operations (Read-Only)
```bash
# Check Harbor health
curl -k https://harbor.bjzy.me/api/v2/health

# List Harbor projects (with token)
HARBOR_TOKEN=$(vault kv get -field=robot_token kvProd_v2/Harbor/Application-Prod)
curl -k -H "Authorization: Bearer $HARBOR_TOKEN" https://harbor.bjzy.me/api/v2.0/projects

# Pull AWX EE from Harbor
docker pull harbor.bjzy.me/awx/awx-ee:latest

# Check Docker logging driver
ssh ansible@huey.bjzy.me "docker info --format '{{.LoggingDriver}}'"
```

## Important Notes

- **Never run destructive commands** (`docker service rm`, `docker stack rm`, `docker swarm leave`) without explicit user confirmation.
- **Always verify environment** (Production vs Development) before executing commands.
- **Use AWX for deployments** - Direct `docker stack deploy` should only be used for emergency fixes.
- **Check Keep alerts first** - Many Swarm issues may already have alerts in Keep AIOps.
- **Document findings** - If you discover issues, enrich Keep alerts or update Notion documentation.

## Related Documentation

- **Notion**: [Docker Swarm Management — Infrastructure Automation Guide](https://www.notion.so/2763569aa25581a88752cd80ff3d68f9)
- **Notion**: [Docker Swarm Deployment Variables Backup](https://www.notion.so/2b83569aa25581a2bbfefa2fd6ef63af)
- **GitHub Repo**: `homelab-playbooks` (BjzyLabs/ansible-homelab)
- **Local Docs**: `docs/DOCKER_SWARM_OPERATIONS.md`, `docs/DOCKER_SWARM_DEPLOYMENT.md`
