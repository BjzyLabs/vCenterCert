# vCenter Certificate Renewal

A comprehensive, test-driven process for renewing vCenter Machine SSL (HTTPS) certificates using Smallstep Step CA.

## Overview

This repository documents a **proven, repeatable process** for renewing vCenter Machine SSL and ESXi host SSL certificates using Smallstep Step CA. All procedures have been successfully executed and validated in production (Feb 1-2, 2026).

**Current Status (Feb 2, 2026):**
- ✅ **vCenter VCSA** — Certificate renewed and deployed (expires Feb 1 2027)
- ✅ **All 6 ESXi Hosts** — Certificates migrated from VMCA to Step CA (expire Feb 1-2 2027)
- ✅ **Unified PKI** — vCenter + all ESXi hosts now on Step CA

**Key Features:**

- ✅ **Step CA Integration** — Automated certificate issuance via Smallstep Step CA
- ✅ **Multi-SAN Support** — Current FQDN, future FQDN, short name, and IP address
- ✅ **Vault Automation** — Credentials retrieved from Vault; zero manual password entry (for ESXi)
- ✅ **Preflight Validation** — Comprehensive checks before any changes
- ✅ **Safety-First** — Explicit checkpoints and operator approval gates
- ✅ **Beads Tracking** — Full audit trail via Beads progress tracking system
- ✅ **Rollback Ready** — Timestamped backups and recovery procedures
- ✅ **Bash Patterns Documented** — All complex command patterns preserved in Notion for future reference

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
| `README.md` | This file — overview and current status |
| `docs/vcenter-tls-renewal-prompt.md` | vCenter renewal procedure (9 phases, fully documented) |
| `docs/esxi-tls-migration-prompt.md` | ESXi migration procedure with **Vault automation** for 6 hosts |
| `docs/step-ca-root.crt` | Step CA root certificate (public, for reference) |
| `standards/` | Bjzy Labs global standards (GITFLOW, testing, security, etc.) |
| **Notion Documentation** | [Complete deployment process with all bash patterns](https://www.notion.so/2fb3569aa255818c918fca25c11d617a) — includes challenges, solutions, and automation approach |

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

## Technical Documentation

**For Detailed Bash Patterns & Solutions:**
- [Complete Deployment Process (Feb 2026) — Notion](https://www.notion.so/2fb3569aa255818c918fca25c11d617a)
  - All bash command patterns used in deployment
  - Challenges encountered and solutions
  - Vault automation approach for future renewals
  - Critical file transfer, certificate generation, and validation techniques

**For Procedural Details:**
- `docs/vcenter-tls-renewal-prompt.md` — vCenter renewal (9 phases)
- `docs/esxi-tls-migration-prompt.md` — ESXi migration (with Vault automation)

**For Infrastructure Standards:**
- `standards/` — Bjzy Labs infrastructure standards and best practices
- Home Lab Docs — [AGENTS Workspace (Notion)](https://www.notion.so/AGENTS-Workspace-25a3569aa25581069532e793601f1fba)

---

**Last Updated:** February 2, 2026
**Status:** ✅ Complete — All hosts (vCenter + 6 ESXi) deployed and validated
**Maintained By:** Bjzy Labs Infrastructure Team
**Repository:** [BjzyLabs/vCenterCert](https://github.com/BjzyLabs/vCenterCert)
