# Quick Reference Commands

Common commands for daily operations in the Bjzy Labs infrastructure.

## GitHub

```bash
# Monitor CI workflows
gh run list --watch

# Create PR (always target develop)
gh pr create --base develop --title "feat: description"

# View issue with ALL details including inline comments
gh issue view [number] --comments

# List pull requests
gh pr list --state open

# View repository
gh repo view
```

## Release Process (Ansible Roles, Libraries)

### Standard Release Flow

```bash
# 1. Create release PR from develop to main
gh pr create --base main --head develop \
  --title "Release vX.Y.Z" \
  --body "Release notes here"

# 2. Wait for checks to complete
# (GitFlow Guard should pass automatically for develop→main)

# 3. Merge with admin override (bypasses any timing issues)
gh pr merge <PR_NUMBER> --squash --admin

# 4. Tag the release
git fetch origin main && git checkout main && git pull
git tag -a vX.Y.Z -m "Release vX.Y.Z

- Feature 1
- Fix 2
- Change 3"
git push origin vX.Y.Z

# 5. Create GitHub release
gh release create vX.Y.Z \
  --title "vX.Y.Z - Brief Description" \
  --notes "## What's Changed

- Feature 1
- Fix 2

**Full Changelog**: https://github.com/BjzyLabs/<REPO>/compare/vPREVIOUS...vX.Y.Z"
```

### Quick Release (Single Command)

For simple releases when checks pass:

```bash
VERSION="v1.0.0" && \
REPO="ansible-role-haproxy" && \
gh pr create --repo "BjzyLabs/$REPO" --base main --head develop \
  --title "Release $VERSION" --body "Release $VERSION" && \
sleep 10 && \
gh pr merge $(gh pr list --repo "BjzyLabs/$REPO" --head develop --base main --json number -q '.[0].number') \
  --repo "BjzyLabs/$REPO" --squash --admin && \
cd /tmp && rm -rf "$REPO" && git clone "git@github.com:BjzyLabs/$REPO.git" && cd "$REPO" && \
git tag -a "$VERSION" -m "Release $VERSION" && \
git push origin "$VERSION" && \
gh release create "$VERSION" --repo "BjzyLabs/$REPO" --generate-notes
```

## Ansible

```bash
# ⚠️ CRITICAL: NEVER run -manage.yml playbooks outside AWX except in emergencies!
# These playbooks require AWX-supplied credentials and WILL FAIL without them!

# Safe operations (validation only):
ansible-playbook --syntax-check playbooks/[name].yml
ansible-playbook --check playbooks/[name].yml  # Dry run
ansible-lint playbooks/[file].yml
```

## Docker Swarm

```bash
# List services
docker service ls

# View service logs (follow)
docker service logs [service_name] -f

# Update service
docker service update [service_name]

# View service details
docker service inspect [service_name]

# Check service status
docker service ps [service_name]
```

## HashiCorp Vault CLI

**Secure Terminal Access:**

```bash
# Set vault address
export VAULT_ADDR="https://vault.bjzy.me:8200"

# Option 1: Login with userpass (creates short-lived token)
vault login -method=userpass username=myuser

# Option 2: AppRole for automation
vault write auth/approle/login \
  role_id="$ROLE_ID" \
  secret_id="$SECRET_ID"

# Retrieve secrets
vault kv get kvProd/path/to/secret

# Your token is stored in ~/.vault-token (auto-expires)
```

**Never do this:**

```bash
# ❌ DON'T store root token in shell profile
export VAULT_TOKEN="hvs.root-token-here"  # BAD!
```

## Standard Ansible Operations

All playbooks support these operations:

- `install` — Deploy/install the service
- `uninstall` — Remove service (keep data)
- `uninstall-purge` — Remove service and all data
- `status` — Check service status
- `config` — View/update configuration

## Linting

```bash
# Ansible
ansible-lint playbooks/[file].yml
yamllint playbooks/[file].yml

# Shell scripts
shellcheck [file].sh

# Python
pylint src/ tests/
```

## Testing

```bash
# Local testing
ansible-playbook --check playbooks/[file].yml

# Run tests
pytest tests/ -v --cov=src

# JavaScript/TypeScript
npm test
```
