# Prescriptive Planning & Archival

For all major architectural changes, lifecycle operations, or multi-step feature epics, agents must adhere to the 'Plan-First' standard:

## Design
Before execution, create a fully prescriptive Implementation Plan. While this is ideally a single `implementation_plan.md`, agents may employ other structured planning methods (e.g., SpecKit, BMAD, or Gemini Conductor) that result in multiple markdown files. Regardless of the format, the plan must contain the exact logic, copy-paste ready code blocks, and a step-by-step verification checklist.

## Execute
Use the defined plan(s) as the absolute source of truth during the coding and deployment phase to ensure consistency across all modified files.

## Archive
Upon successful completion and merge, archive the final "as-built" version of the implementation plan and any associated planning artifacts into the repository's `/docs` directory (e.g., `docs/feature_name_plan.md` or a dedicated subdirectory).

## Rationale
This maintains a high-fidelity audit trail, documents the "why" behind complex logic, and ensures that any mission-critical workflow can be perfectly audited or recreated in the future.
