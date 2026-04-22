---
name: initialize-project
description: Initialize a new project by creating starter files (project.prompt.md, CLAUDE.md, techspec.md, issues.log) at the project root.
disable-model-invocation: true
---

# Initialize Project

Scaffolds the four infrastructure files used by this skill suite. Run this once at the start of a new project.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: fresh context on every run (re-check each target file on disk, do not assume what did or didn't exist from prior runs), and failure handling if a file write fails. No analysis phase here, so no subagent is needed.

## CRITICAL: Do not overwrite existing files

For each file below, use Glob to check if it already exists. If it exists, skip it. NEVER overwrite or modify an existing file. Continue to the next file — do not abort the whole run on the first existing file. Collect the list of created-vs-skipped files and present it in the final report.

## Files to create

1. **`project.prompt.md`** — Empty file. This is where the user will write their project generation prompt. It represents the project's original intent and is NOT updated as the project evolves.

2. **`CLAUDE.md`** — Use the template below. The `## Project Workflow Rules (enforced)` section MUST be included verbatim so that the project's skill rules are enforced in every future conversation. The other sections (`Quick Start`, `Problem Statements`, etc.) are placeholder headings only — leave them empty for the user to fill in.

   ```markdown
   # {Project Name}

   ## Quick Start

   <!-- How to build, run, and test the project. Fill in when the project is generated. -->

   ## Problem Statements

   <!-- What problems this project solves. One bullet per problem. -->

   ## Key Design Justifications

   <!-- Non-obvious design decisions and the reasons behind them. -->

   ## Conventions

   <!-- Project-specific naming, structure, or style conventions that aren't captured in techspec.md. -->

   ## Project Workflow Rules (enforced)

   These rules apply to ALL work in this project and override default behavior.

   Shared reference files (`ISSUE_FORMAT.md`, `EXECUTION_POLICY.md`, `FINAL_REPORT.md`) live in the skills folder — either this project's `.claude/skills/` or the user's `~/.claude/skills/`, depending on where the suite is installed. Use the Read tool to locate them; check the project folder first, then the user-global folder.

   ### Issue tracking — `issues.log`
   - Whenever you discover a bug, defect, design flaw, inconsistency, or other issue in the codebase (whether or not you fix it immediately), you MUST log it in `issues.log` at the project root.
   - The format, severity ladder, category values, and dedup rules for `issues.log` are defined in `ISSUE_FORMAT.md` (see location note above). Follow that schema exactly.
   - Do not silently fix issues without logging them. The log is the source of truth for what has been found and addressed.
   - When an issue is resolved, update its status in place (`[ACTIVE]` → `[FIXED]`) rather than deleting the entry.

   ### Tech spec maintenance — `techspec.md`
   - `techspec.md` is the canonical, comprehensive technical specification of this project. It must remain accurate enough that the project could be regenerated from it.
   - Whenever you make a change that affects architecture, public APIs, data models, dependencies, build/run/test workflow, file/module structure, conventions, or any other detail captured in `techspec.md`, you MUST update `techspec.md` in the same change.
   - If `techspec.md` is empty or missing a section relevant to your change, add the section. Do not defer this — an out-of-date techspec is treated as a bug.
   - Purely internal refactors that do not change behavior, structure, or interfaces do not require a techspec update, but err on the side of updating when in doubt.

   ### Prompt vs techspec
   - `project.prompt.md` is the historical starting point of the project. Do not update it to reflect the current state.
   - `techspec.md` is the as-built, evolving specification. Keep it accurate.

   ### Source control is the safety net
   - Skills in this suite operate autonomously after confirmation and do not preview diffs, track file manifests, or offer rollback. Keep a clean working tree before running `execute-*` skills so that `git diff` / `git restore` are available for review and recovery.

   ### Skills to prefer
   - Use `execute-debug` for codebase-wide bug scans; it writes to `issues.log` automatically.
   - Use `report-bug` to report, verify, and fix a specific bug you've encountered.
   - Use `execute-techspec` to rewrite `techspec.md` from the current codebase.
   - Use `request-feature` to design, log, and implement a specific feature or design change.
   - Use `execute-techreview` to find and fix design-level flaws in the techspec and codebase.
   - Use `evaluate-prompt` to assess `project.prompt.md` for clarity before generating.
   - Autonomous execution, failure handling, and fresh-context rules are defined in `EXECUTION_POLICY.md` (see location note at the top of this section).
   ```

3. **`techspec.md`** — Empty file. Comprehensive technical specification of the current project, maintained by `execute-techspec` and `execute-techreview` so that the project could be regenerated from it.

4. **`issues.log`** — Empty file. Tracks bugs and design findings. Schema is defined in `../ISSUE_FORMAT.md`.

## Final report

Report per `../FINAL_REPORT.md`. Use `N/A (no issue log writes)` for issue counts. List each created file and each skipped file with its path, and point the user at `../README.md` for next steps.
