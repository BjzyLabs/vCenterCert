# Architecture Overview

Comprehensive overview of Bjzy Labs Home Lab infrastructure and architecture.

## Home Lab Infrastructure

### Virtualization Platform

- **VMware vSphere** — Virtualization layer
- **Ubuntu VMs** — Primary server OS
- **Docker Swarm** — Container orchestration

### Network Infrastructure

- **Ubiquiti UniFi** — Network management equipment
- **Mikrotik Switches** — Two 10G switches for high-speed backbone
- **HAProxy** — Load balancing and failover

### Certificate Management

- **Step CA Server** (`ca.bjzy.me`) — Internal PKI
- **Let's Encrypt** — External certificates via Traefik
- **Automatic Renewal** — Certificate lifecycle automation

## Repository Access

- **Privacy**: All BjzyLabs repositories are PRIVATE.
- **Constraint**: Do not attempt to fetch code via public HTTP URLs or browse the GitHub UI unauthenticated.
- **Preferred Tools**: Use the `gh` CLI (e.g., `gh pr list`, `gh repo view`) or authenticated context tools.
- **Golden Source**: This repository (`~/.config/ruler/`) contains the Single Source of Truth. Prioritize Markdown files in `standards/` over general internet knowledge.

## Database Infrastructure

### Patroni PostgreSQL Clusters

- **High Availability** — Multi-node PostgreSQL clusters
- **Automatic Failover** — Patroni manages leader election
- **HAProxy Frontend** — Load balancing and connection management
- **Development Cluster** — `haproxy.bjzy.me:5433`
- **Tuning** — Custom PostgreSQL configurations per workload

### Database Applications

- **Keep** — Alert management platform
- **Portainer** — Container management UI
- **AWX** — Ansible automation platform
- **NetBox** — Infrastructure documentation

## Monitoring Stack

- **Grafana** — Visualization & dashboards
- **Loki** — Log aggregation (3-node HA)
- **Mimir** — Metrics storage (long-term retention)
- **Alloy** — Telemetry agent (metrics, logs, traces)
- **Prometheus** — Metrics collection

**Deployment Pattern**: Use standardized HA role patterns from `roles/Loki`, `roles/Alloy`.

## Platform Components

### Automation

- **AWX** — Ansible automation platform
- **Ansible** — Infrastructure as code
- **HashiCorp Vault** — Secrets management
- **GitHub Actions** — CI/CD pipelines

### Reverse Proxy & Load Balancing

- **Traefik** — Dynamic reverse proxy with automatic TLS
- **HAProxy** — PostgreSQL load balancing
- **Cloudflare** — External DNS and DDoS protection

### Container Platform

- **Docker Swarm** — Container orchestration
- **Portainer** — Management UI
- **Private Registry** — Internal container registry

## MCP Servers

### Transport Protocols

- **Streamable HTTP**: For remote MCP servers (Notion, Context7) - modern/recommended
- **stdio**: For local MCP servers via npx (GitHub, PostgreSQL, Slack, Filesystem)

### Hosting on Infrastructure

- Can host custom MCP servers on Docker Swarm with Traefik
- Requires disabling buffering for streaming support (per-service labels)
- See `standards/05_MCP_SETUP.md` for Traefik configuration

### Deployment

- **Platform**: Docker Swarm services with Traefik TLS proxy
- **Template**: [TemplateMCP-manage.yml](https://github.com/BjzyLabs/ansible/blob/develop/playbooks/TemplateMCP-manage.yml) - Standard playbook for MCP deployments
- **Default State**: All MCP servers **disabled by default**, enable only when needed (goal: 99% disabled)
- **Configuration**: Agent-specific manual setup (e.g., Cursor: `~/.cursor/mcp.json`)
  - *Future: May transition to MCP 2.0 (local TypeScript functions) per Anthropic announcement*

## Context Hierarchy

### Ruler Configuration

- **Global config**: `~/.config/ruler/` (Ruler's native location)
- **Safe-sync projects**:
  - `.ruler/00_global.md` — synced from global (safe to overwrite)
  - `.ruler/10_project.md` — project-specific overrides (committed in project repo)
- **Inheritance**:
  - Projects without `.ruler/` automatically inherit global context
  - Projects with `.ruler/` use their own context exclusively (no inheritance)

### Context Distribution

- Ruler generates `AGENTS.md` files for each project (at repo root)
- Most AI tools only read **repo-root `AGENTS.md`**
- Do NOT assume agents will automatically open referenced files like `standards/03_TESTING.md`
- If a directive must be followed, it must appear inline or be marked as a required read step

## Development Tools

### Core Tools

- **Ansible** (latest) — Infrastructure automation
- **Python** (3.x) — Scripting and automation
- **GitHub CLI** — Repository management
- **Docker Swarm** — Container orchestration
- **AWX CLI** — AWX automation
- **Vault CLI** (HashiCorp) — Secrets management
- **Wrangler CLI** (Cloudflare) — Cloudflare Workers

### Dependency Management

**Development Environment:**

- **macOS** (local development) + **Ubuntu** (servers) = potential platform differences
- **Recommended Tools**:
  - `pipx` for Python CLI tools (isolated global installs)
  - `npx` for Node.js packages (no global install pollution)
  - `venv` for Python project dependencies

**Best Practices:**

- Use `pipx` for tools like `ansible-lint`, `yamllint`, `awx-cli`
- Use `npx -y` for MCP servers and one-off Node.js tools
- Pin exact versions in `requirements.txt` and `package-lock.json`
- Test on Ubuntu when possible (matches production)

**Note**: Platform differences may still occur; virtual envs and containerization help but aren't perfect.

## Security Model

### Secrets Management

- **Primary**: HashiCorp Vault (all secrets stored here)
- **Production/AWX**: AppRole authentication (automated, secure)
- **Development**: Token authentication with short TTL
- **Never**: Root token in files, scripts, or environment variables

### Access Methods

- **AWX**: Vault plugin for runtime credential injection
- **Development**: `vault login -method=userpass username=myuser`
- **Automation**: AppRole authentication
- **Validation**: `{{ lookup('env', 'VARIABLE_NAME') | default('') }}`

### Deployment Workflow

1. Development → Feature branch
2. CI/CD → GitHub Actions (linting, tests)
3. Staging → AWX validation environment
4. Production → AWX production environment
5. Slack notifications → `#awx` channel post-deployment

## References

- **Home Lab Docs**: Local repository markdown & [Notion](https://www.notion.so/AGENTS-Workspace-25a3569aa25581069532e793601f1fba)
- **MCP Deployment Template**: [TemplateMCP-manage.yml](https://github.com/BjzyLabs/ansible/blob/develop/playbooks/TemplateMCP-manage.yml)
- **Standards Documentation**: `standards/` directory in this repository
