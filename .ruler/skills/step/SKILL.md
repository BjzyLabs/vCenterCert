---
name: step
description: Manages Smallstep Step CA certificate authority for certificate issuance, renewal, revocation, ACME protocol operations, PKI management, and automated certificate lifecycle. Use when user mentions Step CA, Smallstep, certificates, ACME, PKI, TLS, or certificate management.
---

# Step CA Certificate Authority

## Instructions

Use this skill to interact with Step CA for certificate authority operations, PKI management, and ACME protocol services. Step CA provides a comprehensive certificate authority solution used throughout the Home Lab infrastructure for SSL/TLS certificate management.

### Configuration

The preferred method of interaction is the `step` CLI tool (installed on the host). Step CA runs as the primary certificate authority for Bjzy Labs, providing certificates for internal services and external-facing applications.

**Smart Authentication:**
Before certificate operations, verify Step CA is reachable and properly configured.

```bash
# Check if Step CA is accessible
if ! step ca health > /dev/null 2>&1; then
  echo "Step CA not accessible"
  export STEP_CA_URL=https://ca.bjzy.me
  # Verify CA root certificate is trusted
  step ca root /tmp/root.crt 2>&1
fi

# Verify CA connectivity
step ca health || echo "CA health check failed"
```

#### Bjzy Labs defaults

- **Step CA Instance**: ca.bjzy.me
- **Common use cases**:
  - **Certificate Issuance**: Generate SSL/TLS certificates for services
  - **ACME Protocol**: Provide ACME v2 endpoint for certbot integration
  - **Root CA Management**: Manage the internal PKI hierarchy
  - **Certificate Renewal**: Automate certificate lifecycle management
- **Integration Points**:
  - **AWX**: Uses Step CA for certificate renewal via certbot
  - **Harbor Registry**: SSL certificates signed by Step CA
  - **Docker Swarm**: Service certificates for internal communication
  - **Vault**: Client certificate authentication

### Environment and Guardrails (Bjzy Labs)

- **CA Access**:
  - Step CA runs on dedicated CA infrastructure
  - CLI access available from management workstations
  - HTTP API available at `https://ca.bjzy.me`
- **Security Rules**:
  - Always verify CA health before certificate operations
  - Use appropriate certificate templates for different service types
  - Be cautious with root CA operations (affect entire PKI)
  - Never provision certificates without proper verification
- **CLI Availability**:
  - The `step` CLI is installed on management workstations
  - Environment variable `STEP_CA_URL` defaults to `https://ca.bjzy.me`
  - Verify access with `step ca health`

### Standard Operating Procedure (SOP)

When asked to "Manage certificates," "Issue certificates," or "Check CA status":

1. **Verify CA Health:** Run `step ca health` to check CA connectivity
2. **Identify Operation:** Determine if you need certificate issuance, renewal, or CA management
3. **Execute Command:** Use appropriate step command with proper parameters
4. **Verify Results:** Check certificate validity and CA response
5. **Document Changes:** Log any certificate issuances or CA modifications

## Examples

### 1. Check CA Health and Status

Verify the Step CA instance is operational and accessible.

- **Method:** `step` CLI
- **Command Pattern:**

```bash
# Check CA health
step ca health

# Check CA configuration
step ca config

# Verify CA connectivity
step ca version

# Check CA root certificate
step ca root /tmp/root.crt
```

### 2. Issue Service Certificates

Generate SSL/TLS certificates for internal services.

- **Method:** `step` CLI
- **Command Pattern:**

```bash
# Issue certificate for a service
step ca certificate <service_name> <cert_file> <key_file> \
  --ca-url https://ca.bjzy.me \
  --root $(step path)/certs/root_ca.crt

# Issue certificate with SANs
step ca certificate <service_name> <cert_file> <key_file> \
  --san <service_name>.local \
  --san <service_name>.bjzy.me \
  --ca-url https://ca.bjzy.me

# Issue certificate with specific template
step ca certificate <service_name> <cert_file> <key_file> \
  --template <template_name> \
  --ca-url https://ca.bjzy.me
```

### 3. Certificate Renewal Operations

Manage certificate lifecycle and renewal processes.

- **Method:** `step` CLI + certbot integration
- **Command Pattern:**

```bash
# Renew certificate via certbot with Step CA
REQUESTS_CA_BUNDLE=$(step path)/certs/root_ca.crt \
  certbot renew --cert-name <domain> \
  --server https://ca.bjzy.me/acme/acme/directory

# Force certificate renewal
REQUESTS_CA_BUNDLE=$(step path)/certs/root_ca.crt \
  certbot renew --cert-name <domain> --force-renewal

# Check certificate expiration
step ca certificate <cert_file> --expires-in
```

### 4. Root CA Management

Manage the root certificate authority and trust chain.

- **Method:** `step` CLI
- **Command Pattern:**

```bash
# Get root certificate
step ca root /tmp/root.crt

# Get intermediate certificates
step ca intermediate /tmp/intermediate.crt

# Verify certificate chain
step certificate verify <cert_file> --roots /tmp/root.crt

# Export CA certificate bundle
step ca bundle --ca-url https://ca.bjzy.me
```

### 5. ACME Protocol Operations

Manage ACME endpoint for automated certificate management.

- **Method:** `step` CLI + HTTP API
- **Command Pattern:**

```bash
# Check ACME directory
curl https://ca.bjzy.me/acme/acme/directory

# Create ACME account
step ca acme register <email> --ca-url https://ca.bjzy.me

# List ACME accounts
step ca acme list --ca-url https://ca.bjzy.me

# Test ACME challenge
step ca acme challenge <token> --ca-url https://ca.bjzy.me
```

### 6. Certificate Inspection and Validation

Verify and inspect issued certificates.

- **Method:** `step` CLI
- **Command Pattern:**

```bash
# Inspect certificate details
step certificate inspect <cert_file>

# Verify certificate against CA
step certificate verify <cert_file> --ca-url https://ca.bjzy.me

# Check certificate expiration
step certificate inspect <cert_file> --expires-in

# Extract certificate information
step certificate inspect <cert_file> --format json
```

### 7. Kubernetes Integration

Manage certificates for Kubernetes workloads.

- **Method:** `step` CLI + kubectl
- **Command Pattern:**

```bash
# Create certificate for Kubernetes service
step ca certificate <service_name> <cert_file> <key_file> \
  --san <service_name>.default.svc.cluster.local \
  --ca-url https://ca.bjzy.me

# Create Kubernetes secret with certificate
kubectl create secret tls <secret_name> \
  --cert=<cert_file> --key=<key_file> \
  --namespace=<namespace>

# Update existing Kubernetes secret
kubectl create secret tls <secret_name> \
  --cert=<cert_file> --key=<key_file> \
  --namespace=<namespace> --dry-run=client -o yaml | \
  kubectl apply -f -
```

### 8. Harbor Registry Integration

Manage SSL certificates for Harbor container registry.

- **Method:** `step` CLI + Docker
- **Command Pattern:**

```bash
# Issue certificate for Harbor
step ca certificate harbor.bjzy.me harbor.crt harbor.key \
  --san harbor.bjzy.me \
  --san registry.bjzy.me \
  --ca-url https://ca.bjzy.me

# Configure Harbor with certificate
sudo cp harbor.crt /data/cert/
sudo cp harbor.key /data/cert/

# Configure Docker to trust Harbor CA
sudo mkdir -p /etc/docker/certs.d/harbor.bjzy.me
sudo cp $(step path)/certs/root_ca.crt /etc/docker/certs.d/harbor.bjzy.me/ca.crt
```

### 9. Certificate Revocation

Revoke compromised or expired certificates.

- **Method:** `step` CLI
- **Command Pattern:**

```bash
# Revoke certificate
step ca revoke <cert_file> --ca-url https://ca.bjzy.me

# Revoke certificate by serial
step ca revoke --serial <serial_number> --ca-url https://ca.bjzy.me

# Check revocation status
step ca revoke --reason <reason> <cert_file> --ca-url https://ca.bjzy.me
```

### 10. Backup and Recovery

Backup CA certificates and configuration.

- **Method:** `step` CLI + filesystem operations
- **Command Pattern:**

```bash
# Backup CA configuration
sudo cp -r ~/.step/config /backup/step-config/

# Export CA certificates
step ca root /backup/root.crt
step ca intermediate /backup/intermediate.crt

# Backup issued certificates
tar -czf /backup/certificates-$(date +%Y%m%d).tar.gz \
  ~/.step/certs/
```

### 11. Certificate Authority Key Rotation

Rotate CA keys and certificates for security best practices.

- **Method:** `step` CLI
- **Purpose:** Regular key rotation to maintain PKI security posture

**Command Pattern:**
```bash
# Check current CA certificate expiration
step ca root /tmp/root.crt
openssl x509 -in /tmp/root.crt -noout -dates

# ⚠️ CRITICAL WARNING: CA Key Rotation is a complex production operation
# This requires root CA access, coordinated client updates, and planned downtime.
# The following is a HIGH-LEVEL overview. Always follow Step CA official documentation
# and your organization's PKI runbook for production CA rotation.

# Step 1: Backup existing CA configuration
cp -r $(step path)/certs $(step path)/certs.backup-$(date +%Y%m%d)
cp -r $(step path)/secrets $(step path)/secrets.backup-$(date +%Y%m%d)

# Step 2: Generate new intermediate certificate (signed by existing root)
# This does NOT use 'step ca init' - that command bootstraps a NEW CA
# Instead, use 'step certificate create' to generate a new intermediate
step certificate create "Bjzy Labs Intermediate CA v2" \
  intermediate_v2.crt intermediate_v2.key \
  --profile intermediate-ca \
  --ca $(step path)/certs/root_ca.crt \
  --ca-key $(step path)/secrets/root_ca_key

# Step 3: Update CA configuration to use new intermediate
# Edit $(step path)/config/ca.json to reference new intermediate cert/key
# This typically requires Step CA restart

# Step 4: Verify new certificate chain
step certificate verify intermediate_v2.crt \
  --roots $(step path)/certs/root_ca.crt

# Step 5: Update CA bundle on all clients
step ca root /etc/ssl/certs/bjzy-ca-bundle.crt

# Step 6: Verify CA is operational with new intermediate
step ca health

# ⚠️ PRODUCTION IMPACT:
# - All previously issued certificates remain valid (same root CA)
# - New certificate issuances will use new intermediate
# - Client trust is maintained (root CA unchanged)
# - For ROOT CA rotation (much rarer), all clients must update trust anchors
# - Coordinate with all service owners before executing
```

## Troubleshooting

### CA Connectivity Issues

```bash
# Check CA health
step ca health

# Verify CA URL configuration
echo $STEP_CA_URL

# Test HTTP API connectivity
curl -k https://ca.bjzy.me/health

# Check root certificate
step ca root /tmp/root.crt
```

### Certificate Issuance Failures

```bash
# Check CA configuration
step ca config

# Verify certificate template
step ca template list --ca-url https://ca.bjzy.me

# Check certificate signing request
step certificate inspect <csr_file>

# Debug certificate issuance
step ca certificate <name> <cert> <key> --ca-url https://ca.bjzy.me --debug
```

### ACME Protocol Issues

```bash
# Test ACME endpoint
curl https://ca.bjzy.me/acme/acme/directory

# Check ACME account
step ca acme list --ca-url https://ca.bjzy.me

# Verify certbot configuration
sudo certbot certificates

# Check certbot logs
sudo tail -50 /var/log/letsencrypt/letsencrypt.log
```

### Certificate Validation Problems

```bash
# Verify certificate chain
step certificate verify <cert_file> --roots $(step path)/certs/root_ca.crt

# Check certificate expiration
step certificate inspect <cert_file> --expires-in

# Validate certificate format
openssl x509 -in <cert_file> -text -noout

# Check certificate SANs
step certificate inspect <cert_file> | grep -E "(DNS|IP)"
```

### Kubernetes Integration Issues

```bash
# Verify kubectl access
kubectl get secrets

# Check certificate in secret
kubectl get secret <secret_name> -o yaml

# Validate certificate format in secret
kubectl get secret <secret_name> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text
```

## Common Service Patterns

### AWX Certificate Renewal

AWX uses automated certificate renewal via certbot and Step CA:

```bash
# Check AWX certificate status
kubectl get secret awx-tls-secret -n awx -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -enddate

# Manual renewal if needed
sudo REQUESTS_CA_BUNDLE=$(step path)/certs/root_ca.crt \
  certbot renew --cert-name awx.bjzy.me \
  --server https://ca.bjzy.me/acme/acme/directory
```

### Docker Swarm Service Certificates

Services in Docker Swarm use Step CA for mutual TLS:

```bash
# Issue certificate for Swarm service
step ca certificate <service_name> <cert> <key> \
  --san <service_name>.local \
  --san tasks.<service_name> \
  --ca-url https://ca.bjzy.me

# Create Docker secret
docker secret create <service_name>.tls <cert_file>
docker secret create <service_name>.key <key_file>
```

### Database SSL Configuration

Configure PostgreSQL with Step CA certificates:

```bash
# Issue certificate for PostgreSQL
step ca certificate postgresql postgres.crt postgres.key \
  --san postgresql.local \
  --san postgresql.bjzy.me \
  --ca-url https://ca.bjzy.me

# Configure PostgreSQL SSL
sudo cp postgres.crt /var/lib/postgresql/data/server.crt
sudo cp postgres.key /var/lib/postgresql/data/server.key
```

### Monitoring Certificate Expiration

Monitor certificate expiration across services:

```bash
# Check all certificates in a directory
for cert in /path/to/certs/*.crt; do
  echo "=== $cert ==="
  step certificate inspect "$cert" --expires-in
done

# Check Kubernetes secrets
kubectl get secrets --all-namespaces -o json | \
  jq -r '.items[] | select(.type=="kubernetes.io/tls") | "\(.metadata.namespace):\(.metadata.name)"'
```
