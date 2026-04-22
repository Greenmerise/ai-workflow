---
name: execute-techreview
description: "Review techspec.md for design flaws, gaps, conflicts, and over/under-engineering. Log findings to issues.log, prompt the user to pick what to fix, then apply fixes to both techspec.md and the codebase."
disable-model-invocation: true
---

# Execute Tech Review

Analyze `techspec.md` for design-level issues, log findings to `issues.log`, prompt the user to fix by severity, then apply selected fixes to both `techspec.md` and the codebase.

This skill is the design-level counterpart to `execute-debug`: `execute-debug` fixes code defects, `execute-techreview` fixes design.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: the analysis phase (Steps 1–2) is delegated to a fresh subagent per the "Fresh context" rule; the main skill consumes the subagent's structured findings and performs logging, prompting, and fixing. Autonomous execution applies after the user picks severities in Step 4. Failure handling rules apply to Steps 5–6.

## Log format

Follow the schema in `../ISSUE_FORMAT.md`. Use `CATEGORY: DESIGN` for all findings from the analysis phase, and `## execute-techreview — YYYY-MM-DD` as the section header. Use `CATEGORY: REFACTOR` for incidental continuity changes made while applying a fix.

For the `{location}` field: use `techspec.md#{section}` when a finding is tied to a specific techspec section (e.g., `techspec.md#4.2`), or `-` for cross-cutting concerns. Do NOT point design findings at source files; design issues are about the spec. When location is `-`, lead the description with the concept name so entries remain greppable.

## Analysis input: techspec only

During Steps 1–2, `techspec.md` is the **only** input. The analysis subagent must NOT read source code, `CLAUDE.md`, `project.prompt.md`, or configuration files. It evaluates the techspec purely on what it says and fails to say.

In Step 6, the main skill (not the subagent) uses the updated techspec as the design authority for codebase changes. That's a separate phase in the main skill's context.

## Analysis delegation

Steps 1–2 (locate/read, evaluate) are ONE task delegated to a fresh subagent via the Agent tool. The subagent returns a structured list of findings: `{location, severity, category: DESIGN, description}` per finding. The main skill then performs Steps 3–7 (log, prompt, apply techspec edits, update codebase, report) in its own context.

## Step 1: Locate and read the techspec

Use Glob to find `techspec.md` in the project root. If it does not exist or is empty, stop with the standard final report.

Read the file in full.

## Step 2: Evaluate

Analyze the techspec across the dimensions below. Identify concrete findings — not vague concerns.

If a finding fits multiple dimensions, use the **first matching dimension in the list order below**. The list is ordered narrowest to broadest.

### Conflicts
- Different sections describing the same thing differently.
- Incompatible technology choices (e.g., a library that doesn't support the stated runtime).
- Architectural patterns that conflict with each other.
- Stated conventions contradicting observable patterns in the structure.

### Gaps
- Components mentioned but never described.
- Data flows with unclear entry or exit points.
- Interfaces referenced but not defined.
- Dependencies listed without version constraints or integration details.
- Sections too thin to be actionable for regeneration.

### Underengineering
- Critical paths with no error handling strategy described.
- Security-sensitive areas with no security considerations.
- Data integrity concerns with no validation described.
- Scalability-sensitive areas with no scaling strategy.

### Overengineering
- Abstraction layers with no clear benefit.
- Patterns designed for scale the project doesn't need.
- Redundant mechanisms serving the same purpose.
- Infrastructure exceeding described use cases.

### Design and architectural flaws
- Muddled responsibilities between components.
- Circular dependencies.
- Single points of failure.
- Data model that doesn't support described use cases.
- Patterns inappropriate for the described scale and complexity.

(Design flaws is listed last as a catch-all — anything that's specifically a conflict, gap, over- or under-engineering issue belongs in those buckets first.)

## Step 3: Log findings

Read the existing `issues.log` and apply the dedup rule from `ISSUE_FORMAT.md` to each new finding. Append new entries under `## execute-techreview — YYYY-MM-DD`.

Then report a summary:

```
Found {N} design findings ({n} Critical, {n} High, {n} Medium, {n} Low, {n} Suggestions).
```

If no findings were made, stop here with the standard final report.

## Step 4: Prompt the user

Use AskUserQuestion:

```
Options:
  A) Fix all findings
  B) Fix Critical findings ({n} items)
  C) Fix High findings ({n} items)
  D) Fix Medium findings ({n} items)
  E) Fix Low findings ({n} items)
  F) Fix Suggestions ({n} items)
  G) Don't fix — exit
```

Only include options with matching findings. User may combine options (e.g., `B,C`). If `G`, stop with the standard final report.

## Step 5: Apply fixes to the techspec

For each selected finding, edit `techspec.md` to incorporate the fix.

- Make targeted edits — do not rewrite the entire techspec.
- Preserve the existing structure and formatting.
- If a fix calls for a new section, add it in the appropriate location.
- If a fix calls for removing content, remove it.

### The prescriptive rule

**Techspec edits must be prescriptive, not descriptive.** The techspec is a design authority — it says what the project *should* look like. When a finding identifies a gap, conflict, or flaw:

- **DO** write the techspec as if the design is already correct: describe the intended behavior, the data model, the API, the algorithm. Then in Step 6, implement it.
- **DO NOT** add notes like "this is a placeholder", "not yet implemented", "audio is non-functional", "coverage gaps exist". Those are observations about the current state, not design decisions. They belong in `issues.log`, not `techspec.md`.
- **DO NOT** document a limitation as "acknowledged" and move on. If the finding says persistence is in-memory only, the fix is to specify file-system persistence in the techspec, then implement it in Step 6.
- **Exception: SUGGESTION severity.** For findings at SUGGESTION level, adding a design note to the techspec is acceptable — the finding is advisory, not a defect.

If a finding requires a code change that is too large or risky to implement autonomously, mark it as `(blocked: scope too large for autonomous fix)` per the failure handling rules in `EXECUTION_POLICY.md`. Do NOT downgrade the fix to a descriptive annotation.

## Step 6: Bring the codebase in line with the updated techspec

After all techspec edits are made, update the codebase to match. NO further user prompts — see `EXECUTION_POLICY.md`.

**`techspec.md` dictates what the project SHOULD look like.** All design decisions come from the (now-updated) techspec. Existing code that contradicts the techspec is wrong and must be changed.

### Exclude these paths

When reading or modifying codebase files, do NOT touch files under any of these directories:

- Dependency directories: `node_modules/`, `vendor/`, `.venv/`, `venv/`, `env/`, `__pycache__/`, `bower_components/`
- Build output: `dist/`, `build/`, `out/`, `target/`, `bin/`, `obj/`, `.next/`, `.nuxt/`, `.svelte-kit/`, `.output/`
- Version control / tooling: `.git/`, `.hg/`, `.svn/`, `.idea/`, `.vscode/`, `.claude/`
- Coverage / caches: `coverage/`, `.nyc_output/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.turbo/`, `.cache/`
- Minified or generated: `*.min.js`, `*.min.css`, `*.map`, `*.generated.*`
- Anything matched by the project's `.gitignore` (read it if present and respect its patterns)

### Rules

- **Update references and imports** when files change: imports, includes, type references, index files, barrel exports.
- **Update interfaces and contracts** in both the definition and every consumer.
- **Propagate renames** everywhere an identifier appears.
- **Update dependency manifests** (package.json, requirements.txt, etc.) when changes require new or removed packages.
- **Remove dead code and stale files** when a fix makes them obsolete. Source control is the safety net for unintended deletions.
- **Do not replace files that already match the techspec.** If a file's content already conforms, leave it untouched.
- **Refactor as necessary** to avoid introducing bugs. Log incidental changes with `CATEGORY: REFACTOR`.
- **Produce complete, working output.** No placeholder or TODO comments.
- **Follow the failure handling rules in `EXECUTION_POLICY.md`** if a change breaks the build, types, or tests, or can't be completed.
- **Mark each finding fixed immediately** per `ISSUE_FORMAT.md`: `[ACTIVE]` → `[FIXED]`, update the date.

## Step 7: Final report

Report per `../FINAL_REPORT.md`.
