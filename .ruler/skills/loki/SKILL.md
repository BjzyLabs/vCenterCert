---
name: loki
description: Interacts with Grafana Loki for log aggregation, LogQL queries, log stream analysis, label filtering, metric extraction, and distributed logging operations. Use when user mentions Loki, logs, LogQL, log aggregation, log queries, or log analysis.
---

# Grafana Loki Log Management

## Instructions

Use this skill to interact with Grafana Loki using the `logcli` command-line tool for log aggregation, querying, and analysis. Loki provides a horizontally scalable, highly available, multi-tenant log aggregation system inspired by Prometheus.

### Configuration

The preferred method of interaction is the `logcli` CLI tool (installed on the host). Loki runs as part of the observability stack in the Home Lab, collecting logs from various services and applications.

**Smart Authentication:**
Before running queries, verify logcli can reach Loki and authentication is configured.

```bash
# Check if Loki is accessible
if ! logcli stats > /dev/null 2>&1; then
  echo "Loki not accessible. Checking configuration..."
  export LOKI_ADDR=https://loki.bjzy.me
  # Retrieve credentials from Vault if required
  export LOKI_USERNAME=$(vault kv get -field=read_user kvProd_v2/Loki/Application-Prod)
  export LOKI_PASSWORD=$(vault kv get -field=read_password kvProd_v2/Loki/Application-Prod)
fi

# Verify connectivity
logcli stats || echo "Loki connectivity issue"
```

### Certificate Handling (Homelab)

Loki may use self-signed certificates in the homelab environment. **Always configure the CA certificate rather than disabling TLS verification.**

```bash
# Configure logcli with homelab CA certificate (RECOMMENDED):
export LOKI_ADDR=https://loki.bjzy.me
export LOKI_CA_CERT=/etc/ssl/certs/homelab-ca.pem

# Test connection
logcli stats

# Alternative: Add homelab CA to system trust store
# macOS: sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /path/to/homelab-ca.pem
# Linux: sudo cp /path/to/homelab-ca.pem /usr/local/share/ca-certificates/homelab-ca.crt && sudo update-ca-certificates

# ⚠️ SECURITY: Never use LOKI_TLS_SKIP_VERIFY=true in production
# This disables certificate verification and enables man-in-the-middle attacks
```

#### Bjzy Labs defaults

- **Loki Cluster**: Part of the observability stack with Alloy for log collection
- **Common use cases**:
  - **Log Querying**: Run LogQL queries to search through aggregated logs
  - **Live Tailing**: Stream logs in real-time from services and applications
  - **Batch Export**: Download large volumes of logs for analysis and auditing
  - **Label Discovery**: Explore available labels and log streams
  - **Volume Analysis**: Understand log volume and storage patterns
- **Integration Points**:
  - **Alloy**: Log collection and forwarding to Loki
  - **Grafana**: Primary visualization and query interface
  - **Docker Swarm**: Container log collection
  - **Kubernetes**: Pod and service log aggregation

### Environment and Guardrails (Bjzy Labs)

- **Cluster Access**:
  - Loki server accessible via HTTP API
  - Local CLI access available from management workstations
  - HTTP API available at configured Loki endpoint
- **Security Rules**:
  - Verify Loki connectivity before running queries
  - Use time ranges to limit query scope and prevent resource exhaustion
  - Be cautious with broad queries that may return large result sets
  - Never run delete operations without proper verification
- **CLI Availability**:
  - The `logcli` CLI is installed on management workstations
  - Environment variable `LOKI_ADDR` should point to Loki endpoint
  - Verify access with `logcli stats`

### Standard Operating Procedure (SOP)

When asked to "Query logs," "Check Loki," or "Analyze log data":

1. **Verify Connectivity:** Run `logcli stats` to check Loki connection
2. **Identify Scope:** Determine time range, services, and log patterns needed
3. **Construct Query:** Build appropriate LogQL query with proper filters
4. **Execute Query:** Use logcli with appropriate output format and limits
5. **Analyze Results:** Review log entries, patterns, and anomalies
6. **Document Findings:** Log relevant insights and recommendations

## Examples

### 1. Check Loki Status

Verify Loki is operational and accessible.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Check Loki connection and basic stats
logcli stats

# Verify Loki version and connectivity
logcli --version

# Test basic query
logcli query '{job="varlogs"}' --limit=10
```

### 2. Discover Available Labels

Explore the label space to understand available log streams.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# List all labels
logcli labels

# List label values for specific label
logcli labels job

# Get series information
logcli series '{job="varlogs"}'

# Show label cardinality
logcli series '{job="varlogs"}' | wc -l
```

### 3. Basic Log Queries

Search through logs using LogQL filtering.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Query logs from specific job
logcli query '{job="varlogs"}'

# Query logs with specific text
logcli query '{job="varlogs"} |= "error"'

# Query logs with regex pattern
logcli query '{job="varlogs"} =~ "error.*failed"'

# Query logs from specific time range
logcli query '{job="varlogs"}' --since=1h

# Query logs with output format options
logcli query '{job="varlogs"}' --output=jsonl
```

### 4. Service-Specific Log Queries

Target logs from specific services or applications.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Docker container logs
logcli query '{container_name="nginx"}'

# Kubernetes pod logs
logcli query '{namespace="default",pod="my-app"}'

# Application logs by level
logcli query '{app="my-service"} |= "ERROR"'

# System service logs
logcli query '{job="systemd"} |= "service"'
```

### 5. Time-Based Analysis

Analyze logs within specific time windows.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Recent logs (last hour)
logcli query '{job="varlogs"}' --since=1h

# Logs from specific time range
logcli query '{job="varlogs"}' --from="2024-01-01T00:00:00Z" --to="2024-01-01T23:59:59Z"

# Live tail logs (similar to tail -f)
logcli query '{job="varlogs"}' --tail

# Logs with step interval for metrics
logcli query 'rate({job="varlogs"}[5m])'
```

### 6. Error and Exception Analysis

Focus on error patterns and exceptions.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Find all error logs
logcli query '{} |= "ERROR" or |= "FATAL"'

# HTTP error analysis
logcli query '{job="nginx"} |= "5[0-9][0-9]"'

# Application stack traces
logcli query '{app="my-service"} |= "Exception"' --output=raw

# Error rate calculation
logcli instant-query 'rate({job="varlogs"} |= "error"[5m])'
```

### 7. Performance and Resource Analysis

Analyze log volumes and patterns.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Get volume statistics
logcli volume

# Volume for specific query
logcli volume '{job="varlogs"}'

# Volume range analysis
logcli volume_range '{job="varlogs"}' --from="2024-01-01" --to="2024-01-02"

# Detected fields in logs
logcli detected-fields '{job="varlogs"}'
```

### 8. Log Export and Batch Operations

Export logs for offline analysis.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Export logs to file
logcli query '{job="varlogs"}' --output=raw > logs.txt

# Export JSON format
logcli query '{job="varlogs"}' --output=jsonl > logs.jsonl

# Export with large limit
logcli query '{job="varlogs"}' --limit=10000 --output=raw > large_logs.txt

# Export specific time window
logcli query '{job="varlogs"}' --from="2024-01-01T00:00:00Z" --to="2024-01-02T00:00:00Z" > daily_logs.txt
```

### 9. Advanced LogQL Queries

Use complex LogQL expressions for sophisticated analysis.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Log aggregation with count
logcli query 'count by (level) ({job="varlogs"})'

# Rate calculations
logcli query 'rate({job="varlogs"}[5m])'

# Log filtering with multiple conditions
logcli query '{job="varlogs", level="error"} |= "database"'

# Log parsing and extraction
logcli query '{job="varlogs"} | logfmt | level="error"'

# Time series from logs
logcli instant-query 'sum(rate({job="varlogs"}[5m])) by (job)'
```

### 10. Integration with Grafana

Understand how queries relate to Grafana dashboards.

- **Method:** `logcli` CLI
- **Command Pattern:**

```bash
# Test dashboard queries
logcli query '{job="varlogs"} |= "error"' --limit=100

# Validate label selectors
logcli labels job

# Check query performance
logcli stats --query='{job="varlogs"}'

# Explore available metrics
logcli query 'rate({job="varlogs"}[1h])'
```

### 11. Advanced LogQL Query Patterns

Complex LogQL queries for log analysis and troubleshooting.

- **Method:** `logcli` CLI
- **Purpose:** Extract insights from logs using LogQL syntax

**Command Pattern:**
```bash
# Count log lines by label
logcli query 'count_over_time({job="varlogs"}[1h])'

# Rate of log entries per second
logcli query 'rate({app="api"}[5m])'

# Filter logs with regex
logcli query '{job="varlogs"} |~ "error|ERROR|failed"' --since=1h

# Extract JSON fields
logcli query '{app="api"} | json | level="error"' --since=30m

# Aggregate by extracted field
logcli query 'sum by (status_code) (count_over_time({app="nginx"} | json [5m]))'

# Multi-line log parsing
logcli query '{job="app"} |= "stack trace" --context=10' --since=1h

# Calculate percentiles (requires unwrap for numeric field extraction)
logcli query 'quantile_over_time(0.95, {app="api"} | json | unwrap response_time [5m])'

# Combine multiple labels
logcli query '{app="api", environment="production"} | json | status_code >= 500' --since=24h --limit=100
```

## Troubleshooting

### Connection Issues

```bash
# Verify Loki endpoint
echo $LOKI_ADDR

# Test basic connectivity
curl -I $LOKI_ADDR/ready

# Check logcli configuration
logcli help

# Verify authentication (if required)
logcli stats --username=<user> --password=<pass>
```

### Query Performance Issues

```bash
# Check query stats
logcli stats --query='{job="varlogs"}'

# Limit time range for better performance
logcli query '{job="varlogs"}' --since=10m

# Use specific labels to reduce scope
logcli query '{job="varlogs", instance="host1"}'

# Check volume before running large queries
logcli volume '{job="varlogs"}'
```

### No Results Found

```bash
# Verify label names and values
logcli labels

# Check if job exists
logcli series '{job="varlogs"}'

# Expand time range
logcli query '{job="varlogs"}' --since=24h

# Try broader query
logcli query '{}'
```

### Large Result Sets

```bash
# Use limit to prevent overwhelming output
logcli query '{job="varlogs"}' --limit=100

# Use output formatting for better parsing
logcli query '{job="varlogs"}' --output=jsonl

# Export to file for large datasets
logcli query '{job="varlogs"}' --limit=10000 > large_output.txt

# Use tail for live monitoring
logcli query '{job="varlogs"}' --tail
```

### Label Cardinality Issues

```bash
# Check label cardinality
logcli series '{job="varlogs"}' | wc -l

# Find high-cardinality labels
logcli labels | head -20

# Use more specific queries
logcli query '{job="varlogs", instance="specific-host"}'

# Monitor label usage
logcli stats
```

## Common Service Patterns

### Docker Swarm Log Collection

Docker Swarm services forward logs to Loki via Alloy:

```bash
# Query specific service logs
logcli query '{com_docker_swarm_service="my-app"}'

# Query by container name
logcli query '{container_name="my-app"}'

# Query Swarm node logs
logcli query '{com_docker_swarm_node="worker1"}'
```

### System Log Analysis

System logs are collected and forwarded to Loki:

```bash
# Systemd service logs
logcli query '{job="systemd"} |= "nginx"'

# Authentication logs
logcli query '{job="auth"} |= "failed"'

# Kernel messages
logcli query '{job="kernel"} |= "error"'
```

### Application Log Patterns

Application logs follow structured formats:

```bash
# Structured JSON logs
logcli query '{app="my-service"} | json'

# Logfmt formatted logs
logcli query '{app="my-service"} | logfmt'

# Error tracking
logcli query '{app="my-service"} | logfmt | level="error"'
```

### Security and Audit Logs

Security events are captured and analyzed:

```bash
# Failed authentication attempts
logcli query '{job="auth"} |= "failed"'

# Suspicious activity patterns
logcli query '{job="security"} |= "suspicious"'

# Access log analysis
logcli query '{job="nginx"} |= "404"'
```

### Performance Monitoring

Use logs for performance insights:

```bash
# Response time analysis
logcli query '{app="api"} |= "response_time"'

# Database query logs
logcli query '{app="database"} |= "slow_query"'

# Resource utilization
logcli query '{job="system"} |= "memory"'
```

## LogQL Reference

### Basic Selectors

```bash
# Label selector
{job="varlogs"}

# Multiple labels
{job="varlogs", instance="localhost"}

# Regex label matching
{job=~"varlogs|system"}

# Text filtering
{job="varlogs"} |= "error"

# Regex text filtering
{job="varlogs"} =~ "error.*failed"
```

### Pipeline Operations

```bash
# Line format parsing
{job="varlogs"} | logfmt

# JSON parsing
{job="varlogs"} | json

# Field extraction
{job="varlogs"} | line_format "{{.message}}"

# Label extraction
{job="varlogs"} | label_format level="error"
```

### Aggregation Operations

```bash
# Count by labels
count by (level) ({job="varlogs"})

# Rate calculation
rate({job="varlogs"}[5m])

# Sum over time
sum_over_time({job="varlogs"}[5m])

# Top K
topk(10, {job="varlogs"})
```

## Best Practices

### Query Optimization

- Use specific time ranges to limit data scanned
- Include relevant labels to reduce query scope
- Use `limit` to prevent excessive result sets
- Check query volume before running large queries

### Label Management

- Monitor label cardinality to avoid performance issues
- Use consistent labeling strategies
- Avoid high-cardinality labels like user IDs
- Regularly review and clean up unused labels

### Security Considerations

- Be mindful of sensitive data in logs
- Use appropriate authentication for Loki access
- Consider log retention policies
- Audit access to log data

### Performance Monitoring

- Monitor Loki resource usage
- Track query performance metrics
- Set up alerts for high query latency
- Regularly review log volume patterns
