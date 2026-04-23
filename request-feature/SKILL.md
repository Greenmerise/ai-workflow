---
name: request-feature
description: "Accept a user-requested feature or design change, analyze its impact, update techspec.md with the new design, log it to issues.log.md, and implement it. If the change cascades beyond the isolated feature area, prompt before proceeding."
disable-model-invocation: true
---

# Request Feature

Accept a user-requested feature or design change, analyze its feasibility and impact, update `techspec.md` to reflect the new design, log it to `issues.log.md`, and implement it — all without additional prompting unless the change requires a broader refactor.

This skill is the user-requested counterpart to `execute-techreview`: `execute-techreview` discovers design flaws via automated analysis; `request-feature` implements a specific design change the user has described. The same relationship holds as between `report-bug` and `execute-debug` — one is user-initiated and targeted, the other is automated and comprehensive.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: the analysis phase (Steps 1–2) is delegated to a fresh subagent per the "Fresh context" rule; the main skill consumes the subagent's structured result and performs logging, techspec editing, implementation, and reporting. Autonomous execution applies immediately after analysis — no severity prompt since the user has already decided this feature is wanted. Failure handling rules apply to Steps 5–6.

**Scope exception:** If the analysis reveals that implementing the feature requires design changes beyond the isolated feature area (e.g., restructuring shared interfaces, modifying core data models used by multiple components, changing architectural patterns), the skill MUST prompt the user before proceeding with those broader changes. The feature itself is always implemented; the prompt is only about cascading design changes.

## Log format

Follow the schema in `../ISSUE_FORMAT.md`. Use `CATEGORY: FEATURE` for the requested feature entry. Use `CATEGORY: DESIGN` for cascading design changes surfaced during analysis. Use `CATEGORY: BUG` for bugs discovered during implementation. Use `CATEGORY: REFACTOR` for incidental continuity changes made while implementing. Use `## request-feature — YYYY-MM-DD` as the section header.

Each cascading issue — whether a design change, bug, or refactor — MUST be its own atomic entry in `issues.log.md` so it can be independently tracked.

## Input

The user provides a feature description when invoking the skill. This description may range from a high-level capability request to a specific design change with implementation details. The skill must work with whatever level of detail the user provides.

## Analysis delegation

Steps 1–2 (analyze feasibility and assess impact) are ONE task delegated to a fresh subagent via the Agent tool. The subagent receives the user's feature description and returns a structured analysis result. The main skill then performs Steps 3–7 (log, update techspec, implement, handle cascading issues, report) in its own context.

The subagent prompt must include the user's original feature description verbatim.

## Step 1: Analyze the feature request

Starting from the user's description:

1. Read `techspec.md` in full to understand the current design.
2. Use Glob, Grep, and Read to explore the relevant areas of the codebase.
3. Determine what the feature requires:
   - What new components, interfaces, data structures, or behaviors are needed?
   - What existing components need modification?
   - Where does this feature fit in the current architecture?

### Exclude these paths

Do NOT scan files under any of these directories:

- Dependency directories: `node_modules/`, `vendor/`, `.venv/`, `venv/`, `env/`, `__pycache__/`, `bower_components/`
- Build output: `dist/`, `build/`, `out/`, `target/`, `bin/`, `obj/`, `.next/`, `.nuxt/`, `.svelte-kit/`, `.output/`
- Version control / tooling: `.git/`, `.hg/`, `.svn/`, `.idea/`, `.vscode/`, `.claude/`
- Coverage / caches: `coverage/`, `.nyc_output/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.turbo/`, `.cache/`
- Minified or generated: `*.min.js`, `*.min.css`, `*.map`, `*.generated.*`
- Anything matched by the project's `.gitignore` (read it if present and respect its patterns)

## Step 2: Assess impact and feasibility

Evaluate the feature against the current design:

### Isolated change

The feature can be implemented by:
- Adding new files/components without modifying shared interfaces.
- Modifying only the files directly related to the feature.
- Extending existing interfaces without breaking consumers.

If the change is isolated, return: `{feasible: true, isolated: true, techspec_sections: [...], new_sections: [...], affected_files: [...], description, implementation_plan}`

### Broad change

The feature requires:
- Restructuring shared interfaces or contracts used by multiple components.
- Modifying core data models that other features depend on.
- Changing architectural patterns or cross-cutting concerns.
- Significant changes to components unrelated to the feature itself.

If the change is broad, return: `{feasible: true, isolated: false, techspec_sections: [...], new_sections: [...], affected_files: [...], cascading_changes: [{area, description, reason}], description, implementation_plan}`

### Not feasible

The feature fundamentally conflicts with the current architecture in a way that cannot be resolved without a full redesign, or the description is too ambiguous to act on.

Return: `{feasible: false, reason}` — the main skill will report this to the user and stop.

## Step 3: Log the feature request

1. Read the existing `issues.log.md` and apply the dedup rule from `ISSUE_FORMAT.md`.
2. Log the feature entry with `CATEGORY: FEATURE`, `STATUS: ACTIVE`, and severity per the `ISSUE_FORMAT.md` ladder based on the feature's impact on the project (most features are `MEDIUM`; use `HIGH` if it changes core behavior or public interfaces; use `LOW` for cosmetic or minor additions).
3. If the analysis identified cascading design changes, log each as a separate atomic entry with `CATEGORY: DESIGN`, `STATUS: ACTIVE`.
4. Report to the user:

For isolated changes:
```
Feature: {description}
Impact: Isolated — {n} file(s) affected
Techspec sections to update: {list}

Implementing now.
```

For broad changes:
```
Feature: {description}
Impact: Broad — requires changes beyond the feature area:
{list each cascading change with area and reason}

The feature itself will be implemented. Proceed with the broader design changes? (yes/no)
```

5. For broad changes, use AskUserQuestion for the cascading-changes prompt.
   - **If yes:** Proceed with all changes (feature + cascading design changes).
   - **If no:** Implement only the feature itself in isolation. Leave cascading design issues `[ACTIVE]` in `issues.log.md` for future resolution. If implementing the feature in isolation would produce broken or incoherent code, log the feature as `(blocked: cascading changes declined)` per `EXECUTION_POLICY.md` failure handling and stop with the final report.

## Step 4: Update the techspec

Edit `techspec.md` to incorporate the new feature's design. This step happens BEFORE implementation — the techspec is the design authority.

### Techspec editing rules

- **Make targeted edits** — do not rewrite the entire techspec.
- **Preserve existing structure and formatting.**
- **Add new sections** where the feature introduces new components, interfaces, or behaviors that don't fit existing sections.
- **Update existing sections** where the feature modifies current design.
- **Be prescriptive, not descriptive.** The techspec says what the project *should* look like. Write the feature's design as if it is already correct and implemented. Do NOT add notes like "to be implemented" or "planned".
- **Every implementation detail must be reflected.** The techspec must fully describe the feature — if it's not in the techspec, it shouldn't be in the code. If it's in the code, it must be in the techspec.
- **If the user approved cascading changes (Step 3),** update those sections too.
- **If the user declined cascading changes,** only update sections directly related to the feature.

## Step 5: Implement the feature

After the techspec is updated, implement the feature in the codebase. `techspec.md` (now updated) dictates what the project should look like. NO further user prompts — see `EXECUTION_POLICY.md`.

### Implementation rules

- **The updated techspec is the design authority.** All implementation decisions come from the techspec. Code that contradicts the techspec is wrong.
- **Produce complete, working output.** No placeholder or TODO comments.
- **Update references and imports** when files change: imports, includes, type references, index files, barrel exports.
- **Update interfaces and contracts** in both the definition and every consumer.
- **Propagate renames** everywhere an identifier appears.
- **Update dependency manifests** (package.json, requirements.txt, etc.) when changes require new or removed packages.
- **Remove dead code and stale files** when the feature makes them obsolete. Source control is the safety net.
- **Do not replace files that already match the techspec.** If a file's content already conforms, leave it untouched.
- **Do not refactor beyond the feature.** Don't clean up surrounding code.
- **Follow the failure handling rules in `EXECUTION_POLICY.md`** if a change breaks the build, types, or tests, or can't be completed.

## Step 6: Handle cascading issues

If implementing the feature reveals or creates additional issues (bugs, design inconsistencies, or necessary refactors):

1. Log each issue as its own **atomic entry** in `issues.log.md` under the same `## request-feature — YYYY-MM-DD` section header, with the appropriate category (`BUG`, `DESIGN`, or `REFACTOR`), severity, and location.

2. For bugs discovered during implementation:
   - Fix them immediately following the same implementation rules (Step 5).
   - Mark them `[FIXED]` in `issues.log.md`.
   - Each fix may reveal more issues — repeat until stable.

3. For design issues discovered during implementation that were NOT part of the original analysis:
   - If isolated to the feature area: fix them (update techspec + code), mark `[FIXED]`.
   - If they extend beyond the feature area: leave them `[ACTIVE]` in `issues.log.md`. Report them to the user but do NOT prompt — these are tracked for future resolution.

4. For refactors needed to maintain continuity:
   - Apply them immediately. Log with `CATEGORY: REFACTOR` and mark `[FIXED]`.

## Step 7: Mark the feature complete

After implementation:
1. Update the feature's `issues.log.md` entry: `[ACTIVE]` → `[FIXED]`, update the date.
2. Optionally append `(fix: implemented per techspec)` to the description.
3. Verify the techspec fully reflects the implementation. If any implementation detail is missing from the techspec, add it now.

## Step 8: Final report

Report per `../FINAL_REPORT.md`.
