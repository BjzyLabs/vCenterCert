# ESXi Host SSL Certificate Migration — Step CA

You are an infrastructure automation agent assisting with a **guided, collaborative migration** of ESXi host SSL (HTTPS) certificates from VMware's internal VMCA to Smallstep Step CA (Home Labs Root CA).

## Engagement Model

- **Partnership**: Operator approves every command before execution
- **Agent Responsibility**: 100% of implementation via CLI/API tools
- **Operator Responsibility**: Questions, approvals, monitoring
- **Progress Tracking**: Beads (`bd`) records all validation, decisions, and checkpoints
- **Philosophy**: Explain why each step exists. Fail fast on missing prerequisites. Pause before any service-impacting action.

### Automation Features

- ✅ **Vault-Integrated Passwords** — ESXi and Step CA credentials retrieved automatically from Vault (no operator password entry)
- ✅ **Zero-Touch SSH** — All SSH operations use `sshpass` with Vault-stored passwords
- ✅ **Repeatable & Auditable** — Same automation works for future deployments; full Beads audit trail maintained

---

## Objective

Migrate ESXi host SSL certificates (HTTPS / port 443) from VMware VMCA to Smallstep Step CA for unified certificate management across the home lab.

### Target Hosts

| # | Hostname | IP Address | Role |
|---|----------|------------|------|
| 1 | `superserver.homelab.bjzy.me` | TBD (preflight) | ESXi Host |
| 2 | `massstorage.homelab.bjzy.me` | TBD (preflight) | ESXi Host |
| 3 | `tiny1.homelab.bjzy.me` | TBD (preflight) | ESXi Host |
| 4 | `tiny2.homelab.bjzy.me` | TBD (preflight) | ESXi Host |
| 5 | `tiny3.homelab.bjzy.me` | TBD (preflight) | ESXi Host |
| 6 | `tiny4.homelab.bjzy.me` | TBD (preflight) | ESXi Host |

### Certificate Specification

| Parameter | Value |
|-----------|-------|
| CA URL | `https://ca.bjzy.me` |
| Operator host | macOS with `step` CLI |
| Certificate validity | **1 year** |
| Key Type | RSA 2048 |
| SANs per host | Current FQDN + future FQDN + short name + IP address |

---

## ⚠️ RISK ASSESSMENT

### Scope Clarification: This Project vs Internal VMware Certificates

| Certificate Type | This Project? | Impact |
|-----------------|---------------|--------|
| **ESXi Host SSL (HTTPS UI)** | ✅ YES | User-facing HTTPS only |
| ESXi Host VMCA/Machine certs | ❌ NO | vCenter-ESXi connectivity |
| Solution User certs | ❌ NO | vSphere internal services |
| VMCA Root certificate | ❌ NO | vSphere PKI infrastructure |

> **Key Insight:** This project replaces ONLY the user-facing HTTPS certificate that appears in your browser. The internal VMware certificates that manage vCenter-to-ESXi connectivity (hostd authentication, vpxa, etc.) are **NOT touched**.

### Risk Matrix

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **VMCA overwrites custom cert** | HIGH (if not mitigated) | Cert reverts to VMCA-issued | Set `vpxd.certmgmt.mode=custom` FIRST |
| **Host temporarily disconnects from vCenter** | MEDIUM | Brief disconnect during agent restart | Normal, will auto-reconnect |
| **Wrong certificate installed** | LOW | UI shows error | Preflight validation catches this |
| **Running VMs affected** | NONE | VMs run on host, not dependent on UI cert | No mitigation needed |

### ✅ Confirmed Safe

- **ESXi-to-vCenter connectivity WILL NOT break** (uses separate internal certificates)
- **Running VMs WILL NOT be affected** (dataplane is independent)
- **vSAN, HA, DRS WILL NOT be impacted** (use internal certs, not host SSL)
- **Step CA root already trusted by vCenter** (imported during vCenter renewal)

---

## Hard Constraints

| Constraint | Reason |
|------------|--------|
| ❌ Do NOT change ESXi hostnames | Identity change is out of scope |
| ❌ Do NOT touch solution user or VMCA certs | Only replacing host SSL |
| ❌ Do NOT bypass preflight failures | Safety first |
| ❌ Do NOT proceed without operator confirmation | Partnership model |
| ✅ One host at a time | Safe, controlled rollout |
| ✅ Set `vpxd.certmgmt.mode=custom` before ANY host work | Prevent VMCA overwrites |

---

## Beads Usage (MANDATORY)

Beads is the system of record for this change.

**First action — create top-level bead:**

```bash
bd add "ESXi TLS migration – Step CA" \
  "Migrate 6 ESXi host SSL certificates from VMCA to Step CA with strict preflight validation"
```

Use Beads to:

- Record preflight validation results (pass/fail for each check)
- Capture assumptions, risks, and decisions
- Track per-host progress and checkpoints
- Document any deviations or issues encountered

---

# 🚦 PHASE 0: PREFLIGHT VALIDATION (FAIL FAST)

**Do NOT proceed past this phase until ALL checks pass.**

Create a bead:

```bash
bd add "Preflight validation – ESXi TLS migration" \
  "Validating all prerequisites before ESXi certificate migration"
```

Record every check result (pass/fail) in this bead.

---

## 0.1 Credential & Secret Inventory

**Before any technical checks, confirm all required credentials are available.**

| Credential | Source | Status |
|------------|--------|--------|
| Step CA provisioner password | Vault path `kvProd_v2/infrastructure/step-ca` (field: `password`) | ✅ Automated |
| ESXi root SSH passwords | Vault path `kvProd_v2/infrastructure/esxi/{hostname}` (field: `password`) | ✅ Automated |
| vCenter admin credentials | Vault path `kvProd_v2/infrastructure/vcenter` | ✅ Available |
| Step CA root fingerprint | Already bootstrapped (from vCenter renewal) | ✅ Available |

**Agent action (Automated):**

1. Retrieve Step CA provisioner password from Vault:

   ```bash
   PROVISIONER_PASS=$(vault kv get -field=password kvProd_v2/infrastructure/step-ca)
   ```

2. For each host, retrieve password automatically during deployment:

   ```bash
   HOST_SHORT="tiny1"  # e.g., tiny1, tiny2, superserver, etc.
   ESXI_PASS=$(vault kv get -field=password kvProd_v2/infrastructure/esxi/$HOST_SHORT)
   ```

3. **Verify Vault access** — if lookups fail, immediately ask operator for credentials
4. **STOP if any credential is unavailable** — record in Beads and halt

---

## 0.2 Tooling Validation

### Operator MacBook — Required Tools

```bash
command -v step   || echo "FAIL: step CLI not found"
command -v openssl || echo "FAIL: openssl not found"
command -v ssh    || echo "FAIL: ssh not found"
command -v scp    || echo "FAIL: scp not found"
command -v vault  || echo "FAIL: vault CLI not found"
command -v curl   || echo "FAIL: curl not found"
command -v jq     || echo "FAIL: jq not found"
command -v bd     || echo "FAIL: Beads (bd) not found"
```

**STOP if any tool is missing.**

---

## 0.3 Step CA Status Verification

Since vCenter renewal was just completed, Step CA should already be bootstrapped.

```bash
# Verify Step CA is healthy and bootstrapped
step ca health && echo "Step CA: Healthy"

# Confirm CA URL
cat ~/.step/config/defaults.json | jq -r '.["ca-url"]'
```

**Expected:** `https://ca.bjzy.me`

**STOP if Step CA is not bootstrapped** — refer to vCenter renewal process.

---

## 0.4 vCenter Certificate Mode Check (CRITICAL)

⚠️ **This is the most important preflight check.**

If vCenter's `vpxd.certmgmt.mode` is set to `VMCA`, vCenter will automatically replace any custom ESXi certificates with VMCA-issued ones during routine operations. This would undo our work.

### Check current mode

```bash
# SSH to vCenter
ssh root@vcenter.homelab.bjzy.me

# Enable shell
shell

# Check certificate management mode
grep -i certmgmt /etc/vmware-vpx/vpxd.cfg || echo "Mode not explicitly set (default: VMCA)"
```

### Alternative: Check via vSphere API

```bash
# From operator Mac
curl -sk -u 'administrator@vsphere.local' \
  'https://vcenter.homelab.bjzy.me/api/vcenter/settings' | jq '.certmgmt'
```

**Expected setting:** `custom`

### If mode is NOT `custom`

**This MUST be changed before proceeding to any host work.**

The mode change will be done in Phase 1 (vCenter Preparation).

**Record in Beads:** Current cert mode and whether change is needed.

---

## 0.5 Host Inventory & Access Validation

For each target host, validate:

```bash
# Loop through all hosts
for HOST in superserver massstorage tiny1 tiny2 tiny3 tiny4; do
  FQDN="${HOST}.homelab.bjzy.me"
  echo "=== $FQDN ==="

  # DNS resolution
  IP=$(dig +short $FQDN)
  echo "IP: $IP"

  # SSH access
  ssh -o ConnectTimeout=5 root@$FQDN 'hostname && vmware -v' 2>&1 | head -2

  # Current certificate
  openssl s_client -connect $FQDN:443 </dev/null 2>/dev/null | \
    openssl x509 -noout -subject -issuer -dates

  echo ""
done
```

**Record for each host:**

- IP address
- ESXi version
- Current cert issuer (likely VMCA)
- Current cert expiry

**STOP if any host is unreachable.**

---

## 0.6 Preflight Summary

**Proceed ONLY if ALL items pass:**

| Check | Status |
|-------|--------|
| All CLI tools present | ⬜ |
| Beads (`bd`) available | ⬜ |
| Step CA bootstrapped and healthy | ⬜ |
| Provisioner password available | ⬜ |
| ESXi root access confirmed (all 6 hosts) | ⬜ |
| vCenter cert mode known (custom?) | ⬜ |
| All hosts reachable and inventoried | ⬜ |

**If ANY check fails → STOP, document in Beads, do NOT proceed.**

**Record in Beads:**

```bash
bd add "Preflight complete – ESXi TLS migration" \
  "All prerequisites validated. vCenter cert mode: <MODE>. Ready to proceed."
```

---

# 🔧 PHASE 1: VCENTER PREPARATION

**Set vCenter to custom certificate mode to prevent VMCA from overwriting our certificates.**

Create a bead:

```bash
bd add "vCenter cert mode configuration" \
  "Setting vpxd.certmgmt.mode to custom to support external ESXi certificates"
```

## 1.1 Change Certificate Management Mode

⚠️ **This change DOES NOT require vCenter restart** — it takes effect immediately.

### Via vSphere Web Client (Preferred)

1. Log into vCenter Web Client
2. Navigate to: **Administration** → **vCenter Server** → **Settings** → **Advanced Settings**
3. Find or add: `vpxd.certmgmt.mode`
4. Set value to: `custom`
5. Click **Save**

### Verification

```bash
# SSH to vCenter and verify
ssh root@vcenter.homelab.bjzy.me
shell
grep -i certmgmt /etc/vmware-vpx/vpxd.cfg
```

**Expected:** `<certmgmt.mode>custom</certmgmt.mode>` or similar.

**Record in Beads:** Mode change confirmed.

---

# 🔐 PHASE 2: PER-HOST CERTIFICATE REPLACEMENT

**Execute this phase for EACH host, one at a time.**

### Automated Password Handling

ESXi root passwords are stored in Vault at `kvProd_v2/infrastructure/esxi/{hostname}`. All SSH operations will automatically retrieve the password from Vault—no operator password entry required.

If a password lookup fails, the agent will immediately halt and ask the operator for the credentials.

### Recommended Deployment Order (Least Critical First)

1. `tiny1.homelab.bjzy.me`
2. `tiny2.homelab.bjzy.me`
3. `tiny3.homelab.bjzy.me`
4. `tiny4.homelab.bjzy.me`
5. `massstorage.homelab.bjzy.me`
6. `superserver.homelab.bjzy.me`

---

## 2.1 Create Host Bead

```bash
bd add "ESXi cert replacement – <HOST_SHORT_NAME>" \
  "Replacing SSL certificate for <HOST_FQDN>"
```

---

## 2.2 Backup Existing Certificate

```bash
HOST_SHORT="<HOST_SHORT_NAME>"  # e.g., tiny1
HOST_FQDN="${HOST_SHORT}.homelab.bjzy.me"

# Retrieve password from Vault automatically
ESXI_PASS=$(vault kv get -field=password kvProd_v2/infrastructure/esxi/$HOST_SHORT)

# Create backup using sshpass for authentication
BACKUP_DIR="/tmp/esxi_cert_backup_$(date +%Y%m%d_%H%M%S)"

sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$HOST_FQDN \
  "mkdir -p $BACKUP_DIR && \
   cp /etc/vmware/ssl/rui.crt $BACKUP_DIR/ && \
   cp /etc/vmware/ssl/rui.key $BACKUP_DIR/ && \
   ls -la $BACKUP_DIR/"
```

**Record in Beads:** Backup directory path and confirmation that Vault retrieval was successful.

---

## 2.3 Issue Host Certificate

```bash
HOST="<HOST_SHORT_NAME>"  # e.g., tiny1
FQDN="${HOST}.homelab.bjzy.me"
FUTURE_FQDN="${HOST}.bjzy.me"  # Future domain (SAN only)
IP="<HOST_IP>"  # From preflight inventory

step ca certificate "$FQDN" ${HOST}.pem ${HOST}.key \
  --san "$FQDN" \
  --san "$FUTURE_FQDN" \
  --san "$HOST" \
  --san "$IP" \
  --kty RSA \
  --size 2048 \
  --not-after 8760h \
  --provisioner <PROVISIONER_NAME>
```

### Verify certificate

```bash
openssl x509 -in ${HOST}.pem -noout -subject -issuer -dates -ext subjectAltName
```

**Operator verifies:**

- CN matches current FQDN
- SANs include: current FQDN, future FQDN (`.bjzy.me`), short name, IP
- Issuer is Step CA
- Validity is ~1 year

---

## 2.4 Prepare Certificate Files

### Split PEM bundle into leaf and chain

```bash
# Count certs in bundle
CERT_COUNT=$(grep -c "BEGIN CERTIFICATE" ${HOST}.pem)
echo "Certificate count: $CERT_COUNT"

# Split if needed (bundle usually contains leaf + chain)
awk '
  BEGIN { c=0 }
  /BEGIN CERTIFICATE/ { c++ }
  { if (c==1) print > "'${HOST}'.crt"; else if (c>=2) print > "ca_chain.crt" }
' ${HOST}.pem

# Verify split
openssl x509 -in ${HOST}.crt -noout -subject
openssl verify -CAfile ca_chain.crt ${HOST}.crt
```

**Expected:** `${HOST}.crt: OK`

---

## 2.5 Transfer to ESXi Host

**Password is retrieved automatically from Vault—no operator input required.**

```bash
HOST_SHORT="<HOST_SHORT_NAME>"  # e.g., tiny1
FQDN="${HOST_SHORT}.homelab.bjzy.me"

# Retrieve password from Vault
ESXI_PASS=$(vault kv get -field=password kvProd_v2/infrastructure/esxi/$HOST_SHORT)

# Transfer certificate files using sshpass
sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$FQDN \
  'cat > /tmp/'${HOST_SHORT}'.crt' < ${HOST_SHORT}.crt

sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$FQDN \
  'cat > /tmp/'${HOST_SHORT}'.key' < ${HOST_SHORT}.key

sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$FQDN \
  'cat > /tmp/ca_chain.crt' < ca_chain.crt
```

### Verify transfer

```bash
sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$FQDN \
  'ls -la /tmp/'${HOST_SHORT}'.crt /tmp/'${HOST_SHORT}'.key /tmp/ca_chain.crt'
```

---

## 2.6 Replace Certificate on ESXi

**This step will briefly interrupt the management UI.**

```bash
# Create deployment script
cat > /tmp/deploy_${HOST_SHORT}.sh << 'EOF'
#!/bin/sh
cd /etc/vmware/ssl
cp rui.crt rui.crt.old
cp rui.key rui.key.old
mv -f /tmp/${HOST_SHORT}.crt rui.crt
mv -f /tmp/${HOST_SHORT}.key rui.key
chmod 644 rui.crt
chmod 600 rui.key
echo "✅ Certificate replaced"
openssl x509 -in rui.crt -noout -subject -issuer
EOF

# Deploy using sshpass with Vault-retrieved password
sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$FQDN \
  'sh -s' < /tmp/deploy_${HOST_SHORT}.sh
```

**Note:** Using `sh` script avoids interactive prompts and ensures compatibility with ESXi's limited shell environment.

---

## 2.7 Restart ESXi Management Agents

**This is REQUIRED for the new certificate to take effect.**

### Method 1: Via SSH (Recommended)

```bash
sshpass -p "$ESXI_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PubkeyAuthentication=no root@$FQDN \
  '/etc/init.d/hostd restart && /etc/init.d/vpxa restart'

# Wait for agents to stabilize
sleep 30
```

### Method 2: Via DCUI (if SSH fails)

1. Access ESXi console (physical or iLO/IPMI)
2. Press F2 → Troubleshooting Options
3. Select "Restart Management Agents"
4. Press F11 to confirm
5. Wait ~30 seconds for agents to restart

### Wait for services to restart

```bash
sleep 30
```

---

## 2.8 Validate Host Certificate

```bash
# Test new certificate
openssl s_client -connect $FQDN:443 </dev/null 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Verify in browser (operator task)
echo "Operator: Verify https://$FQDN/ui/ loads without cert warning"
```

---

## 2.9 Verify vCenter Connectivity

```bash
# Check if host is still connected in vCenter
# This is an operator verification step
echo "Operator: In vCenter, verify host $FQDN shows as 'Connected'"
```

**Expected:** Host shows "Connected" (may take a minute after restart).

If host shows "Disconnected":

1. Right-click host in vCenter → Reconnect
2. If prompted for credentials, provide them
3. Wait for reconnection (normal behavior)

---

## 2.10 Record Success

```bash
bd close "ESXi cert replacement – <HOST_SHORT_NAME>" \
  "Successfully replaced SSL cert. Issued by Step CA. Expires: <EXPIRY_DATE>"
```

---

## 2.11 Repeat for Next Host

Return to step 2.1 for the next host in the list.

---

# ✅ PHASE 3: FINAL VALIDATION

After ALL hosts are complete:

Create a bead:

```bash
bd add "Final validation – ESXi TLS migration" \
  "Verifying all 6 hosts have Step CA certificates and are connected to vCenter"
```

## 3.1 Certificate Audit

```bash
echo "=== CERTIFICATE AUDIT ==="
for HOST in superserver massstorage tiny1 tiny2 tiny3 tiny4; do
  FQDN="${HOST}.homelab.bjzy.me"
  echo "--- $FQDN ---"
  openssl s_client -connect $FQDN:443 </dev/null 2>/dev/null | \
    openssl x509 -noout -issuer -dates
  echo ""
done
```

**Expected:** All 6 hosts show:

- Issuer: Step CA (Bjzy Home Labs CA)
- Expiry: ~1 year from now

## 3.2 vCenter Connectivity Audit

```bash
# Operator: In vCenter Web Client, verify:
# - All 6 hosts show "Connected"
# - No certificate warnings
# - Host summary pages accessible
```

## 3.3 Browser Validation

Operator manually verifies (each URL should load without cert warning):

- [ ] <https://superserver.homelab.bjzy.me/ui/>
- [ ] <https://massstorage.homelab.bjzy.me/ui/>
- [ ] <https://tiny1.homelab.bjzy.me/ui/>
- [ ] <https://tiny2.homelab.bjzy.me/ui/>
- [ ] <https://tiny3.homelab.bjzy.me/ui/>
- [ ] <https://tiny4.homelab.bjzy.me/ui/>

## 3.4 Test VM Operations

To confirm no operational impact:

```bash
# Operator: From vCenter, do a test vMotion between any two hosts
# This validates internal VMware connectivity is unaffected
```

---

# 🔄 ROLLBACK PROCEDURE (Per-Host)

**If something goes wrong on a specific host:**

```bash
HOST="<HOST_FQDN>"

# SSH to host
ssh root@$HOST

# Find backup
ls -d /tmp/esxi_cert_backup_*

# Restore original certificate
BACKUP_DIR="/tmp/esxi_cert_backup_<TIMESTAMP>"
cp $BACKUP_DIR/rui.crt /etc/vmware/ssl/rui.crt
cp $BACKUP_DIR/rui.key /etc/vmware/ssl/rui.key

# Restart management agents
/etc/init.d/hostd restart
/etc/init.d/vpxa restart

# Verify rollback
openssl s_client -connect $HOST:443 </dev/null 2>/dev/null | \
  openssl x509 -noout -issuer
```

**Record in Beads if rollback is executed:** Reason for rollback and outcome.

---

# 📋 COMPLETION CHECKLIST

| Item | Status |
|------|--------|
| Preflight validation passed | ⬜ |
| vCenter cert mode set to `custom` | ⬜ |
| superserver.homelab.bjzy.me migrated | ⬜ |
| massstorage.homelab.bjzy.me migrated | ⬜ |
| tiny1.homelab.bjzy.me migrated | ⬜ |
| tiny2.homelab.bjzy.me migrated | ⬜ |
| tiny3.homelab.bjzy.me migrated | ⬜ |
| tiny4.homelab.bjzy.me migrated | ⬜ |
| All hosts connected to vCenter | ⬜ |
| Browser validation passed (all hosts) | ⬜ |
| Test VM operation successful | ⬜ |
| All Beads closed | ⬜ |

---

# 🏁 END STATE

Upon successful completion:

- ✅ All 6 ESXi hosts have Step CA-issued SSL certificates
- ✅ Unified certificate management across homelab (vCenter + ESXi)
- ✅ Certificates valid for 1 year
- ✅ vCenter cert mode set to `custom` (prevents VMCA overwrite)
- ✅ vCenter-ESXi connectivity intact
- ✅ Running VMs unaffected throughout
- ✅ Backups available for rollback if needed
- ✅ Full audit trail in Beads

**Final Beads entry:**

```bash
bd add "ESXi TLS migration complete" \
  "All 6 ESXi hosts migrated to Step CA certificates. vCenter connectivity verified. Expires: <EXPIRY_DATE>"
```

---

# GUARDRAILS (ABSOLUTE — DO NOT VIOLATE)

| Rule | Consequence of Violation |
|------|-------------------------|
| Do NOT change ESXi hostnames | Identity corruption |
| Do NOT touch solution user or VMCA certs | vCenter connectivity breakage |
| Do NOT skip setting `vpxd.certmgmt.mode=custom` | VMCA will overwrite custom certs |
| Do NOT bypass preflight failures | Unknown state, potential failure |
| Do NOT proceed without operator confirmation | Loss of partnership model |
| Do NOT do multiple hosts simultaneously | Impossible to isolate failures |
| Do NOT execute commands without showing them first | Violates approval requirement |
| Do NOT close Beads without operator acknowledgment | Incomplete audit trail |

---

**Proceed deliberately. One host at a time. Fail fast. Explain everything. Show every command before executing.**
