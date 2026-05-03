---
name: implement-design
description: "Implement a design produced by design-prompt. Accepts either a whole design folder (design/{name}) or a single phase (design/{name}/phase-{n}). Decisions cascade: design file, then techspec.md, then existing codebase, then best judgment — never prompts."
disable-model-invocation: true
---

# Implement Design

Implement a design plan produced by the `design-prompt` skill. Each phase file under `design/{name}/` is a self-contained sub-prompt; this skill turns those sub-prompts into working code.

The design file is the contract and the unit of review. Decisions cascade — design file, then `techspec.md`, then the existing codebase, then best judgment — and the skill never prompts mid-implementation.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. In particular: implementation runs in the main skill (no analysis subagent — the design phase already encodes the analysis); execution is autonomous after the optional design-selection prompt; failure handling rules apply to Step 4.

The design file is the user's confirmation. Source control is the safety net. The user is expected to have a clean working tree before running this skill.

## Log format

This skill does not write to `issues.log.md`. The schema in `../ISSUE_FORMAT.md` does not apply here. Use `N/A (no issue log writes)` for the issue-related counts in the final report.

## Input

A single optional parameter naming what to implement, resolved relative to the project root:

| Input | Meaning |
|---|---|
| _(none)_ | Prompt the user to pick a design from `design/`. |
| `design/{name}` | Implement every phase of that design, in order. |
| `design/{name}/phase-{n}` | Implement a single phase. |
| `design/{name}/phase-{n}.design.md` | Same as above — the `.design.md` suffix is optional. |

Trailing slashes are tolerated. The parameter is treated as a path, not a glob.

## Step 1: Resolve the target

1. If a parameter was provided:
   - Strip any trailing slash.
   - If the path resolves to a directory, treat it as a whole-design target. Collect all `phase-*.design.md` files under it, sorted by phase number (numerically — `phase-2` before `phase-10`).
   - If the path resolves to a file (with or without `.design.md` appended), treat it as a single-phase target.
   - If neither resolves, report the missing path to the user and stop with the standard final report.

2. If no parameter was provided:
   - Use Glob to list `design/*/` directories at the project root.
   - **No designs found:** report that `design/` is empty or missing and stop with the standard final report. Mention that `design-prompt` is the skill that produces designs.
   - **Designs found:** present the list to the user via AskUserQuestion. Use up to 4 design folders as direct options; if more exist, show the most recently modified ones as direct options and rely on the "Other" affordance for the rest. After the user selects, treat the chosen folder as a whole-design target and proceed with Step 2.

3. Validate the chosen target:
   - For a whole-design target, the folder must contain at least one `phase-*.design.md` file.
   - For a single-phase target, the file must exist and be non-empty.
   - On validation failure, report and stop.

This is the only prompt this skill issues. After this point, execution is autonomous per `../EXECUTION_POLICY.md`.

## Step 2: Read the supporting context once

Before implementing any phase, read:

1. Each design phase file selected in Step 1, in full.
2. `techspec.md` at the project root, in full, if it exists. Note its absence in the final report if it is missing — but do not stop.

These are the inputs the implementation will consult. Do NOT pre-survey the codebase here; surveys happen on demand during Step 3 only when the design and techspec do not specify an answer.

## Step 3: Implement each phase in order

For each phase file (one phase if a single-phase target, all phases in order if a whole-design target), implement the phase as described.

### Decision waterfall

For every implementation decision (naming, types, file location, error handling, dependency choice, control flow, anything not nailed down by code alone), consult sources in this order and stop at the first one that answers:

1. **The current phase's design file.** If the phase prescribes it, do exactly that.
2. **`techspec.md`.** If the design is silent, follow the as-built spec.
3. **The existing codebase.** If the techspec is silent, match the patterns, conventions, and dependencies already in use. Reuse before introducing.
4. **Best judgment.** If none of the above answer, pick the option most consistent with the phase, the techspec, and the surrounding code. Do NOT prompt the user. The design file is the confirmation; source control is the rollback.

### Implementation rules

- **Produce complete, working output.** No `TODO`, no placeholder bodies, no "stub for later" comments. If the design names a function, write its full implementation.
- **Honor the phase's contracts exactly.** Class, interface, and function signatures from the phase file are binding — match types, parameter order, return shapes, error behaviors, and side effects.
- **Reuse named techspec patterns.** When the phase or techspec names an existing pattern, component, or module, integrate with it by that name rather than creating a parallel implementation.
- **Update consumers when contracts change.** If a phase modifies an existing interface, propagate the change to every caller, import, index/barrel export, and type reference.
- **Update dependency manifests** (`package.json`, `requirements.txt`, `*.csproj`, `Cargo.toml`, `go.mod`, etc.) when the phase introduces or removes a dependency.
- **Overwrite freely.** When a phase calls for replacing a file, replace it. Source control preserves the prior version.
- **Do not delete unrelated files.** Only remove files the phase explicitly makes obsolete. Stale files outside the phase's scope are the user's concern.
- **Do not refactor beyond the phase.** Don't clean up surrounding code. Don't combine unrelated phases. Stay inside the phase's stated scope.
- **Earlier phases may be assumed complete.** When implementing phase _N_ in a whole-design run, treat the in-tree results of phases _1..N-1_ as authoritative — do not re-read or re-derive their design files.

### Failure handling

Follow the failure handling rules in `../EXECUTION_POLICY.md`. If a phase breaks the build, types, or tests, or cannot be completed:

1. Do not revert the partial change.
2. Continue with the next phase if there is one — a later phase may resolve the breakage, and even if it does not, the partial output is still useful for the user's diff.
3. Surface every failure in the final report with a one-line reason and the phase it occurred in.

## Step 4: Final report

Report per `../FINAL_REPORT.md`. Use `N/A (no issue log writes)` for the three issue-related counts. Add skill-specific lines above the four core counts:

```
- Design implemented: design/{name}/{phase-file or "all phases"}
- Phases implemented: {n} of {total}
- techspec.md consulted: {yes | no — file missing}
```

If the run stopped early (no parameter and no designs available, missing path, empty phase file), include the one-sentence reason above the Summary block per `FINAL_REPORT.md`.
