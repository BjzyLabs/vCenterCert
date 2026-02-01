---
name: consul
description: Interacts with HashiCorp Consul for service discovery, health checking, key-value storage, service mesh operations, DNS resolution, and distributed configuration management. Use when user mentions Consul, service discovery, health checks, KV store, service mesh, or distributed configuration.
---

# HashiCorp Consul Service Discovery

## Instructions

Use this skill to interact with HashiCorp Consul for service discovery, health checking, and distributed key-value storage. Consul provides service registration, health monitoring, and a distributed KV store used throughout the Home Lab infrastructure.

### Configuration

The preferred method of interaction is the `consul` CLI tool (installed on the host). Consul runs as a 3-node cluster in the Home Lab providing high availability for critical services.

#### Bjzy Labs defaults

- **Consul Cluster**: 3-node standalone cluster
- **Common use cases**:
  - **Service Discovery**: Find and connect to microservices
  - **Health Checking**: Monitor service health and availability
  - **KV Store**: Distributed configuration and coordination
  - **Leader Election**: Coordinate primary/standby roles (Patroni PostgreSQL)
- **Integration Points**:
  - **Patroni PostgreSQL**: Uses Consul for leader election and HA coordination
  - **Docker Swarm**: Service registration and discovery
  - **Vault**: Service-to-service authentication

### Environment and Guardrails (Bjzy Labs)

- **Cluster Access**:
  - Consul agents run on all infrastructure nodes
  - Local CLI access available from management workstations
  - HTTP API available at `http://localhost:8500` (when agent is running)
- **Security Rules**:
  - The agent must verify cluster health before making changes
  - Use read-only operations for service discovery queries
  - Be cautious with KV write operations (affect distributed state)
  - Never modify service health check configurations without verification
- **CLI Availability**:
  - The `consul` CLI is installed on management workstations
  - Environment variable `CONSUL_HTTP_ADDR` defaults to `127.0.0.1:8500`
  - Verify access with `consul members`

### Standard Operating Procedure (SOP)

When asked to "Check Consul," "Query services," or "Manage KV data":

1. **Verify Cluster:** Run `consul members` to check cluster connectivity
2. **Identify Operation:** Determine if you need service discovery, health checks, or KV operations
3. **Execute Query:** Use appropriate consul command with proper filtering
4. **Verify Results:** Cross-check service health and availability
5. **Document Changes:** Log any modifications to service configurations

## Examples

### 1. Check Cluster Health

Verify the Consul cluster is operational and all nodes are healthy.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# Check cluster members and their status
consul members

# Check operator-level health
consul operator raft list-peers

# Verify leader election
consul operator raft list-peers | grep Leader
```

### 2. List Registered Services

Discover all services currently registered in Consul.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# List all services
consul services

# List services with health status
consul catalog services

# Get detailed service information
consul catalog service <service_name>
```

### 3. Query Service Health

Check the health status of specific services or nodes.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# Check health of all services
consul health checks

# Check health for specific service
consul health checks -service <service_name>

# Check node-level health
consul health checks -node <node_name>

# Get service health with passing status only
consul health checks -service <service_name> -passing
```

### 4. Key-Value Store Operations

Manage distributed configuration and coordination data.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# List all keys in KV store
consul kv list

# List keys under specific path
consul kv list <path>/

# Get value from KV store
consul kv get <key>

# Put value to KV store (use with caution)
consul kv put <key> <value>

# Delete key from KV store
consul kv delete <key>

# Get key with metadata
consul kv get -detailed <key>
```

### 5. Service Registration Queries

Find services and their endpoints for connectivity.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# Find service instances with their addresses
consul catalog service <service_name>

# Get service details including tags
consul catalog service <service_name> -tags

# Query services by tag
consul catalog services -tag <tag_name>

# Get node information for service
consul catalog nodes -service <service_name>
```

### 6. Patroni PostgreSQL Integration

Monitor PostgreSQL cluster coordination via Consul.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# Check Patroni leader election status
consul kv get patroni/<cluster_name>/leader

# List Patroni cluster members
consul kv list patrono/<cluster_name>/members/

# Check PostgreSQL cluster health via Consul
consul health checks -service postgresql
```

### 7. Maintenance Mode Operations

Safely perform maintenance on services registered with Consul.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# Enable maintenance mode for a node
consul maintenance -enable -node <node_name> -reason "Maintenance"

# Disable maintenance mode
consul maintenance -disable -node <node_name>

# Check maintenance status
consul maintenance -list
```

### 8. Event Monitoring

Monitor Consul events for service changes.

- **Method:** `consul` CLI
- **Command Pattern:**

```bash
# Watch for service changes
consul event fire -name <event_name> -node <node_name>

# List recent events
consul event list

# Monitor events in real-time
consul event -name <event_name>
```

## Troubleshooting

### Cluster Connectivity Issues

```bash
# Check if Consul agent is running
consul version

# Verify cluster membership
consul members

# Check agent health
consul agent -info

# Test HTTP API connectivity
curl http://localhost:8500/v1/status/leader
```

### Service Not Found

```bash
# Verify service exists
consul catalog services | grep <service_name>

# Check service registration details
consul catalog service <service_name>

# Verify health check is passing
consul health checks -service <service_name>
```

### KV Store Access Issues

```bash
# Check KV permissions
consul kv list /

# Verify key exists
consul kv get <key>

# Check key metadata
consul kv get -detailed <key>
```

### Health Check Failures

```bash
# List all failing health checks
consul health checks -failing

# Get detailed health check information
consul health checks -service <service_name> -detailed

# Check specific health check
consul health checks -check-id <check_id>
```

### Leader Election Issues

```bash
# Check Raft consensus status
consul operator raft list-peers

# Verify cluster has a leader
consul operator raft configuration

# Check Raft metrics
consul operator raft get-consistency-index
```

## Common Service Patterns

### Docker Swarm Integration

Services in Docker Swarm automatically register with Consul for discovery:

```bash
# Find Swarm services
consul catalog services | grep docker

# Get service endpoints for load balancing
consul catalog service <swarm_service_name>
```

### Database Connection Strings

Use Consul for dynamic database endpoint resolution:

```bash
# Get PostgreSQL primary endpoint
consul catalog service postgresql | grep passing
```

### Configuration Management

Store application configuration in Consul KV store:

```bash
# Get application configuration
consul kv get config/<app_name>/<environment>/<setting>

# List all configuration for application
consul kv list config/<app_name>/<environment>/
```
