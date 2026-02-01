# Bjzy Labs Global Context

Unified instructions for all AI coding assistants working with Bjzy Labs Home Lab infrastructure.

## Critical Rules (Zero Tolerance)

1. **NO hardcoded secrets** — All secrets in HashiCorp Vault
2. **NO test failures** — Never merge when tests or verification fail
3. **Branch from `develop`** — Never branch from `main` for features
4. **NO placeholder code** — Always implement complete, working solutions
5. **Hostname SAFETY** — `devHuey` ≠ `Huey` (dev vs prod are DIFFERENT servers)
6. **Progress Tracking (MANDATORY)** — All non-trivial changes tracked with Beads → `standards/10_PROGRESS_TRACKING.md`

## Test-Driven Development (TDD)

**Core Principle:** All work must be provable by a test which is designed and programmed *first*.

1. **Write tests first** — Create failing tests that define expected behavior before implementation.
2. **Red → Green → Refactor** — Follow the cycle:
   - 🔴 **Red**: Write a failing test
   - 🟢 **Green**: Write minimal code to make it pass
   - 🔵 **Refactor**: Improve quality while keeping tests green
3. **Test types by context**:
   - **Ansible Roles**: Use Molecule with `verify.yml` assertions
   - **Python Functions**: Use `pytest` with fixtures
   - **AWX Playbooks**: Include `operation=test` for infrastructure validation
   - **Infrastructure Changes**: Add verification tasks that confirm expected state
4. **Mandatory Proof** — All PRs must include tests. Never skip tests.
5. **Propose First** — When implementing, propose the test structure *before* implementation.

**Detailed TDD Standards** → `standards/03_TESTING.md`

### Standards & Gates
- **Linting**: `ansible-lint`, `yamllint`, `shellcheck` (zero violations)
- **Coverage**: 80-90% target, focus on critical paths
- **Zero Tolerance**: NO merge when tests fail
- **Testing Stages**: Local dry-run → CI → AWX staging → Production

## Core Tools

**Installed**: Ansible | Python 3.x | GitHub CLI (`gh`) | Docker Swarm | AWX CLI | Vault CLI
**MCP Servers**: All **disabled by default**, enable only when needed → `standards/05_MCP_SETUP.md`

## Quick Start

```bash
# GitHub - Create PR (always target develop)
gh pr create --base develop --title "feat: description"

# Ansible - Validation only (never run -manage.yml locally!)
ansible-playbook --check playbooks/[name].yml
ansible-lint playbooks/[file].yml

# Vault - Secure access (never use root token!)
export VAULT_ADDR="https://vault.bjzy.me:8200"
vault login -method=userpass username=myuser
```

**Full command reference** → `standards/00_QUICK_REFERENCE.md`

## GitFlow (MANDATORY)

- **Feature/bugfix**: Branch from `develop`, PR to `develop`
- **Hotfix**: Branch from `main` (emergency only)
- **Conventional commits**: `type(scope): description` (e.g., `feat(ansible): add monitoring role`)
- **Repo exception (`BjzyLabs/myContext`)**: PRs are optional for merges to `develop` or `main`

**Complete workflow** → `standards/02_GITFLOW.md`

## Code Standards

**Quick summary:**

- **Python**: PEP 8, type hints, virtual envs
- **Ansible**: 2-space indent, `ansible.builtin.*` modules, `changed_when: false` for reads, 160 char max
- **Shell**: `#!/bin/bash`, `set -euo pipefail`, quote all variables
- **Linting**: Zero violations required before commit

**Detailed conventions** → `standards/01_CODE_CONVENTIONS.md`

## Security (Non-Negotiable)

- **Secrets**: HashiCorp Vault only (never in code, logs, or environment files)
- **Vault Access**:
  - **Production/AWX**: AppRole authentication (automated)
  - **Development**: `vault login -method=userpass` (short TTL)
  - **NEVER**: Root token on disk or in scripts
- **Privilege**: `become: true` only when necessary
- **Hostname Safety**: Distinguish dev/prod (e.g., `devHuey` ≠ `Huey`)

**Full security guidelines** → `standards/04_SECURITY.md`


## Infrastructure Essentials

**Ansible Operations**: `install` | `uninstall` | `uninstall-purge` | `status` | `config`
**Secrets Flow**: Vault → AWX (runtime injection via plugin) → Playbooks
**Notifications**: Post-deployment Slack alerts to `#awx` channel
**Monitoring Stack**: Grafana, Loki (3-node HA), Mimir, Alloy

**Architecture details** → `standards/07_ARCHITECTURE.md`

## Planning & Archival

**Standard**: Prescriptive planning required for all major changes.
**Process**: Design (Implementation Plan) → Execute (Source of Truth) → Archive (to `/docs`).
**Flexibility**: Supports multiple planning formats (e.g., SpecKit, BMAD, Gemini Conductor).

**Planning details** → `standards/08_PLANNING.md`

## Conduit Parallel Agent Planning

- Use a TASK-0 Foundation for shared changes; merge before parallel work
- Every task prompt must declare SCOPE and DO NOT TOUCH; no file overlap
- Use phased execution when dependencies exist between tasks

**Conduit planning details** → `standards/09_CONDUIT_PARALLEL_AGENTS.md`

## AI Behavior

**Professional, Senior Engineer tone:**

- Always explain *why* changes are requested
- Use code blocks for suggestions
- Be concise and thorough
- Ask for missing information rather than using placeholders

**Workflow Preferences:**
- **Show edits as you go**: Provide visibility into each change during multi-file operations
- Explain what's being added/modified before executing
- Summarize impact after each file

**Efficiency:**

- Do not generate speculative code or "just in case" scripts
- Provide requested solution only
- For multiple solutions, provide high-level summaries first

**Review Priorities** (in order):

1. **Security** — Credential leaks, injection vulnerabilities
2. **Performance** — N+1 queries, memory leaks
3. **Readability** — Naming, structure, clarity

## State Preservation

**Rule**: When a working configuration is achieved, immediately prompt the user to **Commit and Push** to the remote feature branch.
**Reasoning**: Local commits are insufficient. Pushing to GitHub ensures the git hash is secured off-site.

## References

**Standards Documentation** (Global Context):

> **For AI Agents**: When a topic references a standards file, proactively read it using the absolute path below.

- `standards/00_QUICK_REFERENCE.md` — Common commands
- `standards/01_CODE_CONVENTIONS.md` — Code style and conventions
- `standards/02_GITFLOW.md` — Branch strategy and workflow
- `standards/03_TESTING.md` — Testing requirements and best practices
- `standards/04_SECURITY.md` — Security guidelines
- `standards/05_MCP_SETUP.md` — MCP server configuration
- `standards/06_MCP_SERVERS_REFERENCE.md` — MCP server catalog
- `standards/07_ARCHITECTURE.md` — Infrastructure architecture
- `standards/08_PLANNING.md` — Prescriptive planning and archival standards
- `standards/09_CONDUIT_PARALLEL_AGENTS.md` — Conduit parallel agent planning
- `standards/10_PROGRESS_TRACKING.md` — Progress tracking with Beads (mandatory)

**External Resources:**

- [MCP Template Playbook](https://github.com/BjzyLabs/ansible/blob/develop/playbooks/TemplateMCP-manage.yml)
- [Home Lab Docs (Notion)](https://www.notion.so/AGENTS-Workspace-25a3569aa25581069532e793601f1fba)
