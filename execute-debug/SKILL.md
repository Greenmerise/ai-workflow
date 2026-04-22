---
name: execute-debug
description: "Scan the entire codebase for bugs, log findings to issues.log, and fix them by severity after user approval."
disable-model-invocation: true
---

# Execute Debug

Scan the codebase for bugs, log findings to `issues.log`, prompt the user to choose what to fix, then apply fixes autonomously.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: the analysis phase (Steps 1–3) is delegated to a fresh subagent per the "Fresh context" rule; the main skill consumes the subagent's structured findings and performs logging, prompting, and fixing. Autonomous execution applies after the user picks severities in Step 5. Failure handling rules apply to Step 6.

## Log format

Follow the schema in `../ISSUE_FORMAT.md`. Use `CATEGORY: BUG` for all entries from this skill, and `## execute-debug — YYYY-MM-DD` as the section header. Use `CATEGORY: REFACTOR` for incidental continuity changes made while fixing a bug (see Step 6).

## Analysis delegation

Steps 1–3 (locate, automated checks, manual scan) are ONE task delegated to a fresh subagent via the Agent tool. The subagent follows the instructions in Steps 1–3 and returns a structured list of findings: `{location, severity, category, description}` per finding. The main skill then performs Steps 4–7 (log, prompt, fix, report) in its own context.

## Step 1: Locate sources

Use Glob to identify source, configuration, and build files. Determine the language(s) and framework(s) in use.

### Exclude these paths

Do NOT scan files under any of these directories, and do not log issues about files in them:

- Dependency directories: `node_modules/`, `vendor/`, `.venv/`, `venv/`, `env/`, `__pycache__/`, `bower_components/`
- Build output: `dist/`, `build/`, `out/`, `target/`, `bin/`, `obj/`, `.next/`, `.nuxt/`, `.svelte-kit/`, `.output/`
- Version control / tooling: `.git/`, `.hg/`, `.svn/`, `.idea/`, `.vscode/`, `.claude/`
- Coverage / caches: `coverage/`, `.nyc_output/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.turbo/`, `.cache/`
- Minified or generated: `*.min.js`, `*.min.css`, `*.map`, `*.generated.*`
- Anything matched by the project's `.gitignore` (read it if present and respect its patterns)

For monorepos, scan each workspace's source tree but skip each workspace's own build output and dependency directories.

## Step 2: Run automated checks

Before manual analysis, run any tooling the project already has. Capture all errors and warnings — these feed into Step 3.

- Linter: if a config exists (`.eslintrc*`, `.pylintrc`, `pyproject.toml` with `[tool.ruff]`, `.rubocop.yml`, etc.), run it via Bash.
- Type checker: `tsc --noEmit`, `mypy`, `pyright`, `go vet`, etc.
- Compiler: `go build ./...`, `dotnet build`, `cargo check`, etc.
- Package scripts: if `package.json` has `lint`, `typecheck`, `check`, or `test:types` scripts, run them.

Run every applicable check — do not stop after the first one. If a tool is configured but not installed, note it in the final report's failures list and continue — do NOT log tooling gaps to `issues.log`. `issues.log` is for code defects, not environment problems.

## Step 3: Deep manual scan

Read source files and analyze for the categories below. This is not a surface scan — trace code paths, follow calls, verify contracts.

If a finding fits multiple categories, use the **first matching category in the list order below**. Categories are ordered most-specific to most-general.

### Syntax errors
- Missing or mismatched brackets, braces, parentheses
- Unterminated strings or template literals
- Missing semicolons (where required)
- Malformed expressions, invalid syntax constructs

### Reference errors
- Imports/using statements referencing nonexistent modules or files
- References to variables, functions, classes, or properties that don't exist
- References using old/renamed identifiers that no longer match their target
- Missing index/barrel export entries for new modules
- Broken relative paths

### Type and signature errors
- Function calls with wrong number or types of arguments
- Return type mismatches: signature promises one type, implementation returns another
- Interface/contract violations: missing methods or wrong signatures on a claimed implementation
- Null/undefined access on values that can be null
- Type narrowing gaps
- Missing required fields in struct/object initialization
- Unresolved generics or type parameters

### Integration errors
- API calls with wrong endpoint paths, methods, or payload shapes
- Database queries referencing nonexistent columns or tables
- Environment variables used but never defined or documented
- Event names or message types mismatched between publisher and subscriber

### Return value and data flow errors
- Unchecked error returns (in languages where errors are return values)
- Promises or async calls with missing `await`
- Functions that sometimes return a value and sometimes don't
- Unused return values from functions that return error indicators

### Logical errors
- Inverted boolean conditions
- Missing or incomplete conditional branches
- Off-by-one errors in loops or array access
- Short-circuit evaluation mistakes
- Incorrect operator precedence without explicit grouping
- Dead code behind unconditional returns

(Previously there was a separate "Compile errors" bucket. Compilation-surfaced errors belong in Syntax/Reference/Type above, depending on their nature. The compiler is a detection tool, not a category.)

## Step 4: Log findings

Read the existing `issues.log` and apply the dedup rule from `ISSUE_FORMAT.md` to each new finding. Append new entries under `## execute-debug — YYYY-MM-DD`.

Then report a summary:

```
Found {N} issues ({n} Critical, {n} High, {n} Medium, {n} Low, {n} Suggestions).
```

If no issues were found, stop here with the standard final report.

## Step 5: Prompt the user

Use AskUserQuestion:

```
Options:
  A) Fix all issues
  B) Fix Critical issues ({n} items)
  C) Fix High issues ({n} items)
  D) Fix Medium issues ({n} items)
  E) Fix Low issues ({n} items)
  F) Fix Suggestions ({n} items)
  G) Don't fix — exit
```

Only include options with matching findings. User may combine options (e.g., `B,C`). If `G`, stop with the standard final report.

## Step 6: Fix selected issues

After the user selects, fix every issue in the selected severity level(s) one at a time. NO further user prompts — see `EXECUTION_POLICY.md`.

### Fixing rules

- **Fix the root cause, not symptoms.** If a name is wrong in 5 places, fix the name everywhere.
- **Maintain continuity.** When a fix changes an interface, signature, or export, propagate the change to all consumers.
- **Do not introduce new bugs.** If fixing one issue requires a change elsewhere to keep the code coherent, make that change too. Log it with `CATEGORY: REFACTOR` (e.g., `CATEGORY: REFACTOR` is used for renames, import updates, and consumer updates made to support a bug fix).
- **Do not refactor beyond the fix.** Don't clean up surrounding code.
- **Follow the failure handling rules in `EXECUTION_POLICY.md`** if a fix breaks the build, types, or tests, or can't be completed.
- **Mark each issue fixed immediately** per `ISSUE_FORMAT.md`: `[ACTIVE]` → `[FIXED]`, update the date, adjust the location if it shifted.

## Step 7: Final report

Report per `../FINAL_REPORT.md`.
