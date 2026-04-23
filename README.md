# Project Skills

*Insert this content into the `.claude/skills` folder of your project or in your user profile folder to use them in a new claude session*

Skills for building and maintaining a project with AI. The system adds structure to two recurring activities:

1. **Generating a project** from a written intent (`project.prompt.md`).
2. **Maintaining a project** against an as-built specification (`techspec.md`) and an issue log (`issues.log.md`).

## Verb convention

- **`evaluate-*`** skills are read-only analyses. They report findings to the user.
- **`execute-*`** skills modify files — they write, generate, or apply fixes. None auto-invoke; they run only when the user invokes them.

Classification by behavior, not prefix, determines which policy rules apply (see `EXECUTION_POLICY.md`).

## The skills

| Skill | Role |
|---|---|
| `initialize-project` | Scaffolds `project.prompt.md`, `CLAUDE.md`, `techspec.md`, `issues.log.md` at the project root. |
| `evaluate-prompt` | Assesses `project.prompt.md` for clarity. Read-only. |
| `execute-prompt` | Generates the full project from `project.prompt.md`. Overwrites existing files; relies on source control for rollback. |
| `execute-techspec` | Writes a comprehensive `techspec.md` describing the current codebase. |
| `execute-techreview` | Analyzes `techspec.md` for design flaws, logs them to `issues.log.md`, prompts user to pick fixes, and applies them to both spec and code. |
| `execute-debug` | Scans the codebase for bugs, logs them to `issues.log.md`, prompts user to pick fixes, and applies them. |
| `report-bug` | Accepts a user-reported bug description, verifies it in the codebase, logs it to `issues.log.md`, and immediately fixes it. Cascading bugs are logged and offered for fixing. |
| `request-feature` | Accepts a user-requested feature or design change, updates `techspec.md` with the new design, logs it to `issues.log.md`, and implements it. Prompts before making broad design changes beyond the feature area. |

## Happy path

### Starting a new project

1. `initialize-project` — scaffold files.
2. Write `project.prompt.md` describing what you want built.
3. `evaluate-prompt` — iterate on the prompt until its clarity score is satisfactory.
4. `execute-prompt` — generate the initial project.
5. `execute-techspec` — capture the as-built state.

### Ongoing maintenance

- `execute-techspec` whenever the codebase has materially changed and you want an up-to-date spec.
- `execute-techreview` to find and fix design-level flaws.
- `execute-debug` to find and fix code-level bugs.
- `report-bug` to verify, log, and fix a specific bug you've encountered.
- `request-feature` to design, log, and implement a specific feature or design change you want.

### Prompt ↔ techspec relationship

`project.prompt.md` is the **historical intent** — the project's starting point. It is not updated as the project evolves.

`techspec.md` is the **as-built specification** — it reflects the project's current, evolved state.

This separation is intentional. Do not reconcile them.

## Shared conventions

- **Issue log format** is defined in `ISSUE_FORMAT.md`. All skills that touch `issues.log.md` follow that schema.
- **Execution policy** — fresh context (via subagents for analysis), autonomous operation, and failure handling — is defined in `EXECUTION_POLICY.md`.
- **Final report format** — the Summary block every skill ends with — is defined in `FINAL_REPORT.md`.
- **Severity levels** (`CRITICAL` / `HIGH` / `MEDIUM` / `LOW` / `SUGGESTION`) are consistent across all skills.
- **Source control is the safety net.** No skill tracks file manifests, previews diffs, or offers rollback. Keep a clean working tree before running file-modifying skills so `git diff` / `git restore` are available.
