# Repo-persisted project state and handoff

Use this pattern for multi-session implementation/research projects where the user wants continuity without relying on chat context.

## Trigger

- User says to continue prior work, but not to keep long-term task state in chat/context.
- User asks for project construction over multiple phases.
- Work creates data/audit/modeling artifacts that future sessions must understand.

## Repository files to create/update

Prefer these durable files inside the repository:

- `docs/project_brief.md` — stable goal, scope, target outputs.
- `docs/confirmed_decisions.md` — decisions that constrain future work.
- `docs/non_negotiable_constraints.md` — things future agents must not violate.
- `docs/implementation_plan.md` — phased plan with acceptance checks.
- `docs/progress_log.md` — append-only phase status; current state and next gated step.
- `docs/data_inventory.md` — copied/generated data paths, source, shape/size, git-ignore status.
- `docs/open_questions.md` — unresolved items requiring user decision.
- `configs/paths.yaml` — project-relative runtime paths only.

Generated outputs should live in `outputs/` and be git-ignored unless the user asks otherwise. Large inputs should live in `data/` and be git-ignored.

## Workflow

1. Read repo docs first; treat them as the source of truth over chat memory.
2. Before side-effectful work, verify git status for both current and source/old repos if copying from another project.
3. Encode constraints in repo docs, not only in the response.
4. Use project-relative paths after any data copy; add a path guard when accidental old-project access would be harmful.
5. For scientific ML repos, run audit/split-feasibility before formal training.
6. If formal training is out of scope or gated by confirmation, implement only safe skeletons and smoke/audit checks.
7. Append progress to `docs/progress_log.md` before final response.
8. Final response should be short: what changed, verification, key output paths, and any gated next step.

## Pitfalls

- Do not leave “next steps” only in chat. Put them in `docs/progress_log.md` or `docs/open_questions.md`.
- Do not persist transient command failures as project facts; persist the final reproducible command or dependency in repo files.
- Do not report huge generated file lists in the final response; point to inventory/report files.
- Do not require a clean old/source repo if baseline status already had pre-existing user changes; compare against baseline instead.
