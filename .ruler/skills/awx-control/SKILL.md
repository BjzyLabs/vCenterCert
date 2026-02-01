---
name: awx-control
description: Interacts with AWX/Ansible Tower for automation workflow management, playbook execution, job monitoring, inventory operations, credential management, and infrastructure automation orchestration. Use when user mentions AWX, Ansible Tower, automation, playbooks, job templates, workflows, or infrastructure orchestration.
---

# AWX (Ansible Tower) Integration

## Instructions

Use this skill to interact with the AWX Automation Controller. AWX is the "Single Source of Truth" for all configuration management in the Home Lab.
**Always prefer launching a Job Template over manually editing files or restarting services via SSH.**

### Configuration

The preferred method of interaction is the `awx` CLI tool (installed on the host). If unavailable, fall back to the REST API.

**Smart Authentication:**
Before running commands, the agent must verify if the current session is valid using `awx me`. If that fails, retrieve the token from Vault.

### CLI vs API Decision Tree

**Always try CLI first. Use API only when CLI is unavailable or lacks the capability.**

| Operation | Use CLI | Use API |
|-----------|---------|--------|
| List/Get templates, jobs, inventories | ✅ `awx <resource> list/get` | ❌ |
| Launch job templates | ✅ `awx job_templates launch` | Only if CLI broken |
| Monitor running jobs | ✅ `awx job_templates launch --monitor` | ❌ |
| Create workflow templates | ✅ `awx workflow_job_templates create` | ❌ |
| Create approval nodes | ❌ CLI does not support | ✅ Required |
| Link workflow nodes | ❌ CLI does not support | ✅ Required |
| Approve/Deny workflow approvals | ❌ CLI does not support | ✅ Required |
| Get job stdout | ✅ `awx jobs stdout` | Only for very large logs |

**Rule:** If a CLI command exists for the operation, use it. Only fall back to API for operations the CLI cannot perform.

#### Bjzy Labs defaults

- **Controller URL**: `https://awx.bjzy.me` (Manages both Prod and Dev).
- **Vault lookup (for TOWER_OAUTH_TOKEN)**:
  - Path: `kvProd_v2/AWX/API`
  - Field: `api_token`

### Certificate Handling (Homelab)

The homelab AWX instance uses a self-signed certificate. **All `awx` CLI commands must include `--conf.insecure`** to bypass certificate verification.

```bash
# Correct pattern for ALL awx commands in this environment:
awx <command> --conf.insecure
```

This flag is included in all examples below. If you see SSL/certificate errors, ensure this flag is present.

### Environment and Guardrails (Bjzy Labs)

- **Managed Environments (Inventory IDs)**:
  - **⚠️ Production (ID: 15)**: Subnet `192.168.60.x`. Nodes: `Huey`, `Dewey`, `Louie`.
  - **Development (ID: 16)**: Subnet `192.168.50.x`. Nodes: `devHuey`, `devDewey`, `devLouie`.
- **Execution**:
  - The agent must **NEVER** run `ansible-playbook` manually on the CLI.
  - The agent must **NEVER** edit `/etc/` config files directly on nodes; changes must be made in the `homelab-playbooks` repo and deployed via AWX.
- **CLI Availability**:
  - The `awx` Python CLI is installed on the local MacBook.
  - Verify access with `awx me`.

### CRITICAL: Production Safety Protocol

**Before launching ANY job that targets Production (Inventory ID 15), the agent MUST:**

1. **Verify the target inventory** by inspecting the job template:
   ```bash
   awx job_templates get <ID> --conf.insecure | jq '{name, inventory}'
   ```

2. **Explicitly confirm with the user** using the `ask_questions` tool with a message like:
   > "This job targets **Production** (Inventory ID 15: Huey, Dewey, Louie). Do you want to proceed?"
   
   Options should include: "Yes, run on Production" and "No, switch to Development (ID 16)"

3. **Never assume production is intended.** If the user says "run this job" without specifying an environment, **default to Development (ID 16)** and confirm.

**Why this matters:** Production and Development inventory IDs differ by only one digit (15 vs 16). A single mistake can affect live services. When in doubt, ask.

**Preferred Pattern for Critical Production Jobs:**
Use AWX Workflow Job Templates with an **Approval Node** for platform-enforced human confirmation. See the "Production Workflow with Approval" example below.

### Standard Operating Procedure (SOP)

When asked to "Restart a service," "Update configuration," or "Fix a drift":

1. **Verify Auth:** Run `awx me`. If it fails, fetch the token from Vault.
2. **Search Templates:** Find the relevant Job Template ID for the task (e.g., "Restart Traefik", "System Updates").
3. **Launch & Monitor:** Launch the job using the CLI and **wait** for it to complete.
    - *Note: If targeting a specific environment (Dev vs Prod), verify the Job Template uses the correct Inventory ID (15 or 16).*
4. **Verify:** Once the job succeeds, verify the service health (optionally using the `keep-aiops` skill).

## Examples

### 1. Smart Authentication Check

Always run this check before attempting to launch jobs.

- **Method:** Bash Conditional
- **Command Pattern:**

```bash
# Check if current token is valid
if ! awx me --conf.insecure > /dev/null 2>&1; then
  echo "Token expired or missing. Fetching from Vault..."
  export TOWER_HOST=https://awx.bjzy.me
  export TOWER_OAUTH_TOKEN=$(vault kv get -field=api_token kvProd_v2/AWX/API)
fi
```

### 2. List Available Job Templates (Find the ID)

Use this to discover which job handles the requested task.

- **Method:** `awx` CLI
- **Command Pattern:**

```bash
# Search for the template
awx job_templates list --name "Traefik" --conf.insecure
# Output will provide the 'id' (e.g., 14) and 'inventory' (e.g., 15 for Prod)
```

### 3. Launch a Job Template (Preferred)

Trigger the playbook run and watch the output stream to confirm completion.

- **Method:** `awx` CLI with monitor
- **Command Pattern:**

```bash
# Launch Template ID 14 and follow logs
awx job_templates launch 14 --monitor --conf.insecure

# OPTIONAL: Override inventory to Development (ID 16) if the template allows prompt-on-launch
awx job_templates launch 14 --inventory 16 --monitor --conf.insecure
```

### 4. Check Status of a Running Job

If a job was launched without monitoring, or to check on a long-running system update.

- **Method:** `awx` CLI
- **Command Pattern:**

```bash
awx jobs get <JOB_ID> --format json --conf.insecure | jq '.status'
```

### 5. Fallback: API Launch (If CLI fails)

Use this only if the `awx` CLI tool is broken or missing.

- **Method:** `POST /api/v2/job_templates/{id}/launch/`
- **Command Pattern:**

```bash
# Get Token
TOKEN=$(vault kv get -field=api_token kvProd_v2/AWX/API)

# Launch Job 14
curl -X POST -k -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://awx.bjzy.me/api/v2/job_templates/14/launch/"
```

### 6. Troubleshooting: Check AWX Node Health

If AWX itself is unresponsive, you may verify the Docker containers on the Swarm nodes.

- **Method:** SSH (Read-Only)
- **Command Pattern:**

```bash
# Check Production Node (Huey)
ssh ansible@Huey "docker ps --format 'table {{.Names}}\t{{.Status}}' | rg awx"
```

### 7. Production Pre-Flight Check (REQUIRED)

Before launching any job on Production, verify the target inventory.

- **Method:** `awx` CLI inspection
- **Command Pattern:**

```bash
# Check which inventory a job template uses
awx job_templates get <TEMPLATE_ID> --conf.insecure | jq '{
  id,
  name,
  inventory,
  ask_inventory_on_launch
}'

# Verify inventory contents to confirm you're targeting the right hosts
awx hosts list --inventory 15 --conf.insecure | jq -r '.results[].name'
# Expected for Production: Huey, Dewey, Louie

awx hosts list --inventory 16 --conf.insecure | jq -r '.results[].name'
# Expected for Development: devHuey, devDewey, devLouie
```

**After running this check, you MUST confirm with the user before proceeding with Production.**

### 8. Production Workflow with Approval Node (Recommended for Critical Jobs)

For high-risk production operations, use a Workflow Job Template with an **Approval Node** that pauses execution until a human approves via the AWX UI or API.

- **Method:** AWX Workflow with approval step (100% CLI/API creation)

#### Create Approval Workflow via CLI/API

```bash
# Ensure token is set
export TOWER_HOST=https://awx.bjzy.me
export TOWER_OAUTH_TOKEN=$(vault kv get -field=api_token kvProd_v2/AWX/API)

# Step 1: Create the Workflow Job Template (CLI)
WORKFLOW_ID=$(awx workflow_job_templates create \
  --name "Deploy Traefik - PROD with Approval" \
  --organization 1 \
  --description "Production deployment with human approval gate" \
  --conf.insecure \
  -f json | jq -r '.id')
echo "Created workflow template ID: $WORKFLOW_ID"

# NOTE: Steps 2-5 REQUIRE the API because the awx CLI does not
# support creating approval nodes or linking workflow nodes.

# Step 2: Create the Approval Node (unified_job_template is NULL - this makes it an approval node)
APPROVAL_NODE_ID=$(curl -sk -X POST \
  -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "$TOWER_HOST/api/v2/workflow_job_templates/$WORKFLOW_ID/workflow_nodes/" \
  -d '{"identifier": "production-approval"}' | jq -r '.id')
echo "Created approval node ID: $APPROVAL_NODE_ID"

# Step 3: Configure the Approval Template (name, description, timeout)
curl -sk -X POST \
  -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "$TOWER_HOST/api/v2/workflow_job_template_nodes/$APPROVAL_NODE_ID/create_approval_template/" \
  -d '{
    "name": "Approve Production Deployment",
    "description": "⚠️ Human approval required before deploying to Production (Inventory ID 15)",
    "timeout": 0
  }'

# Step 4: Create the Job Template Node (after approval)
# Replace 14 with your actual job template ID
JOB_TEMPLATE_ID=14
JOB_NODE_ID=$(curl -sk -X POST \
  -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "$TOWER_HOST/api/v2/workflow_job_templates/$WORKFLOW_ID/workflow_nodes/" \
  -d "{\"identifier\": \"run-production-job\", \"unified_job_template\": $JOB_TEMPLATE_ID}" | jq -r '.id')
echo "Created job node ID: $JOB_NODE_ID"

# Step 5: Link Approval → Job on Success (approval granted)
curl -sk -X POST \
  -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "$TOWER_HOST/api/v2/workflow_job_template_nodes/$APPROVAL_NODE_ID/success_nodes/" \
  -d "{\"id\": $JOB_NODE_ID}"
echo "Linked approval → job on success"
```

#### Launch and Approve/Deny via CLI/API

```bash
# Launch the workflow (pauses at approval node)
awx workflow_job_templates launch $WORKFLOW_ID --conf.insecure
# Note the workflow job ID returned

# Check pending approvals
curl -sk -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  "$TOWER_HOST/api/v2/workflow_approvals/?status=pending" | jq '.results[] | {id, name, status}'

# ✅ APPROVE (via API) - User clicks approve in UI or runs this:
APPROVAL_ID=<pending_approval_id>
curl -sk -X POST \
  -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "$TOWER_HOST/api/v2/workflow_approvals/$APPROVAL_ID/approve/" \
  -d '{}'

# ❌ DENY (via API) - Cancels the workflow
curl -sk -X POST \
  -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "$TOWER_HOST/api/v2/workflow_approvals/$APPROVAL_ID/deny/" \
  -d '{}'
```

#### AWX UI Approval (Preferred)
Once the workflow is launched, navigate to **AWX → Workflow Jobs → [running job]** and click the approval node. The UI will show "Approve" and "Deny" buttons.

### 9. Safe Production Launch Pattern

The complete pattern for safely launching a production job:

- **Method:** Multi-step verification + confirmation
- **Command Pattern:**

```bash
# Step 1: Find the job template
awx job_templates list --name "Traefik" --conf.insecure

# Step 2: Inspect the template's default inventory
JT_ID=14
awx job_templates get $JT_ID --conf.insecure | jq '{name, inventory}'
# If inventory is 15 (Production), STOP and confirm with user

# Step 3: After user confirms Production is intended:
echo "⚠️  PRODUCTION DEPLOYMENT - User confirmed"
awx job_templates launch $JT_ID --monitor --conf.insecure

# Alternative: Force Development inventory (safer default)
awx job_templates launch $JT_ID --inventory 16 --monitor --conf.insecure
```

## Troubleshooting

### Certificate Verification Issues

If you encounter SSL certificate verification errors when running `awx` commands, you can bypass verification temporarily:

```bash
# Add --conf.insecure to any command
awx job_templates list --name "Traefik" --conf.insecure
awx job_templates launch 14 --monitor --conf.insecure
```

**Note:** This flag should only be used as a temporary workaround. The proper solution is to ensure your Execution Environment and local environment trust your homelab CA certificates. File an issue to track fixing the certificate trust chain if this becomes a persistent problem.

### Authentication Failures

**Symptom:** `401 Unauthorized` or `Authentication credentials were not provided`

**Solution:** Re-fetch token from Vault and re-export:

```bash
export TOWER_HOST=https://awx.bjzy.me
export TOWER_OAUTH_TOKEN=$(vault kv get -field=api_token kvProd_v2/AWX/API)
awx me --conf.insecure  # Verify auth works
```

### Wrong Inventory Selected

**Symptom:** Job runs on wrong hosts, or "No eligible hosts" error

**Solution:** Always verify inventory before launching:

```bash
# Check job template's default inventory
awx job_templates get <ID> --conf.insecure | jq '{name, inventory}'

# List hosts in that inventory
awx hosts list --inventory <INVENTORY_ID> --conf.insecure | jq -r '.results[].name'
```

**Prevention:** If `ask_inventory_on_launch` is True, always explicitly specify the inventory:
```bash
awx job_templates launch <ID> --inventory 16 --monitor --conf.insecure  # Force Dev
```

### Job Log Retrieval (Incomplete --monitor Output)

**Symptom:** Job shows as failed but `--monitor` output is truncated

**Solution:** Retrieve full stdout after job completes:

```bash
# Method 1: AWX CLI
awx jobs stdout <JOB_ID> --conf.insecure

# Method 2: API (for very large logs)
curl -sk -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
  "https://awx.bjzy.me/api/v2/jobs/<JOB_ID>/stdout/?format=txt_download" > job_output.log
```

### Survey Validation Errors

**Symptom:** `variables_needed_to_start` or `Value X expected to be one of [Y, Z]`

**Solution:**

1. Get survey spec first:
   ```bash
   awx job_templates get <ID> --conf.insecure | jq '.related.survey_spec'
   # Then fetch the actual spec:
   curl -sk -H "Authorization: Bearer $TOWER_OAUTH_TOKEN" \
     "https://awx.bjzy.me/api/v2/job_templates/<ID>/survey_spec/" | jq '.spec[]'
   ```

2. Match variable types exactly (e.g., `"True"` string not `true` boolean)
3. Provide all required variables in `--extra_vars`

### Project Sync Issues

**Symptom:** Job uses stale code after recent commits

**Solution:** Force a project sync before launching:

```bash
# Find the project ID from the job template
PROJECT_ID=$(awx job_templates get <JT_ID> --conf.insecure | jq -r '.project')

# Sync the project and wait
awx projects update $PROJECT_ID --conf.insecure --wait

# Verify sync succeeded
awx projects get $PROJECT_ID --conf.insecure | jq '{name, status, scm_revision}'
```

### CLI Limitations (When API is Required)

The `awx` CLI does not support all AWX operations. The following **require direct API calls**:

| Operation | API Endpoint |
|-----------|-------------|
| Create approval node | `POST /api/v2/workflow_job_templates/{id}/workflow_nodes/` with no `unified_job_template` |
| Configure approval template | `POST /api/v2/workflow_job_template_nodes/{id}/create_approval_template/` |
| Link workflow nodes | `POST /api/v2/workflow_job_template_nodes/{id}/success_nodes/` |
| Approve workflow | `POST /api/v2/workflow_approvals/{id}/approve/` |
| Deny workflow | `POST /api/v2/workflow_approvals/{id}/deny/` |

For these operations, use `curl` with the `$TOWER_OAUTH_TOKEN` as shown in Example 8.
