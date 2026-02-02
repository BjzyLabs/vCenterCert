# Conduit Parallel Agent Planning

When running parallel AI coding agents in Conduit (any CLI: Claude Code, Aider, Cursor, Cline, etc.), follow these rules to prevent merge conflicts and overlapping work.

## Rule 1: Foundation First

If multiple tasks share dependencies, create a TASK-0: Foundation that runs before parallel work and merges first.

TASK-0: Foundation (sequential - must merge first)
- Shared docs (e.g., docs/IMPLEMENTATION_PLAN.md)
- requirements.yml / package.json / pyproject.toml changes
- Shared module or role skeletons
- Entrypoint stubs (playbooks, main.py, etc.)

TASK-1 through TASK-N: Features (parallel - after TASK-0 merges)
- Each touches only its isolated directory

## Rule 2: Explicit File Boundaries

Every task prompt must include SCOPE and DO NOT TOUCH sections.

```markdown
## SCOPE (files you may create/modify)
- `src/my-feature/*` (entire directory)
- `tests/my-feature/*` (tests for this module)

## DO NOT TOUCH (protected files)
- `requirements.yml` / `package.json` - handled by Foundation task
- `docs/*` - handled by Foundation task
- `src/main.py` or entrypoints - handled by Foundation task
- Any directory other than your assigned one
```

## Rule 3: True Parallelism Test

Before marking tasks as parallel, verify:

1. Zero file overlap - no two tasks touch the same file
2. No implicit dependencies - task B does not need task A's output
3. Independent validation - each task can be tested in isolation

If any condition fails, tasks must be sequential or phased.

## Rule 4: Phased Execution Pattern

When tasks have dependencies but some parallelism is possible:

Phase 1 (parallel):   TASK-1, TASK-4  -- merge all
Phase 2 (parallel):   TASK-2, TASK-5  -- merge all (branch from Phase 1)
Phase 3 (sequential): TASK-3          -- merge (depends on Phase 2)

Each phase must fully merge before the next phase branches.

## Rule 5: Task Prompt Template

```markdown
# TASK-N: [Feature Name]

## Objective
[Single sentence describing the deliverable]

## SCOPE (files you may create/modify)
- `path/to/allowed/directory/*`

## DO NOT TOUCH
- Dependency manifests (requirements.yml, package.json, etc.)
- Shared documentation (docs/*)
- Entrypoints and shared modules
- Any directory not explicitly listed in SCOPE

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Linting passes with zero violations

## Branch Naming
`feature/[feature-name]`

## On Completion
1. Commit with conventional commit message
2. Push to remote
3. Create PR to `develop` (or specified base branch)
4. Report: PR number, files changed, any blockers
```

## Rule 6: Post-Parallel Merge Strategy

1. Merge in dependency order - Foundation first, then independent features
2. Rebase before merge - if conflicts arise, rebase the PR branch on updated base
3. Squash merge recommended - keeps history clean when parallel branches have messy commits

## Anti-Patterns (Never Do This)

- "Tasks 1-5 can run in parallel" without verifying zero file overlap
- Multiple tasks modifying shared manifests or config files
- Assuming agents will figure out boundaries implicitly
- Parallel tasks where one logically depends on another's output
- Skipping the Foundation task to save time
