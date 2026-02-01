# vCenter Certificate Renewal

A comprehensive, test-driven process for renewing vCenter Machine SSL (HTTPS) certificates using Smallstep Step CA.

## Overview

This repository preserves a proven, repeatable process for vCenter certificate renewal that was originally documented in personal notes. The goal is to never lose this process again and make it readily available for future certificate renewal operations.

**Key Features:**

- ✅ **Step CA Integration** — Automated certificate issuance via Smallstep Step CA
- ✅ **Multi-SAN Support** — Current FQDN, future FQDN, short name, and IP address
- ✅ **Preflight Validation** — Comprehensive checks before any changes
- ✅ **Safety-First** — Explicit checkpoints and operator approval gates
- ✅ **Beads Tracking** — Full audit trail via Beads progress tracking system
- ✅ **Rollback Ready** — Timestamped backups and recovery procedures

## Quick Start

### Prerequisites

- macOS with `step`, `openssl`, `ssh`, `scp`, `curl`, `jq` CLI tools installed
- SSH root access to vCenter Server Appliance (VCSA)
- Provisioner password for Step CA (in HashiCorp Vault or operator-provided)
- Step CA already bootstrapped or accessible at `https://ca.bjzy.me`

### Begin Renewal Process

1. Ensure Beads (`bd`) is installed on your system
2. Review the complete renewal prompt:

   ```bash
   cat docs/vcenter-tls-renewal-prompt.md
   ```

3. Provide the prompt to your infrastructure automation agent (e.g., Sonnet)
4. Follow along as the agent executes each phase with your approval
5. Validate the new certificate is active and services are healthy

## What's Included

| File | Purpose |
|------|---------|
| `README.md` | This file — overview and quick start |
| `docs/vcenter-tls-renewal-prompt.md` | Complete renewal process with all phases, checks, and guardrails |
| `docs/step-ca-root.crt` | Step CA root certificate (public, for reference) |
| `standards/` | Bjzy Labs global standards (GITFLOW, testing, security, etc.) |

## Process Highlights

### Phases

1. **Preflight Validation** — All tools, credentials, and access verified
2. **Backup Existing Certificate** — Timestamped backup before any changes
3. **Issue New Certificate** — Request multi-SAN cert via Step CA
4. **Prepare Certificate Files** — Split PEM into VMware-compatible format
5. **Transfer to VCSA** — SCP files to vCenter
6. **Apply Certificate** — Use VMware certificate-manager to replace
7. **Validation** — Verify new cert active, all SANs functional, services healthy

### Guardrails

- ❌ No hostname or PNID changes
- ❌ No VMCA or solution user cert regeneration
- ❌ No preflight bypass
- ✅ Operator approval required at every checkpoint
- ✅ Full Beads audit trail
- ✅ Rollback procedure included

## Certificate Details

**Certificate Identity:**

- **CN:** `vcenter.homelab.bjzy.me` (current, authoritative)
- **SANs:**
  - `vcenter.homelab.bjzy.me` (current FQDN)
  - `vcenter.bjzy.me` (future FQDN — enables future migration)
  - `vcenter` (short name)
  - `192.168.30.40` (IP address — WiFi connection, DNS fallback)
- **Validity:** 1 year
- **Key Type:** RSA 2048
- **Issuer:** Smallstep Step CA

## Notes for Operators

- **Service Impact:** vCenter Web UI and API will be briefly unavailable during certificate replacement (~5 minutes)
- **Running VMs:** Not affected — they continue on ESXi hosts
- **Rollback:** Full backup and recovery procedure included if needed
- **Network:** vCenter is WiFi-connected; IP address `192.168.30.40` is included as a SAN for failover access if DNS becomes unavailable
- **⚠️ VECS Troubleshooting:** If `certificate-manager` completes but the OLD cert is still served, VECS may need manual update. See "VECS Fallback Procedure" in `vcenter-tls-renewal-prompt.md`

## Repository Standards

This repository follows **Bjzy Labs GitFlow** and code standards:

- Branches: `main` (production), `develop` (integration)
- Commit format: `type(scope): description` (e.g., `docs: update certificate renewal process`)
- Branch protection: `main` and `develop` protected per standards
- Testing: All changes validated before merge

See `standards/02_GITFLOW.md` for complete workflow details.

## Questions?

Refer to:

- `docs/vcenter-tls-renewal-prompt.md` — Complete step-by-step renewal guide
- `standards/` — Bjzy Labs infrastructure standards and best practices
- Home Lab Docs — [AGENTS Workspace (Notion)](https://www.notion.so/AGENTS-Workspace-25a3569aa25581069532e793601f1fba)

---

**Last Updated:** February 1, 2026
**Maintained By:** Bjzy Labs Infrastructure Team
**Repository:** [BjzyLabs/vCenterCert](https://github.com/BjzyLabs/vCenterCert)
