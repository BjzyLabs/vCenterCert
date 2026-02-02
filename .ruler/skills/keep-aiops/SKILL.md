---
name: keep-aiops
description: Interacts with Keep AIOps platform for alert management, workflow automation, incident response orchestration, and intelligent alert routing across multiple monitoring sources. Use when user mentions Keep, AIOps, alert aggregation, incident workflows, or alert routing.
---

# Keep AIOps Integration

## Instructions

Use this skill to interact with the Keep AIOps platform. Keep is the "Single Source of Truth" for alerts, investigation context, and automated workflows.
**Always check Keep before attempting to manually SSH into servers or check raw logs.**

### Configuration
Ensure the following environment variables are set in the agent's context:
- `KEEP_API_URL`: The base URL of the Keep instance (e.g., `http://keep-backend:8080`)
- `KEEP_API_KEY`: The authentication token.

#### Bjzy Labs defaults (if not provided)
- Prod API: `https://keep-api.bjzy.me`
- Dev API: `https://devkeep-api.bjzy.me`
- Vault lookup (no secrets in logs):
  - Prod: `kvProd_v2/Keep/Application-Prod` field `keep_alertmanager_api_key`
  - Dev: `kvProd_v2/Keep/Application-Dev` field `keep_alertmanager_api_key`

### Environment and Guardrails (Bjzy Labs)
- **Deployment method**: Use AWX job templates (do not run `docker stack deploy` manually).
- **Keep runs on Docker Swarm nodes**:
  - Dev: `devHuey`, `devDewey`, `devLouie` (192.168.50.81-83)
  - Prod: `Huey`, `Dewey`, `Louie` (192.168.60.81-83)
- **Frontend URLs**:
  - Dev UI: `https://devkeep.bjzy.me`
  - Prod UI: `https://keep.bjzy.me`
- **API URLs**:
  - Dev API: `http://192.168.50.81:8085`
  - Prod API: `https://keep-api.bjzy.me`
- **Ports and healthchecks (direct node access)**:
  - UI: `:3000`
  - API: `:8085` with `/healthcheck`
  - WebSocket: `:8081`
- **Load balancer/HAProxy**:
  - HAProxy routes Keep UI/API by SNI hostnames.
  - Frontend hosts: `devkeep.bjzy.me`, `devkeep-api.bjzy.me`, `keep.bjzy.me`, `keep-api.bjzy.me`.
  - Backend targets: Docker Swarm nodes (Dev: 192.168.50.81-83, Prod: 192.168.60.81-83).
  - Config source of truth: `roles/haproxy/tasks/configure_keep_frontend.yml`.
- **Database (Keep usage only)**:
  - Keep uses a Patroni PostgreSQL cluster behind HAProxy.
  - Standard connection pattern: `postgresql://keep_user:***@haproxy.bjzy.me:5433/keep_db`.
  - Avoid direct DB changes unless explicitly requested; prefer AWX playbooks and Keep API.
- **SSH is allowed for read-only verification**. Example checks:
  - `ssh ansible@devHuey "docker ps --format 'table {{.Names}}\t{{.Status}}' | rg keep"`
  - `ssh ansible@Huey "curl -fsS http://localhost:8085/healthcheck"`
  - `ssh ansible@devDewey "sudo ss -lntp | rg ':8085|:3000|:8081'"`
- **Hostname safety**: `devHuey` != `Huey` (dev vs prod are different).

#### If deeper operational context is needed
- Consult `docs/KEEP_DEPLOYMENT.md` and `docs/KEEP_DNS_HAPROXY_SETUP.md` for storage, HAProxy, and service routing details.

### Standard Operating Procedure (SOP)
When asked to "Investigate an outage" or "Check alerts":
1. **Query Active Alerts:** Fetch firing alerts from Keep to determine the scope.
2. **Check Incidents/Rules:** If workflows are incident-triggered, confirm incidents exist and correlation rules are configured.
3. **Check Workflow Status:** See if Keep has already triggered an auto-remediation workflow for this alert ID.
4. **Enrich:** If you discover new info (e.g., by checking a log file elsewhere), POST that info back to the Alert using the enrichment endpoint to maintain the Single Source of Truth.

## Examples

### 1. Retrieve Active Alerts
Use this to identify what is currently firing or to find specific historical alerts.

- **Method:** `GET /alerts`
- **Useful Filters (Query Params):**
  - `?status=firing`
  - `?source=["prometheus", "grafana"]`
- **Command Pattern:**
```bash
curl -H "X-API-KEY: $KEEP_API_KEY" "$KEEP_API_URL/alerts?status=firing"
```

### 2. Trigger Investigation/Remediation Workflows
If a known issue is detected, trigger a workflow rather than running manual scripts.

- **Method:** `POST /workflows/{workflow_id}/run`
- **Command Pattern:**
```bash
curl -X POST -H "X-API-KEY: $KEEP_API_KEY" "$KEEP_API_URL/workflows/investigate-service-down/run" -d '{"alert_id": "12345"}'
```

### 3. Enrich an Alert (Add Investigation Notes)
Store your findings directly in Keep so future agents (or humans) can see what was analyzed.

- **Method:** `POST /alerts/{alert_id}/enrich`
- **Command Pattern:**
```bash
curl -X POST -H "X-API-KEY: $KEEP_API_KEY" "$KEEP_API_URL/alerts/{alert_id}/enrich" \
-d '{"enrichment": "Agent Analysis: Root cause appears to be OOM kill on worker-node-01"}'
```

### 4. Quick Actions (do not echo secrets)
- List current firing alerts (X-API-KEY is required in this environment):
```bash
curl -ksS -H "X-API-KEY: $KEEP_API_KEY" "$KEEP_API_URL/alerts?status=firing" | jq
```
- Confirm incidents exist (incident-triggered workflows only fire when incidents are created):
```bash
curl -ksS -H "X-API-KEY: $KEEP_API_KEY" "$KEEP_API_URL/incidents" | jq length
```
- Confirm correlation rules exist (rules create incidents from alerts):
```bash
curl -ksS -H "X-API-KEY: $KEEP_API_KEY" "$KEEP_API_URL/rules" | jq length
```
- Dismiss a specific alert (use `event_id` as `alert_id`):
```bash
curl -ksS -X POST -H "X-API-KEY: $KEEP_API_KEY" -H "Content-Type: application/json" \
  "$KEEP_API_URL/alerts/event/error/dismiss" \
  -d '{"alert_id": "uuid-from-firing-alert"}'
```
