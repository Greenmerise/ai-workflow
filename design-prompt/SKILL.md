---
name: design-prompt
description: "Break a {name}.prompt.md down into ordered, human-reviewable build phases under design/{name}/. Each phase is a small sub-prompt detailed enough that two implementations would be functionally equivalent, with rationale for any new pattern, technology, or dependency. Does not write code."
disable-model-invocation: true
---

# Design Prompt

Generate a multi-phase design plan from a `{name}.prompt.md` file. Each phase is a small-scope sub-prompt scoped to one human-reviewable concern, with enough detail that running it twice would produce functionally equivalent results.

This skill complements `execute-techspec`: where `techspec.md` documents **what is** in the project, the design plan documents **why** — the reasoning behind the patterns, strategies, technologies, and dependencies chosen for the work the prompt describes.

This skill does NOT implement code. Its only output is design markdown under `design/{name}/`.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: the analysis and design phase (Step 4) is delegated to a fresh subagent per the "Fresh context" rule; the main skill consumes the subagent's structured result and writes the phase files. Autonomous execution applies after the user confirms any folder replacement in Step 2. Failure handling rules apply to Step 5.

## Log format

This skill does not write to `issues.log.md`. The schema in `../ISSUE_FORMAT.md` does not apply here. Use `N/A (no issue log writes)` for the issue-related counts in the final report.

## Input

- A single optional parameter naming the prompt file (e.g., `feature-1.prompt.md` or `prompts/feature-1.prompt.md`), resolved relative to the project root.
- If no parameter is provided, default to `project.prompt.md` in the project root.
- If the resolved file does not exist or is empty, report that to the user and stop with the standard final report.

## Output folder

The output folder is always `design/{basename}/` at the project root, where `{basename}` is the prompt filename with the `.prompt.md` suffix removed and any leading directories stripped:

| Input | Output folder |
|---|---|
| `project.prompt.md` | `design/project/` |
| `feature-1.prompt.md` | `design/feature-1/` |
| `prompts/feature-1.prompt.md` | `design/feature-1/` |

## Step 1: Resolve and validate the prompt file

1. Resolve the prompt file path:
   - If a parameter was provided, use it relative to the project root.
   - Otherwise, use `project.prompt.md`.
2. Verify the file exists and is non-empty. If not, report and stop.
3. Compute the output folder per the table above.

## Step 2: Confirm folder replacement

Check whether the output folder already exists.

- **If it does not exist:** proceed without prompting.
- **If it exists:** use AskUserQuestion to confirm:

  ```
  design/{basename}/ already exists. Replacing it will delete all current files inside.

  Options:
    A) Replace — delete the existing folder and regenerate
    B) Cancel — do not modify anything
  ```

  If the user picks B (or declines), stop with the standard final report. If the user picks A, delete the folder and its contents before continuing.

This is the only confirmation prompt this skill issues. After this point, execution is autonomous per `../EXECUTION_POLICY.md`.

## Step 3: Gather project context

Read the inputs that the design subagent will need:

1. The prompt file in full.
2. `techspec.md` in full if it exists. Note its absence in the brief if it does not.
3. Each existing sibling design under `design/` — read `design/*/phase-*.design.md` to surface prior or parallel design decisions that should inform this design (so phases stay coherent across designs).

The main skill assembles a brief from these inputs to pass to the subagent in Step 4.

## Step 4: Design the phases (delegated subagent)

Delegate to a fresh subagent via the Agent tool. The subagent's job is to survey the codebase and produce the structured phase plan; the main skill writes the files.

### Subagent inputs

- The prompt file content (verbatim).
- The full `techspec.md` content (or a note that it is missing).
- A summary of existing sibling designs under `design/`.

### Subagent directives

- **Survey the codebase** with Glob, Grep, and Read to identify patterns, strategies, technologies, and dependencies already in use that this design should reuse.
- **Follow the structures from `techspec.md`.** Reuse named patterns and components by their existing names so the design integrates rather than parallels.
- **Break the prompt into ordered build phases.** Each phase must be scoped to one human-reviewable concern — small enough that a reviewer can read it without being overwhelmed.
- **Combine related work into a single phase** when the resulting code would be repetitive or share a clear common shape. For example: implementing `PNGImageModule`, `GIFImageModule`, and `JPGImageModule` against the same `ImageModule` base belongs in one phase, because the implementations would be near-identical.
- **Reuse existing patterns, strategies, technologies, and dependencies** wherever they apply. Do not introduce a new one when an existing one fits.
- **Justify anything new.** When a phase introduces a new pattern, strategy, technology, or dependency, the phase must include a human-readable rationale: what it is, why it was chosen over the existing options, what tradeoffs it accepts.
- **Add a refactor phase** when a new pattern, strategy, technology, or dependency could safely replace an existing one. The refactor phase covers migrating existing usages so the codebase ends up consistent.
- **Document why, not what.** Where `techspec.md` describes what exists, design phases describe why this approach was chosen — what was reused, what was introduced, what was rejected.

### Excluded paths during the codebase survey

Do NOT scan files under:

- Dependency directories: `node_modules/`, `vendor/`, `.venv/`, `venv/`, `env/`, `__pycache__/`, `bower_components/`
- Build output: `dist/`, `build/`, `out/`, `target/`, `bin/`, `obj/`, `.next/`, `.nuxt/`, `.svelte-kit/`, `.output/`
- Version control / tooling: `.git/`, `.hg/`, `.svn/`, `.idea/`, `.vscode/`, `.claude/`
- Coverage / caches: `coverage/`, `.nyc_output/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.turbo/`, `.cache/`
- Minified or generated: `*.min.js`, `*.min.css`, `*.map`, `*.generated.*`
- Anything matched by the project's `.gitignore` (read it if present and respect its patterns)

### Subagent return shape

The subagent returns an ordered list of phase objects:

```
[
  { number: 1, summary: "...", body: "..." },
  { number: 2, summary: "...", body: "..." },
  ...
]
```

- `number` — sequential integer starting at 1.
- `summary` — one or two human-readable sentences describing what the phase accomplishes.
- `body` — the full phase contents per Step 5's required structure.

## Step 5: Write the phase files

For each phase returned by the subagent, write:

```
design/{basename}/phase-{number}.design.md
```

Numbers are sequential starting at `phase-1.design.md`. Zero-pad only if there are 10 or more phases (`phase-01.design.md`, ..., `phase-10.design.md`).

### Required phase file structure

Every phase file MUST:

1. **Begin with a `## Summary` section** — one or two plain-language sentences stating what this phase is, in as few sentences as possible. This is what a reviewer reads first.

2. **Be a self-contained sub-prompt.** Running this phase as a prompt twice should produce functionally equivalent results. That requires:
   - **Explicit contracts.** Class, interface, and function signatures with their inputs, outputs, types, return shapes, error behaviors, and side effects. Two implementations should interoperate even if their internal code differs.
   - **All inputs and outputs.** What this phase consumes and produces, including file locations.
   - **Dependencies named.** Which existing modules, services, or libraries this phase relies on.
   - **Reused techspec patterns named explicitly** so the phase ties back to the as-built spec rather than reinventing.

3. **Document the why.** For every new pattern, strategy, technology, or dependency the phase introduces, include a rationale section explaining what was chosen, why it was chosen over the existing options, and what tradeoff it accepts.

4. **NOT contain implementation code.** Function bodies, full class implementations, and complete source files belong in the implementation step, not the design. Pseudocode is acceptable when it is the clearest way to specify required behavior; signatures and contracts are required.

### Replacement, not merge

Step 2 already deleted any prior contents of the folder (or the folder did not exist). Write the full set of phase files fresh. This skill does not merge with prior designs.

### Failure handling

If writing a phase file fails, follow the failure handling rules in `../EXECUTION_POLICY.md`: do not revert, leave partial output in place, continue with remaining phases, and surface the failure in the final report.

## Step 6: Final report

Report per `../FINAL_REPORT.md`. Use `N/A (no issue log writes)` for the three issue-related counts. Add a skill-specific line above the four core counts:

```
- Design phases written: {n}
- Output folder: design/{basename}/
```
