# Git Workflow (GitFlow)

Standard branching strategy for all projects. **MANDATORY for all contributors.**

## Core Principle

- **Branch from `develop`** (NOT `main`)
- **Target `develop`** for all PRs
- **`main` is production releases only**

## Repository Exception (`BjzyLabs/myContext`)

For `myContext` only, PRs are optional for merges to `develop` or `main`.

## Branch Types

| Branch | Source | Target | Purpose |
|--------|--------|--------|---------|
| `feature/*` | develop | develop | New features |
| `bugfix/*` | develop | develop | Non-critical bugs |
| `hotfix/*` | main | main & develop | Emergency production fixes |
| `develop` | - | - | Integration branch, CI target |
| `main` | develop | - | Production releases, auto-deploy |

## Quick Start

### Starting a Feature
```bash
git checkout develop
git pull origin develop
git checkout -b feature/user-authentication

# Make changes...
git add .
git commit -m "feat: implement user authentication"
git push origin feature/user-authentication

# Create PR targeting develop
gh pr create --base develop
```

### Starting a Bugfix
```bash
git checkout develop
git pull origin develop
git checkout -b bugfix/login-error

# Fix bug...
git add .
git commit -m "fix: resolve login validation error"
git push origin bugfix/login-error
gh pr create --base develop
```

### Emergency Hotfix (Rare!)
```bash
git checkout main
git pull origin main
git checkout -b hotfix/security-patch

# Apply fix...
git commit -m "hotfix: patch security vulnerability"
git push origin hotfix/security-patch

# Create PR to main (requires admin approval)
gh pr create --base main
# Then also merge to develop
```

## Branch Protection Rules

**Automated enforcement:**
- ✅ Direct commits to `main` blocked
- ✅ PRs to wrong branch automatically rejected
- ✅ Status checks (CI/CD, tests) must pass
- ✅ Code review required before merge

### Branch Protection Standard (Bjzy Labs)

**MANDATORY for all Bjzy Labs repositories.**

This is the standardized configuration to be applied to ALL current and future repositories in the home lab:

#### Protected Branches
- `main`
- `develop`

#### Recommended Branch Protection Settings (Updated Jan 2026)

For **`main`** branch:

- **Required status checks:**
  - `GitFlow Guard / Validate GitFlow rules`
  - `strict: false` (Don't require branch up-to-date; reduces friction)
- **Required pull request reviews:**
  - Required approving reviews: `0` (Optional for small teams)
  - Dismiss stale reviews: `true`
- **Enforce restrictions for admins:** `false` (Allow admin override for releases)
- **Allow squash merging:** `true`
- **Allow merge commits:** `false`
- **Allow rebase merging:** `false`

**Why these settings:**
- `strict: false`: Avoids requiring `git pull` before every merge (develop→main rarely conflicts).
- `enforce_admins: false`: Allows quick releases when checks pass but GitHub API has timing issues.
- **Squash-only**: Keeps main history clean with one commit per PR.

#### Setting Branch Protection via CLI

```bash
# Initial setup for new repo
gh api --method PUT repos/BjzyLabs/<REPO>/branches/main/protection --input - <<'EOF'
{
  "required_status_checks": {
    "strict": false,
    "checks": [
      {"context": "GitFlow Guard / Validate GitFlow rules"}
    ]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": false,
    "required_approving_review_count": 0
  },
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
EOF

# Update existing repo (just fix the problematic settings)
gh api --method PATCH repos/BjzyLabs/<REPO>/branches/main/protection/required_status_checks \
  --field strict=false

gh api --method DELETE repos/BjzyLabs/<REPO>/branches/main/protection/enforce_admins
```

#### For AI Agents: Handling Releases

When user requests a release or you need to merge develop→main:

1. **Check if checks are passing** before attempting merge:
   `gh pr checks <PR_NUMBER>`
2. **Always use `--admin` flag** for release merges:
   `gh pr merge <PR_NUMBER> --squash --admin`
3. **If merge still fails, check branch protection**:
   `gh api repos/BjzyLabs/<REPO>/branches/main/protection -q '{enforce_admins: .enforce_admins.enabled, strict: .required_status_checks.strict}'`
4. **Common failure modes**:
   - `enforce_admins: true`: Admins blocked by checks (should be `false`)
   - `strict: true`: Branch must be up-to-date (should be `false`)
   - Check status not registered: Wait 10-30s and retry
   - Merge method not allowed: Use `--squash` only
5. **Never silently fail**: If merge fails, explain to user what's blocking and offer to fix branch protection.

## Release Process

1. Features merged to `develop`
2. `develop` tested thoroughly
3. Create release branch: `release/v1.2.0`
4. Final testing and bug fixes
5. Merge to `main` and tag version
6. Merge back to `develop`

## What NOT to Do

```bash
# ❌ WRONG - Never target main for features
gh pr create --base main

# ❌ WRONG - Don't branch from main for features
git checkout main
git checkout -b feature/my-feature

# ❌ WRONG - Don't commit directly
git push --force-with-lease (destructive!)
```

## Non-Compliance

PRs that violate GitFlow will be:
- Automatically rejected by branch protection
- Closed with explanation
- Required to be recreated with proper targeting

**Remember:** This protects production stability and ensures smooth collaboration.