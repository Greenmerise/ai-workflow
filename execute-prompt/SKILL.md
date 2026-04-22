---
name: execute-prompt
description: "Execute project.prompt.md to perform a full project rewrite. Relies on source control for rollback and diff review."
disable-model-invocation: true
---

# Execute Prompt

Perform a full project rewrite from `project.prompt.md`.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: fresh context (the prompt is re-read from disk; no memory of prior generations is used), autonomous execution after the initial confirmation, and failure handling for generation errors. Source control is the mechanism for review and rollback — the user is expected to have a clean working tree before running this skill.

Generation itself is performed by the main skill (not a subagent) because the output IS the action; there is no separate analysis phase to delegate.

## Step 1: Locate and read the prompt

1. Use Glob to locate `project.prompt.md` in the project root. If it does not exist or is empty, report that to the user and stop.
2. Read `project.prompt.md` in full. This is your generation instructions.

## Step 2: Confirm before proceeding

Use AskUserQuestion to present the user with a summary before executing:

- Note that this is a **full project rewrite** from `project.prompt.md`.
- Remind the user to have a clean working tree (uncommitted changes may be lost).
- Ask: **"Proceed? (yes/no)"**

If the user declines, stop immediately.

## Step 3: Generate the project

Carry out the generation instructions from `project.prompt.md`. Generate all code, configuration, and content files described or implied by the prompt.

While generating:
- **The prompt file is your sole source of truth.** Do NOT read any other project file (including `CLAUDE.md`, `techspec.md`, `issues.log.md`, `README.md`, `package.json`, or anything else) for generation context. The only input is `project.prompt.md`. This ensures the prompt is a self-contained, reproducible generation input.
- If the generated output mismatches the existing `CLAUDE.md` or other unrelated project files, that is acceptable — the user will reconcile via source control.
- Use your best judgment for anything the prompt does not specify.
- Do not leave placeholder or TODO comments — produce complete, working output.
- Overwrite existing files freely when the prompt calls for it. The user's source control preserves prior state.
- **Follow the failure handling rules in `EXECUTION_POLICY.md`** if generation breaks the build, produces invalid output that cannot be completed, or a dependency install fails.

## Step 4: Do NOT delete files outside the generation

This skill creates and overwrites files. It does NOT remove files that exist in the project but are not produced by the prompt. Stale files from prior runs or unrelated user content are the user's responsibility to clean up via source control or manually.

## Step 5: Final report

Report per `../FINAL_REPORT.md`. Use `N/A (no issue log writes)` for the three issue-related counts.
