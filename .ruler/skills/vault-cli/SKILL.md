---
name: vault-cli
description: Interacts with HashiCorp Vault for secrets management, authentication, encryption, policy administration, and secure credential operations including reading, writing, rotating, and managing access controls. Use when user mentions Vault, secrets, credentials, encryption, policies, or secure storage.
---

# HashiCorp Vault CLI Integration

## Instructions

Use this skill to interact with HashiCorp Vault for secure secrets management. Vault is the "Single Source of Truth" for all credentials, API keys, tokens, and sensitive configuration in the Home Lab.
**Always retrieve secrets from Vault at runtime. NEVER hardcode secrets in code, logs, or environment files.**

### Configuration

The preferred method of interaction is the `vault` CLI tool (installed on the host). If unavailable, fall back to the REST API.

**Smart Authentication:**
Before running commands, the agent must verify if the current session is valid using `vault token lookup`. If that fails, authenticate using the appropriate method.

#### Bjzy Labs defaults

- **Vault URL**: `https://vault.bjzy.me:8200`
- **KV Secrets Engine**: `kvProd_v2` (KV version 2)
- **Common secret paths**:
  - `kvProd_v2/AWX/Application-Prod` - AWX tokens
  - `kvProd_v2/Keep/Application-Prod` - Keep API keys
  - `kvProd_v2/Keep/Application-Dev` - Keep Dev API keys
  - `kvProd_v2/Grafana/Application-Prod` - Grafana credentials
  - `kvProd_v2/PostgreSQL/Application-Prod` - Database credentials

### Environment and Guardrails (Bjzy Labs)

- **Authentication Methods**:
  - **Production/AWX**: AppRole authentication (automated, short-lived tokens)
  - **Development/Agent**: `vault login -method=userpass` (interactive, short TTL)
  - **NEVER**: Use root token on disk or in scripts
- **Security Rules**:
  - The agent must **NEVER** echo secrets to stdout or logs
  - The agent must **NEVER** store secrets in files (use environment variables or direct injection)
  - The agent must **NEVER** commit secrets to version control
  - Always use `-field=` to extract specific values (avoids exposing full secret metadata)
- **Token Management**:
  - Tokens have TTLs (time-to-live); re-authenticate if expired
  - Prefer short-lived tokens for automation
  - Revoke tokens when no longer needed
- **CLI Availability**:
  - The `vault` CLI is installed on the local MacBook
  - Environment variable `VAULT_ADDR` should be set to `https://vault.bjzy.me:8200`
  - Verify access with `vault token lookup`

### Standard Operating Procedure (SOP)

When asked to "Get a secret," "Retrieve credentials," or "Check Vault access":

1. **Verify Auth:** Run `vault token lookup`. If it fails, authenticate using userpass.
2. **Identify Path:** Determine the correct secret path (check Bjzy Labs defaults above).
3. **Retrieve Secret:** Use `vault kv get -field=<field_name> <path>` to get specific values.
4. **Use Securely:** Pass the secret directly to the consuming command (pipe or subshell).
5. **Never Log:** Ensure the secret is not echoed or logged.

## Examples

### 1. Smart Authentication Check

Always run this check before attempting to retrieve secrets.

- **Method:** Bash Conditional
- **Command Pattern:**

```bash
# Check if current token is valid
if ! vault token lookup > /dev/null 2>&1; then
  echo "Token expired or missing. Authenticating..."
  export VAULT_ADDR=https://vault.bjzy.me:8200
  vault login -method=userpass username=$USER
fi
```

### 2. Retrieve a Specific Secret Field

Use this to get a single value from a secret (preferred method).

- **Method:** `vault` CLI with field extraction
- **Command Pattern:**

```bash
# Get a specific field (does not expose other fields)
vault kv get -field=token kvProd_v2/AWX/Application-Prod

# Get API key for Keep
vault kv get -field=keep_alertmanager_api_key kvProd_v2/Keep/Application-Prod
```

### 3. Use Secret in Another Command (Secure Pattern)

Pass secrets directly to commands without storing in variables when possible.

- **Method:** Command substitution
- **Command Pattern:**

```bash
# Direct injection into environment (preferred)
export TOWER_OAUTH_TOKEN=$(vault kv get -field=token kvProd_v2/AWX/Application-Prod)

# Direct injection into curl (no intermediate variable)
curl -H "X-API-KEY: $(vault kv get -field=keep_alertmanager_api_key kvProd_v2/Keep/Application-Prod)" \
  "https://keep-api.bjzy.me/alerts"
```

### 4. List Available Secrets at a Path

Use this to discover what secrets exist under a path.

- **Method:** `vault` CLI
- **Command Pattern:**

```bash
# List secrets at a path (does not reveal values)
vault kv list kvProd_v2/

# List secrets under a specific application
vault kv list kvProd_v2/Keep/
```

### 5. View Secret Metadata (Without Values)

Check when a secret was last updated without exposing the value.

- **Method:** `vault` CLI
- **Command Pattern:**

```bash
# Get metadata only (safe to log)
vault kv metadata get kvProd_v2/AWX/Application-Prod
```

### 6. Check Available Secret Fields

See what fields exist in a secret (field names only, not values).

- **Method:** `vault` CLI with jq
- **Command Pattern:**

```bash
# List field names (keys) without values
vault kv get -format=json kvProd_v2/AWX/Application-Prod | jq -r '.data.data | keys[]'
```

### 7. Authenticate with AppRole (Automation)

For automated systems like AWX that use AppRole authentication.

- **Method:** `vault` CLI
- **Command Pattern:**

```bash
# AppRole login (role_id and secret_id from secure source)
vault write auth/approle/login \
  role_id="$VAULT_ROLE_ID" \
  secret_id="$VAULT_SECRET_ID"
```

### 8. Revoke Current Token (Cleanup)

Revoke your token when finished with a session.

- **Method:** `vault` CLI
- **Command Pattern:**

```bash
# Revoke current token
vault token revoke -self
```

### 9. Policy Management

Manage Vault policies that control access to secrets.

- **Method:** `vault` CLI
- **Purpose:** Define and audit what tokens can access which paths

**Command Pattern:**
```bash
# List all policies
vault policy list

# Read a specific policy
vault policy read my-policy

# Write a policy from HCL file
vault policy write my-policy policy.hcl

# Example policy HCL:
cat > app-policy.hcl <<EOF
path "kvProd_v2/data/myapp/*" {
  capabilities = ["read", "list"]
}
path "kvProd_v2/metadata/myapp/*" {
  capabilities = ["list"]
}
EOF
vault policy write app-readonly app-policy.hcl

# Delete a policy
vault policy delete old-policy

# Check what paths a policy grants
vault policy read my-policy
```

### 10. Dynamic Secrets (Database Credentials)

Generate short-lived database credentials on-demand.

- **Method:** `vault` CLI
- **Purpose:** Avoid static database passwords, get time-limited creds

**Command Pattern:**
```bash
# Read dynamic database credentials (creates new user)
vault read database/creds/my-role

# Response includes:
# - username: v-userpass-my-role-xyz123
# - password: A1B2C3D4E5F6
# - lease_duration: 3600 (1 hour)

# Use the credentials immediately
DB_USER=$(vault read -field=username database/creds/my-role)
DB_PASS=$(vault read -field=password database/creds/my-role)
psql -h postgres.bjzy.me -U $DB_USER -d mydb

# Renew the lease if needed
vault lease renew database/creds/my-role/abc123

# Revoke the lease manually (user is deleted)
vault lease revoke database/creds/my-role/abc123

# List active dynamic secret leases
vault list sys/leases/lookup/database/creds/my-role
```

### 11. Transit Encryption Engine

Use Vault for encryption-as-a-service without exposing keys.

- **Method:** `vault` CLI
- **Purpose:** Encrypt/decrypt data using Vault-managed keys

**Command Pattern:**
```bash
# Enable transit engine (if not already enabled)
vault secrets enable transit

# Create an encryption key
vault write -f transit/keys/my-app-key

# Encrypt plaintext (base64 encoded)
echo -n "secret data" | base64 | vault write transit/encrypt/my-app-key plaintext=- | jq -r '.data.ciphertext'
# Returns: vault:v1:abc123def456...

# Decrypt ciphertext
vault write transit/decrypt/my-app-key ciphertext="vault:v1:abc123def456..." | jq -r '.data.plaintext' | base64 -d

# Rotate the encryption key
vault write -f transit/keys/my-app-key/rotate

# Rewrap data with new key version (without exposing plaintext)
vault write transit/rewrap/my-app-key ciphertext="vault:v1:oldversion..."

# List encryption keys
vault list transit/keys

# Delete an encryption key (after min_decryption_version)
vault delete transit/keys/old-key
```

## Troubleshooting

### Token Expired or Invalid

```bash
# Check token status
vault token lookup

# If expired, re-authenticate
vault login -method=userpass username=$USER
```

### Permission Denied

```bash
# Check what policies your token has
vault token lookup -format=json | jq '.data.policies'

# Verify you have access to the path
vault kv get kvProd_v2/path/to/secret
# Error will indicate if it's a permissions issue
```

### Certificate Verification Issues

If you encounter SSL certificate verification errors:

```bash
# Temporary workaround (not recommended for production)
export VAULT_SKIP_VERIFY=true

# Proper fix: ensure CA certificate is trusted
export VAULT_CACERT=/path/to/ca.crt
```

**Note:** Certificate issues should be resolved by trusting the homelab CA, not by skipping verification.

### Cannot Connect to Vault

```bash
# Verify VAULT_ADDR is set correctly
echo $VAULT_ADDR
# Should be: https://vault.bjzy.me:8200

# Test connectivity
curl -ksS https://vault.bjzy.me:8200/v1/sys/health | jq
```

### Secret Not Found

```bash
# Verify the path exists
vault kv list kvProd_v2/

# Check if you're using the correct KV version syntax
# KV v2 uses: vault kv get kvProd_v2/path
# KV v1 uses: vault read secret/path
```
