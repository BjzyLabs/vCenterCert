# Code Conventions & Commit Standards

Consistent coding and commit practices across all projects.

## Naming Conventions

- `snake_case` for variables and functions
- `PascalCase` for classes
- `UPPER_SNAKE_CASE` for constants

## Language-Specific Standards

### Python
- Follow PEP 8 style guide
- Use type hints for function signatures
- Use virtual environments for project isolation
- Requirements.txt for dependency management

### Ansible
- 2-space YAML indentation
- Use `ansible.builtin.*` module names (explicit)
- Add `changed_when: false` for read-only tasks
- Maximum 160 characters per line
- Run `ansible-lint` before commits

### Shell Scripts
- Use bash shebang: `#!/bin/bash`
- Set error handling: `set -euo pipefail`
- Quote all variables: `"${var}"`
- Run `shellcheck` before commits

### Frontend (JavaScript/TypeScript/CSS)
- Use Prettier for automatic formatting
- Follow ESLint rules for JavaScript/TypeScript

## Git Conventions

### Commit Messages
Use conventional commits format: `type(scope): description`

**Types:**
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `style:` - Code style (formatting, missing semicolons)
- `refactor:` - Code restructuring without feature changes
- `perf:` - Performance improvements
- `test:` - Adding or updating tests
- `ci:` - CI/CD configuration changes
- `chore:` - Build process, dependencies, tooling
- `hotfix:` - Emergency production fixes

**Examples:**
```
feat(ansible): add new monitoring role
fix(api): resolve null pointer in user handler
docs(readme): update installation instructions
refactor(core): simplify authentication logic
test: add comprehensive test coverage for payments
```

### Branching Strategy
- **Feature branches**: `feature/description` (from develop)
- **Bug fixes**: `bugfix/description` (from develop)
- **Hotfixes**: `hotfix/description` (from main, emergency only)
- Always branch from `develop` for new work (NOT main)
- Target `develop` in pull requests
- `main` branch reserved for production releases

### Pull Request Workflow
1. Branch from `develop`
2. Make changes with conventional commits
3. Create PR targeting `develop`
4. CI must pass before merge
5. Code review required
6. Merge after approval

See `02_GITFLOW.md` for complete workflow details.

