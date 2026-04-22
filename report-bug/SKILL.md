---
name: report-bug
description: "Accept a user-reported bug description, verify the bug in the codebase, log it to issues.log, and immediately fix it. If fixing surfaces additional bugs, log them and offer to fix those too."
disable-model-invocation: true
---

# Report Bug

Accept a user-reported bug, verify it exists in the codebase, log it to `issues.log`, and fix it — all without additional prompting.

This skill is the user-reported counterpart to `execute-debug`: `execute-debug` discovers bugs via automated scanning; `report-bug` investigates a specific bug the user has described.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: the verification phase (Steps 1–2) is delegated to a fresh subagent per the "Fresh context" rule; the main skill consumes the subagent's structured findings and performs logging, fixing, and reporting. Autonomous execution applies immediately after the bug is verified — no severity prompt since the user has already decided this bug needs fixing. Failure handling rules apply to Step 4.

## Log format

Follow the schema in `../ISSUE_FORMAT.md`. Use `CATEGORY: BUG` for the reported issue and any bugs discovered during fixing. Use `## report-bug — YYYY-MM-DD` as the section header. Use `CATEGORY: REFACTOR` for incidental continuity changes made while fixing.

## Input

The user provides a bug description when invoking the skill. This description may be informal — a symptom, an error message, a behavior mismatch, or a reference to a specific file or function. The skill must work with whatever level of detail the user provides.

## Verification delegation

Steps 1–2 (locate and verify) are ONE task delegated to a fresh subagent via the Agent tool. The subagent receives the user's bug description and returns a structured verification result: `{verified: bool, location, severity, description, root_cause, trace}`. The main skill then performs Steps 3–6 (log, fix, handle cascading bugs, report) in its own context.

The subagent prompt must include the user's original bug description verbatim.

## Step 1: Locate the bug

Starting from the user's description, use Glob, Grep, and Read to find the relevant code. Trace the code path that the user's description points to.

### Exclude these paths

Do NOT scan files under any of these directories:

- Dependency directories: `node_modules/`, `vendor/`, `.venv/`, `venv/`, `env/`, `__pycache__/`, `bower_components/`
- Build output: `dist/`, `build/`, `out/`, `target/`, `bin/`, `obj/`, `.next/`, `.nuxt/`, `.svelte-kit/`, `.output/`
- Version control / tooling: `.git/`, `.hg/`, `.svn/`, `.idea/`, `.vscode/`, `.claude/`
- Coverage / caches: `coverage/`, `.nyc_output/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.turbo/`, `.cache/`
- Minified or generated: `*.min.js`, `*.min.css`, `*.map`, `*.generated.*`
- Anything matched by the project's `.gitignore` (read it if present and respect its patterns)

## Step 2: Verify the bug

Confirm that the bug exists by analyzing the code. This is not a surface check — trace the execution path, inspect data flow, and confirm the defect is real.

Return one of:

- **Verified:** The bug exists. Return `{verified: true, location, severity, description, root_cause, trace}` where `trace` is a brief summary of the code path that confirms the defect.
- **Not verified:** The described behavior cannot be confirmed in the code. Return `{verified: false, reason}` explaining why. The main skill will report this to the user and stop.

Assign severity per the `ISSUE_FORMAT.md` ladder based on impact:
- **CRITICAL:** Crash, data loss, security vulnerability, or project won't build.
- **HIGH:** Incorrect behavior in common code paths.
- **MEDIUM:** Incorrect behavior in edge cases.
- **LOW:** Minor — unlikely to cause user-visible problems but technically incorrect.

## Step 3: Log the bug

If the bug is verified:

1. Read the existing `issues.log` and apply the dedup rule from `ISSUE_FORMAT.md`.
2. Append the entry under `## report-bug — YYYY-MM-DD`.
3. Report to the user:

```
Verified: {description}
Location: {location}
Severity: {severity}
Root cause: {root_cause}

Fixing now.
```

If the bug is not verified, report:

```
Could not verify: {reason}
```

Then stop with the standard final report.

## Step 4: Fix the bug

Fix the verified bug immediately. NO user prompt — the user reported it, verification confirmed it, source control is the recovery mechanism.

### Fixing rules

- **Fix the root cause, not symptoms.** If the bug manifests in multiple places, fix the underlying cause.
- **Maintain continuity.** When a fix changes an interface, signature, or export, propagate the change to all consumers.
- **Do not introduce new bugs.** If fixing one issue requires a change elsewhere to keep the code coherent, make that change too. Log it with `CATEGORY: REFACTOR`.
- **Do not refactor beyond the fix.** Don't clean up surrounding code.
- **Follow the failure handling rules in `EXECUTION_POLICY.md`** if a fix breaks the build, types, or tests, or can't be completed.
- **Mark the issue fixed immediately** per `ISSUE_FORMAT.md`: `[ACTIVE]` → `[FIXED]`, update the date, adjust the location if it shifted.

## Step 5: Handle cascading bugs

If fixing the reported bug reveals or creates additional bugs:

1. Log each new bug to `issues.log` under the same `## report-bug — YYYY-MM-DD` section header, with appropriate severity and location.
2. Report the additional bugs to the user:

```
Fixing the reported bug revealed {N} additional issue(s):
{list each with severity and one-line description}

Fix these now? (yes/no)
```

3. Use AskUserQuestion for this prompt.
4. **If yes:** Fix all additional bugs following the same fixing rules (Step 4). Each additional fix may itself reveal more bugs — repeat this step until no new bugs emerge or the user declines.
5. **If no:** Do NOT fix the additional bugs. Leave them `[ACTIVE]` in `issues.log`. The originally reported bug must already be fixed at this point — the user's decline only affects the additional bugs.

If no cascading bugs are found, skip this step entirely.

## Step 6: Final report

Report per `../FINAL_REPORT.md`.
