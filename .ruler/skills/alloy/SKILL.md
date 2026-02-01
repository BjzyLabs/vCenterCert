---
name: alloy
description: Manages Grafana Alloy telemetry pipeline for metrics collection, log forwarding, trace aggregation, data transformation, and observability routing to Mimir, Loki, and Tempo. Use when user mentions Alloy, telemetry, metrics collection, observability pipeline, or data forwarding.
---

# Grafana Alloy Integration

## Instructions

Use this skill to interact with Grafana Alloy for telemetry collection, metrics shipping, and observability pipeline management. Alloy is the successor to Grafana Agent and provides unified collection of metrics, logs, traces, and profiles.
**Always verify the target environment (local vs remote) before executing Alloy commands.**

### Configuration

Grafana Alloy can be deployed in two primary modes:
- **Local On-Device**: Alloy running directly on infrastructure nodes for immediate telemetry collection
- **Remote On-Laptop**: Alloy running on management laptop for remote monitoring and debugging

**Smart Authentication:**
Alloy typically doesn't require authentication for local CLI operations, but remote endpoints (Prometheus, Loki) do.

```bash
# Verify Alloy service is running
if ! systemctl is-active --quiet alloy; then
  echo "Alloy service not running"
  sudo systemctl status alloy
  exit 1
fi

# Check configuration validity before applying
sudo alloy -config /etc/alloy/alloy.alloy -dry-run || { echo "Config validation failed"; exit 1; }

# For remote endpoints, retrieve credentials from Vault
export LOKI_TOKEN=$(vault kv get -field=push_token kvProd_v2/Loki/Application-Prod)
export PROMETHEUS_TOKEN=$(vault kv get -field=remote_write_token kvProd_v2/Prometheus/Application-Prod)
```

### CLI vs API Decision Tree

**Alloy is CLI and config-file based. No API for configuration (only metrics/health endpoints).**

| Operation | Use CLI | Use HTTP Endpoint |
|-----------|---------|-------------------|
| Configure pipelines | ✅ Edit config file + reload | ❌ |
| Validate configuration | ✅ `alloy -dry-run` | ❌ |
| Check service status | ✅ `systemctl status alloy` | ✅ `/-/ready` (health) |
| View metrics | ❌ | ✅ `/metrics` (Prometheus format) |
| Reload configuration | ✅ `systemctl reload alloy` | ✅ `POST /-/reload` |
| Debug pipelines | ✅ CLI logs and dry-run | ❌ |

**Rule:** Alloy configuration is file-based. HTTP endpoints are only for health checks and metrics export.

### Certificate Handling (Homelab)

Alloy sends telemetry to endpoints that may use self-signed certificates.

```bash
# In Alloy config, configure CA certificate for homelab endpoints (REQUIRED):
prometheus.remote_write "loki" {
  endpoint {
    url = "https://loki.bjzy.me/loki/api/v1/push"
    tls_config {
      ca_file = "/etc/ssl/certs/homelab-ca.pem"
    }
  }
}

# For Mimir endpoint with CA certificate:
prometheus.remote_write "mimir" {
  endpoint {
    url = "https://mimir.bjzy.me/api/v1/push"
    tls_config {
      ca_file = "/etc/ssl/certs/homelab-ca.pem"
    }
  }
}

# ⚠️ SECURITY: Never use insecure_skip_verify = true in production
# This disables TLS certificate validation and enables man-in-the-middle attacks.
# Always configure ca_file with your homelab CA certificate instead.
```

**Smart Access Pattern:**
- **Local Operations**: Use direct Alloy CLI on the target node
- **Remote Operations**: Use SSH to access Alloy on management laptop
- **Configuration Management**: Verify Alloy config files and service status
- **Debugging**: Use Alloy CLI for pipeline validation and troubleshooting

#### Bjzy Labs defaults

- **Alloy Deployment Architecture**:
  - **Production**: 3-node Alloy cluster for HA telemetry collection
  - **Development**: Single-node Alloy instance for testing
  - **Management Laptop**: Alloy instance for remote monitoring and debugging
- **Common use cases**:
  - **Metrics Collection**: Prometheus metrics from various sources
  - **Log Shipping**: Loki integration for centralized logging
  - **Pipeline Management**: Configure and validate Alloy pipelines
- **Integration Points**:
  - **Prometheus**: Metrics scraping and remote_write
  - **Loki**: Log aggregation and shipping
  - **Mimir**: Long-term metrics storage and querying
  - **Vault**: Configuration secrets and API keys

### Environment and Guardrails (Bjzy Labs)

- **Managed Environments**:
  - **Production Alloy Cluster**:
    - **Nodes**: alloy-01, alloy-02, alloy-03 (HA configuration)
    - **Network**: 192.168.72.0/24 (Telemetry network)
    - **Config Path**: `/etc/alloy/alloy.alloy`
    - **Data Path**: `/var/lib/alloy/`
  - **Development Alloy**:
    - **Node**: dev-alloy-01
    - **Network**: 192.168.73.0/24 (Dev telemetry network)
    - **Config Path**: `/etc/alloy/alloy.alloy`
  - **Management Laptop (Remote)**:
    - **Host**: MacBook Pro (laptop)
    - **Config Path**: `~/alloy/config/alloy.alloy`
    - **Data Path**: `~/alloy/data/`

- **Execution Rules**:
  - The agent must **ALWAYS** verify target environment before operations
  - The agent must **NEVER** restart Alloy services without explicit confirmation
  - **SSH is allowed for remote laptop access** and configuration management
  - **Configuration safety**: Always backup config files before modifications

- **Alloy Components**:
  - **Metrics Collection**: Prometheus scraping and remote_write
  - **Log Collection**: File and systemd log collection
  - **Pipeline Processing**: Built-in data transformation and routing

### Standard Operating Procedure (SOP)

When asked to "Check Alloy status," "Configure Alloy pipeline," or "Troubleshoot telemetry":

1. **Verify Environment**: Determine if targeting local on-device or remote laptop
2. **Check Service Status**: Verify Alloy is running and healthy
3. **Choose Method**:
   - **Local Operations**: Use Alloy CLI directly on the node
   - **Remote Operations**: SSH to management laptop
4. **Execute & Monitor**: Run the operation and verify results
5. **Validate**: Check pipeline health and telemetry flow

## Examples

### 1. Local On-Device: Check Alloy Service Status

Use this to verify Alloy is running correctly on infrastructure nodes.

- **Method:** Direct Alloy CLI on node
- **Command Pattern:**

```bash
# Check Alloy service status
sudo systemctl status alloy

# Check Alloy process
ps aux | grep alloy

# Check Alloy configuration
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Check Alloy logs
sudo journalctl -u alloy -n 50 -f
```

### 2. Remote On-Laptop: Check Management Alloy

Use this to verify Alloy on the management laptop for remote monitoring.

- **Method:** SSH to management laptop
- **Command Pattern:**

```bash
# SSH to management laptop
ssh bjzy@laptop

# Check Alloy status on laptop
brew services list | grep alloy
# or
ps aux | grep alloy

# Check Alloy configuration
alloy -config ~/alloy/config/alloy.alloy -dry-run

# Check Alloy logs
tail -f ~/alloy/logs/alloy.log
```

### 3. Local On-Device: Validate Alloy Configuration

Verify and test Alloy pipeline configuration on infrastructure nodes.

- **Method:** Direct Alloy CLI on node
- **Command Pattern:**

```bash
# Validate configuration syntax
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Check configuration file
sudo cat /etc/alloy/alloy.alloy

# Test specific pipeline components
sudo alloy -config /etc/alloy/alloy.alloy -dry-run -debug

# Check configuration reload
sudo alloy -config /etc/alloy/alloy.alloy -dry-run -reload-interval 30s
```

### 4. Remote On-Laptop: Debug Telemetry Pipeline

Use the laptop Alloy instance for debugging and troubleshooting telemetry issues.

- **Method:** SSH to management laptop
- **Command Pattern:**

```bash
# SSH to management laptop
ssh bjzy@laptop

# Check telemetry pipeline status
alloy -config ~/alloy/config/alloy.alloy -debug

# View active connections
alloy -config ~/alloy/config/alloy.alloy -debug | grep "connection"

# Check metrics being scraped
curl http://localhost:12345/metrics | grep prometheus

# Check log collection
tail -f ~/alloy/logs/collector.log
```

### 5. Local On-Device: Monitor Resource Usage

Check Alloy resource consumption on infrastructure nodes.

- **Method:** Direct commands on node
- **Command Pattern:**

```bash
# Check Alloy process resources
ps aux | grep alloy

# Check memory usage
sudo pmap -d $(pgrep alloy)

# Check CPU usage
top -p $(pgrep alloy)

# Check disk usage for data directory
sudo du -sh /var/lib/alloy/

# Check network connections
sudo netstat -tulpn | grep alloy
```

### 6. Remote On-Laptop: Test Remote Write

Verify telemetry data is being sent to remote backends from the laptop.

- **Method:** SSH to management laptop
- **Command Pattern:**

```bash
# SSH to management laptop
ssh bjzy@laptop

# Check remote write status
curl -s http://localhost:12345/metrics | grep remote_write

# Test connectivity to local Mimir
curl -I http://mimir.bjzy.me:9009/api/prom/push

# Check Loki write status
curl -s http://localhost:12345/metrics | grep loki
```

### 7. Local On-Device: Manage Alloy Service

Control Alloy service on infrastructure nodes.

- **Method:** System service management
- **Command Pattern:**

```bash
# Restart Alloy service (with caution)
sudo systemctl restart alloy

# Reload configuration
sudo systemctl reload alloy

# Check service logs
sudo journalctl -u alloy -n 100

# Enable/disable service
sudo systemctl enable alloy
sudo systemctl disable alloy
```

### 8. Remote On-Laptop: Update Configuration

Modify Alloy configuration on the management laptop.

- **Method:** SSH to management laptop
- **Command Pattern:**

```bash
# SSH to management laptop
ssh bjzy@laptop

# Edit configuration
nano ~/alloy/config/alloy.alloy

# Validate new configuration
alloy -config ~/alloy/config/alloy.alloy -dry-run

# Restart laptop Alloy
brew services restart alloy

# Check status after restart
alloy -config ~/alloy/config/alloy.alloy -debug
```

### 9. Local On-Device: Check Telemetry Flow

Verify telemetry collection and forwarding on infrastructure nodes.

- **Method:** Direct commands on node
- **Command Pattern:**

```bash
# Check metrics endpoints
curl -s http://localhost:12345/metrics | head -20

# Check log collection
sudo tail -f /var/log/alloy/collector.log

# Verify pipeline health
sudo alloy -config /etc/alloy/alloy.alloy -dry-run -debug
```

### 10. Remote On-Laptop: Backup and Restore

Manage Alloy configuration and data backups on the laptop.

- **Method:** SSH to management laptop
- **Command Pattern:**

```bash
# SSH to management laptop
ssh bjzy@laptop

# Backup configuration
cp ~/alloy/config/alloy.alloy ~/alloy/config/alloy.alloy.backup.$(date +%Y%m%d)

# Backup data
tar -czf ~/alloy/backups/alloy-data-$(date +%Y%m%d).tar.gz ~/alloy/data/

# Restore configuration
cp ~/alloy/config/alloy.alloy.backup.20240101 ~/alloy/config/alloy.alloy

# Check backup integrity
tar -tzf ~/alloy/backups/alloy-data-20240101.tar.gz | head -10
```

## Troubleshooting

### Alloy Service Won't Start

```bash
# Check service status and logs
sudo systemctl status alloy
sudo journalctl -u alloy -n 50

# Check configuration syntax
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Check permissions
sudo ls -la /etc/alloy/alloy.alloy
sudo ls -la /var/lib/alloy/

# Check for port conflicts
sudo netstat -tulpn | grep :12345
```

### Telemetry Not Flowing

```bash
# Check pipeline status
sudo alloy -config /etc/alloy/alloy.alloy -debug

# Verify remote connectivity
curl -I http://mimir.bjzy.me:9009/api/prom/push

# Check metrics being scraped
curl -s http://localhost:12345/metrics | grep prometheus_scrape

# Check log collection
sudo tail -f /var/log/alloy/collector.log
```

### High Resource Usage

```bash
# Check process resources
ps aux | grep alloy

# Monitor memory usage
watch -n 1 'ps aux | grep alloy'

# Check disk usage
sudo du -sh /var/lib/alloy/

# Check for memory leaks
sudo pmap -d $(pgrep alloy) | tail -1
```

### Configuration Issues

```bash
# Validate configuration
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Check configuration file
sudo cat /etc/alloy/alloy.alloy

# Test specific components
sudo alloy -config /etc/alloy/alloy.alloy -dry-run -debug

# Check for syntax errors
sudo alloy -config /etc/alloy/alloy.alloy -dry-run 2>&1 | grep -i error
```

### 11. Log Parsing Pipelines

Configure Alloy to parse structured logs and extract labels.

- **Method:** Alloy configuration
- **Purpose:** Extract fields from logs for better querying in Loki

**Command Pattern:**
```bash
# Example Alloy config for parsing JSON logs
# Note: Use a dedicated config file or templating tool (Ansible) for idempotent configuration
sudo tee /etc/alloy/conf.d/log-parsing.alloy > /dev/null <<'EOF'
loki.source.file "app_logs" {
  targets = [
    {__path__ = "/var/log/app/*.log"},
  ]
  forward_to = [loki.process.parse_json.receiver]
}

loki.process "parse_json" {
  stage.json {
    expressions = {
      level = "level",
      message = "message",
      request_id = "request_id",
    }
  }
  
  stage.labels {
    values = {
      level = "",
      request_id = "",
    }
  }
  
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "https://loki.bjzy.me/loki/api/v1/push"
    tls_config {
      insecure_skip_verify = true
    }
  }
}
EOF

# Validate new config
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Reload Alloy
sudo systemctl reload alloy
```

### 12. Metric Relabeling

Configure metric label transformations and filtering.

- **Method:** Alloy configuration
- **Purpose:** Clean up metrics before shipping to Prometheus/Mimir

**Command Pattern:**
```bash
# Example Alloy config for metric relabeling
# Note: Use a dedicated config file or templating tool (Ansible) for idempotent configuration
sudo tee /etc/alloy/conf.d/metric-relabeling.alloy > /dev/null <<'EOF'
prometheus.scrape "node_exporter" {
  targets = [
    {"__address__" = "localhost:9100"},
  ]
  forward_to = [prometheus.relabel.cleanup.receiver]
}

prometheus.relabel "cleanup" {
  rule {
    source_labels = ["__name__"]
    regex = "go_.*"
    action = "drop"
  }
  
  rule {
    source_labels = ["instance"]
    target_label = "hostname"
    replacement = "prod-node-01"
  }
  
  rule {
    source_labels = ["job"]
    target_label = "environment"
    replacement = "production"
  }
  
  forward_to = [prometheus.remote_write.mimir.receiver]
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "https://mimir.bjzy.me/api/v1/push"
    basic_auth {
      username = "alloy"
      password_file = "/etc/alloy/mimir-password"
    }
  }
}
EOF

# Validate new config
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Reload Alloy
sudo systemctl reload alloy

# Verify relabeling is working
curl -s http://localhost:12345/metrics | grep prometheus_relabel
```

## Quick Reference Commands

### Local On-Device Operations
```bash
# Service status
sudo systemctl status alloy

# Configuration validation
sudo alloy -config /etc/alloy/alloy.alloy -dry-run

# Check logs
sudo journalctl -u alloy -n 50

# Resource usage
ps aux | grep alloy
```

### Remote On-Laptop Operations
```bash
# SSH to laptop
ssh bjzy@laptop

# Check laptop Alloy
brew services list | grep alloy

# Validate config
alloy -config ~/alloy/config/alloy.alloy -dry-run

# Check logs
tail -f ~/alloy/logs/alloy.log
```

### Telemetry Verification
```bash
# Check metrics endpoint
curl -s http://localhost:12345/metrics

# Check pipeline health
alloy -config /path/to/alloy.alloy -debug

# Verify remote write
curl -s http://localhost:12345/metrics | grep remote_write
```

## Important Notes

- **Always verify environment** (local vs remote) before executing commands
- **Backup configuration** before making changes
- **Test configuration** with `-dry-run` before applying
- **Monitor resources** to avoid performance impact
- **Check connectivity** to remote backends regularly
- **Document changes** in Keep alerts or Notion for team visibility
- **Use separate configs** for production vs development environments

## Related Documentation

- **Grafana Alloy Documentation**: [https://grafana.com/docs/alloy/latest/](https://grafana.com/docs/alloy/latest/)
- **Alloy Configuration Reference**: [https://grafana.com/docs/alloy/latest/components/](https://grafana.com/docs/alloy/latest/components/)
- **Notion**: [Telemetry Pipeline Management — Home Lab Guide](https://www.notion.so/telemetry-pipeline-management)
- **GitHub Repo**: `homelab-playbooks` (BjzyLabs/ansible-homelab)
- **Local Docs**: `docs/ALLOY_DEPLOYMENT.md`, `docs/TELEMETRY_OPERATIONS.md`
