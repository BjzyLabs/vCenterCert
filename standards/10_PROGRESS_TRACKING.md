# Progress Tracking with Beads (MANDATORY)

Every non-trivial change must be tracked with Beads from start to finish.

## What Constitutes "Non-Trivial"

- **Multi-step implementations** (3+ distinct phases or tasks)
- **Multi-file changes** (touching 3+ files)
- **Feature additions or significant refactoring**
- **Any work requiring git commits** (if you're committing, you're tracking)

## The Rule

1. **FIRST**: Initialize Beads (`bd init`) or verify database exists
2. **BEFORE CODING**: Create epics/tasks for each phase
3. **DURING WORK**: Update task status (`bd set-state <id> in_progress` → `bd close <id>`)
4. **NO SHORTCUTS**: Do not skip this for "quick" work; estimation is not your job
5. **VISIBILITY**: Progress tracking is non-negotiable for accountability and handoff context

## Why This Matters

- **Auditability** — Future you (or another agent) can see exactly what was done and why
- **Context Preservation** — Task descriptions capture decisions and blockers
- **Dependency Management** — Beads prevents circular work and parallelization errors
- **Agent Autonomy** — Clear task graph allows parallel agents to work safely
- **Handoff Ready** — Another agent can pick up work with full context

## Workflow Example

```bash
# Initialize or verify Beads exists
bd init

# Before starting: Create epic/task structure
bd create-epic "Feature: Add authentication"
bd create-task "epic_id" "Implement login endpoint" "Design and code login"
bd create-task "epic_id" "Add JWT validation" "Validate tokens in middleware"

# During work: Mark tasks as in progress and close when done
bd set-state <task_id> in_progress
# ... code ...
bd close <task_id>

# View progress
bd list
bd show <task_id>
```

## When Friction Arises

If you find Beads setup adding friction, report it — the tool should serve you, not the reverse. But the practice of **structured tracking is not optional**.

## Integration with Planning

Beads tracking complements (does not replace) the prescriptive planning in `standards/08_PLANNING.md`:
- **Planning**: Design phase produces an implementation plan with steps
- **Tracking**: Beads records execution, decisions, blockers, and actual outcomes
- **Archival**: Together, these form the complete audit trail

For major features, your implementation plan should map to a Beads epic with sub-tasks for each phase.
