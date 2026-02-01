---
name: grafana-mcp
description: Manages Grafana dashboards, data sources, users, alerts, and configuration through the Grafana MCP Server for visualization and monitoring operations. Use when user mentions Grafana, dashboards, metrics visualization, alerts, monitoring panels, or Grafana API interactions.
---

# Grafana MCP Server

## Instructions

Use this skill to interact with Grafana instances through the **Grafana MCP Server** - a Model Context Protocol server that provides seamless access to Grafana's HTTP API for dashboards, alerts, data sources, and monitoring configuration.

### Configuration

The primary interface is the **Grafana MCP Server** deployed via AWX. This MCP server wraps Grafana's HTTP API, providing structured access to all Grafana resources without needing to manage authentication tokens or API endpoints manually.

#### Bjzy Labs defaults

- **Grafana MCP Server**:
  - **AWX Job Template**: ID 49 ("Grafana MCP")
  - **Playbook**: `playbooks/GrafanaMCP-manage.yml`
  - **Purpose**: Deploy/manage the Grafana MCP Server on target hosts
  - **Deployment Options**: Container or binary installation
  - **Default Inventory**: MCP - Dev (ID: 12)

- **Grafana Instances**: Multiple environments in the Home Lab
  - **Development**: `https://devgrafana.bjzy.me`
  - **Production**: `https://grafana.bjzy.me`
  - **Deployment**: Docker Swarm with HA configuration

- **Common use cases**:
  - **Dashboard Management**: Import, export, and manage dashboards as code
  - **Alert Configuration**: Set up and manage alert rules and notification channels
  - **Data Source Administration**: Configure and monitor data sources (Prometheus, Loki, Mimir)
  - **User Management**: Handle user accounts, teams, and permissions
  - **Monitoring Integration**: Connect with Keep AIOps for alert management

- **Integration Points**:
  - **Prometheus/Mimir**: Primary metrics data sources
  - **Loki**: Log aggregation and querying (3-node HA cluster)
  - **Alloy**: Telemetry pipeline feeding Grafana
  - **Keep AIOps**: Alert management and incident response
  - **Home Lab Monitoring**: Unified observability dashboard

### Environment and Guardrails (Bjzy Labs)

- **MCP Server Access**:
  - MCP server handles authentication and environment management automatically
  - Deployed via AWX Job Template 49
  - Configuration managed through Ansible playbooks
  - Multiple environments supported (dev, staging, production)

- **Security Rules**:
  - Always verify the target environment before making changes
  - Use read-only operations for dashboard and data source exploration
  - Be cautious with alert rule modifications (affect monitoring coverage)
  - Test dashboard imports in dev before production deployment
  - Credentials stored in HashiCorp Vault (`kvProd_v2/Grafana/`)

- **Deployment Rules**:
  - Use AWX Job Template 49 to deploy/manage the Grafana MCP Server
  - Never manually configure Grafana API tokens outside of Vault
  - Follow standard MCP server deployment patterns (see `standards/05_MCP_SETUP.md`)

### Standard Operating Procedure (SOP)

When asked to "Manage Grafana," "Configure alerts," "Handle dashboards," or "Access Grafana API":

1. **Discover MCP Server:** The Grafana MCP Server provides structured access to Grafana's API - check if it's enabled in your MCP configuration
2. **Verify Deployment:** If needed, deploy/verify MCP server via AWX Job Template 49
3. **Identify Operation:** Determine if you need dashboard, alert, data source, or user management
4. **Use MCP Interface:** Interact through the MCP server which handles authentication and API calls
5. **Verify Results:** Cross-check changes in Grafana UI when possible
6. **Document Changes:** Log any modifications to monitoring configurations

## Examples

### 1. Deploy Grafana MCP Server

Deploy or manage the Grafana MCP Server via AWX.

- **Method:** AWX CLI
- **Command Pattern:**

```bash
# Deploy Grafana MCP Server (container mode)
awx job_template launch 49 \
  --extra_vars '{
    "operation": "install",
    "deployment_mode": "container"
  }' \
  --monitor

# Check MCP Server status
awx job_template launch 49 \
  --extra_vars '{
    "operation": "status"
  }' \
  --monitor

# Uninstall MCP Server
awx job_template launch 49 \
  --extra_vars '{
    "operation": "uninstall"
  }' \
  --monitor
```

### 2. Dashboard Management (via HTTP API)

Access Grafana dashboards using HTTP API when direct access is needed.

- **Method:** Grafana HTTP API
- **Command Pattern:**

```bash
# Get Grafana API token from Vault
GRAFANA_TOKEN=$(vault kv get -field=api_token kvProd_v2/Grafana/API)
GRAFANA_URL="https://grafana.bjzy.me"

# List all dashboards
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/search?type=dash-db" | jq '.'

# Get specific dashboard by UID
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/dashboards/uid/<dashboard_uid>" | jq '.'

# Export dashboard to JSON file
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/dashboards/uid/<dashboard_uid>" \
  | jq '.dashboard' > dashboard-backup.json

# Import dashboard from JSON
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @dashboard.json \
  "$GRAFANA_URL/api/dashboards/db"

# Delete dashboard
curl -X DELETE -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/dashboards/uid/<dashboard_uid>"
```

### 3. Alert Rule Management

Configure and manage alert rules via HTTP API.

- **Method:** Grafana HTTP API
- **Command Pattern:**

```bash
# List all alert rules
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/v1/provisioning/alert-rules" | jq '.'

# Get specific alert rule
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/v1/provisioning/alert-rules/<rule_uid>" | jq '.'

# Create alert rule from file
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @alert-rule.json \
  "$GRAFANA_URL/api/v1/provisioning/alert-rules"

# Update alert rule
curl -X PUT -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @alert-rule.json \
  "$GRAFANA_URL/api/v1/provisioning/alert-rules/<rule_uid>"

# Delete alert rule
curl -X DELETE -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/v1/provisioning/alert-rules/<rule_uid>"
```

### 4. Data Source Administration

Manage Grafana data sources via HTTP API.

- **Method:** Grafana HTTP API
- **Command Pattern:**

```bash
# List all data sources
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources" | jq '.'

# Get data source by UID
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources/uid/<datasource_uid>" | jq '.'

# Create data source
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @datasource.json \
  "$GRAFANA_URL/api/datasources"

# Update data source
curl -X PUT -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @datasource.json \
  "$GRAFANA_URL/api/datasources/uid/<datasource_uid>"

# Test data source connectivity
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources/uid/<datasource_uid>/resources/test" | jq '.'

# Delete data source
curl -X DELETE -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources/uid/<datasource_uid>"
```

### 5. User and Organization Management

Manage users and organizations via HTTP API.

- **Method:** Grafana HTTP API
- **Command Pattern:**

```bash
# List all users
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/users" | jq '.'

# Get current user info
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/user" | jq '.'

# List organization users
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/org/users" | jq '.'

# Create user
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com","login":"john","password":"password"}' \
  "$GRAFANA_URL/api/admin/users"

# List teams
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/teams/search" | jq '.'
```

### 6. Folder Management

Organize dashboards in folders via HTTP API.

- **Method:** Grafana HTTP API
- **Command Pattern:**

```bash
# List all folders
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/folders" | jq '.'

# Get folder by UID
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/folders/<folder_uid>" | jq '.'

# Create folder
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Infrastructure"}' \
  "$GRAFANA_URL/api/folders"

# Update folder
curl -X PUT -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Production Infrastructure","overwrite":true}' \
  "$GRAFANA_URL/api/folders/<folder_uid>"

# Delete folder
curl -X DELETE -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/folders/<folder_uid>"
```

### 7. Health and Status Monitoring

Check Grafana instance health and configuration.

- **Method:** Grafana HTTP API
- **Command Pattern:**

```bash
# Check instance health (no auth required)
curl "$GRAFANA_URL/api/health" | jq '.'

# Get instance statistics
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/admin/stats" | jq '.'

# Get Grafana build info
curl "$GRAFANA_URL/api/frontend/settings" | jq '.buildInfo'

# Check data source health
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources/uid/<datasource_uid>/health" | jq '.'
```

### 8. Backup and Export Operations

Perform bulk export operations for backup purposes.

- **Method:** Grafana HTTP API + Shell Script
- **Command Pattern:**

```bash
# Export all dashboards to directory
GRAFANA_TOKEN=$(vault kv get -field=api_token kvProd_v2/Grafana/API)
GRAFANA_URL="https://grafana.bjzy.me"
BACKUP_DIR="./grafana-backup-$(date +%Y%m%d)"

mkdir -p "$BACKUP_DIR/dashboards"

# Get all dashboard UIDs
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/search?type=dash-db" \
  | jq -r '.[].uid' > "$BACKUP_DIR/dashboard-uids.txt"

# Export each dashboard
while read -r uid; do
  curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
    "$GRAFANA_URL/api/dashboards/uid/$uid" \
    | jq '.dashboard' > "$BACKUP_DIR/dashboards/$uid.json"
done < "$BACKUP_DIR/dashboard-uids.txt"

# Export all data sources
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources" \
  | jq '.' > "$BACKUP_DIR/datasources.json"

# Export alert rules
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/v1/provisioning/alert-rules" \
  | jq '.' > "$BACKUP_DIR/alert-rules.json"

echo "Backup completed: $BACKUP_DIR"
```

## Troubleshooting

### MCP Server Deployment Issues

```bash
# Check MCP Server deployment status via AWX
awx job_template launch 49 \
  --extra_vars '{"operation": "status"}' \
  --monitor

# Check Docker service if deployed as container
ssh ansible@<mcp-host> "docker ps | grep grafana-mcp"

# View MCP Server logs (if container deployment)
ssh ansible@<mcp-host> "docker logs grafana-mcp-server"

# Redeploy MCP Server
awx job_template launch 49 \
  --extra_vars '{"operation": "install", "deployment_mode": "container"}' \
  --monitor
```

### Authentication Issues

```bash
# Verify Grafana API token from Vault
vault kv get kvProd_v2/Grafana/API

# Test token validity
GRAFANA_TOKEN=$(vault kv get -field=api_token kvProd_v2/Grafana/API)
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/user

# Check user permissions
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/user/orgs | jq '.'

# Generate new API token (via Grafana UI)
# Navigate to: Configuration → API Keys → Add API key
```

### Dashboard Import/Export Failures

```bash
# Validate dashboard JSON structure
jq '.' dashboard.json

# Check required data sources exist
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/datasources | jq '.[].type'

# Import with overwrite flag
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dashboard": '"$(cat dashboard.json)"', "overwrite": true}' \
  https://grafana.bjzy.me/api/dashboards/db

# Check import error details
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @dashboard.json \
  https://grafana.bjzy.me/api/dashboards/db | jq '.message'
```

### Alert Rule Configuration Issues

```bash
# Validate alert rule syntax
jq '.' alert-rule.json

# Check alert rule query expressions
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/v1/provisioning/alert-rules/<rule_uid> \
  | jq '.data'

# Test alert rule evaluation
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/v1/eval

# List contact points (notification channels)
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/v1/provisioning/contact-points | jq '.'

# Check alert rule state
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/v1/provisioning/alert-rules/<rule_uid> \
  | jq '.state'
```

### Data Source Connection Problems

```bash
# Test data source connectivity
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/datasources/uid/<datasource_uid>/health | jq '.'

# Check data source configuration
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/datasources/uid/<datasource_uid> | jq '.'

# Test Prometheus data source query
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"queries":[{"refId":"A","expr":"up"}]}' \
  https://grafana.bjzy.me/api/ds/query

# Test Loki data source query
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"queries":[{"refId":"A","expr":"{job=\"varlogs\"}"}]}' \
  https://grafana.bjzy.me/api/ds/query
```

### Grafana Instance Health Check

```bash
# Check overall Grafana health (no auth needed)
curl https://grafana.bjzy.me/api/health | jq '.'

# Check if Grafana is reachable
curl -I https://grafana.bjzy.me

# Check Docker Swarm service status
ssh ansible@Huey "docker service ps grafana --no-trunc"

# Check Grafana service logs via Docker Swarm
ssh ansible@Huey "docker service logs grafana --tail 50"

# Verify Grafana database connectivity (PostgreSQL)
ssh ansible@Huey "docker exec \$(docker ps -q -f name=grafana) \
  grafana-cli admin data-migration list"
```

## Common Service Patterns

### Home Lab Integration

Grafana integrates with various Home Lab services and data sources:

```bash
GRAFANA_TOKEN=$(vault kv get -field=api_token kvProd_v2/Grafana/API)
GRAFANA_URL="https://grafana.bjzy.me"

# List Prometheus/Mimir data sources
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources" | jq '.[] | select(.type=="prometheus")'

# List Loki log sources
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/datasources" | jq '.[] | select(.type=="loki")'

# Find Home Lab infrastructure dashboards
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/search?query=Home%20Lab" | jq '.'

# List dashboards in Infrastructure folder
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/search?folderIds=<folder_id>" | jq '.'
```

### Multi-Environment Management

Manage development and production Grafana instances:

```bash
# Development environment
DEV_TOKEN=$(vault kv get -field=api_token kvDev_v2/Grafana/API)
DEV_URL="https://devgrafana.bjzy.me"

# Production environment
PROD_TOKEN=$(vault kv get -field=api_token kvProd_v2/Grafana/API)
PROD_URL="https://grafana.bjzy.me"

# Export dashboards from dev
curl -H "Authorization: Bearer $DEV_TOKEN" \
  "$DEV_URL/api/search?type=dash-db" | jq -r '.[].uid' | \
while read uid; do
  curl -H "Authorization: Bearer $DEV_TOKEN" \
    "$DEV_URL/api/dashboards/uid/$uid" | \
    jq '.dashboard' > "dev-$uid.json"
done

# Import validated dashboards to production
for file in dev-*.json; do
  curl -X POST -H "Authorization: Bearer $PROD_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"dashboard": '"$(cat "$file")"', "overwrite": false}' \
    "$PROD_URL/api/dashboards/db"
done
```

### Alert Integration with Keep AIOps

Configure Grafana alerts to send notifications to Keep:

```bash
GRAFANA_TOKEN=$(vault kv get -field=api_token kvProd_v2/Grafana/API)
KEEP_WEBHOOK_URL="https://keep-api.bjzy.me/webhook/grafana"

# Create Keep contact point (webhook receiver)
curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Keep AIOps",
    "type": "webhook",
    "settings": {
      "url": "'"$KEEP_WEBHOOK_URL"'",
      "httpMethod": "POST"
    }
  }' \
  https://grafana.bjzy.me/api/v1/provisioning/contact-points

# List existing contact points
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/v1/provisioning/contact-points | jq '.'

# Update alert rule to use Keep contact point
curl -X PUT -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @alert-with-keep.json \
  https://grafana.bjzy.me/api/v1/provisioning/alert-rules/<rule_uid>
```

### Observability-as-Code Workflows

Implement GitOps for Grafana configurations using AWX and version control:

```bash
# 1. Export current Grafana state
BACKUP_DIR="./grafana-config-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"/{dashboards,datasources,alerts}

# Export dashboards
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/search?type=dash-db | jq -r '.[].uid' | \
while read uid; do
  curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
    "https://grafana.bjzy.me/api/dashboards/uid/$uid" | \
    jq '.dashboard' > "$BACKUP_DIR/dashboards/$uid.json"
done

# Export data sources
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/datasources \
  > "$BACKUP_DIR/datasources/all.json"

# Export alert rules
curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  https://grafana.bjzy.me/api/v1/provisioning/alert-rules \
  > "$BACKUP_DIR/alerts/rules.json"

# 2. Commit to version control
cd "$BACKUP_DIR"
git init
git add .
git commit -m "Grafana config snapshot $(date +%Y%m%d)"

# 3. Apply from version control (GitOps)
# This would typically be automated via AWX/Ansible
for dashboard in dashboards/*.json; do
  curl -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"dashboard": '"$(cat "$dashboard")"', "overwrite": true}' \
    https://grafana.bjzy.me/api/dashboards/db
done
```

### MCP Server Usage Pattern

When the Grafana MCP Server is enabled in your agent configuration:

```text
# User workflow:
1. User mentions: "Show me Grafana dashboards" or "Configure Grafana alerts"
2. Agent discovers Grafana MCP Server is available
3. MCP Server handles:
   - Authentication (reads from Vault automatically)
   - Environment selection (dev vs prod)
   - API request construction
   - Response formatting
4. User gets structured results without managing tokens/URLs

# Deployment workflow:
1. Deploy via AWX: awx job_template launch 49
2. MCP server reads Grafana credentials from Vault
3. Exposes structured tools for dashboards, alerts, data sources
4. Agent uses MCP tools instead of raw HTTP API calls
```

## Best Practices

### MCP Server Deployment

- Deploy Grafana MCP Server via AWX Job Template 49 (never manually configure)
- Use container deployment mode for Docker Swarm environments
- Store all credentials in HashiCorp Vault (`kvProd_v2/Grafana/` and `kvDev_v2/Grafana/`)
- Follow standard MCP server deployment patterns (see `standards/05_MCP_SETUP.md`)
- Keep MCP server disabled by default, enable only when needed

### Security

- Use service account API tokens with minimal required permissions
- Rotate API tokens regularly via Vault
- Never hardcode tokens in scripts or configuration files
- Use Vault CLI to retrieve tokens at runtime: `vault kv get -field=api_token kvProd_v2/Grafana/API`
- Always verify target environment (dev vs prod) before making changes
- Enable audit logging for all API operations

### Performance

- Use bulk export operations for multiple dashboards (loop with pagination)
- Cache frequently accessed dashboard/data source metadata locally
- Use `jq` to filter API responses and reduce data transfer
- Limit search results with query parameters when possible
- Use folder-based organization to reduce dashboard list size

### Reliability

- Always test dashboard/alert changes in development before production
- Validate JSON configuration files with `jq` before importing
- Maintain regular backups of critical dashboards and alert rules
- Use version control (Git) for all Grafana configurations
- Document all manual API changes for AWX reconciliation

### Automation

- Integrate Grafana HTTP API calls in AWX playbooks for repeatable deployments
- Use Ansible `uri` module for API calls within playbooks
- Implement health checks before making configuration changes
- Use `--dry-run` patterns (check before apply) when possible
- Automate backup operations on a schedule via AWX

### Grafana HTTP API Reference

Official documentation: https://grafana.com/docs/grafana/latest/developers/http_api/

**Common API endpoints:**
- **Dashboards**: `/api/dashboards/*`, `/api/search`
- **Data Sources**: `/api/datasources/*`
- **Alerts**: `/api/v1/provisioning/alert-rules/*`
- **Users**: `/api/users/*`, `/api/org/users/*`
- **Folders**: `/api/folders/*`
- **Health**: `/api/health` (no auth required)
- **Admin**: `/api/admin/*` (requires admin permissions)
