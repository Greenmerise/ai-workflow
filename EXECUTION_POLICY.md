# Execution Policy

Three rules that apply to every skill in this suite.

## Fresh context

Every skill invocation starts fresh. Analysis phases (surveying code, evaluating a prompt, scanning for bugs, reviewing a techspec) MUST be delegated to a subagent via the Agent tool so they run in a clean context. The main skill then consumes the subagent's structured result.

Fresh context refers to Claude's conversation state — not toolchain caches (ESLint, tsc, etc.) and not rules elsewhere in the skill about which files to read.

Skills with no analysis phase (`initialize-project`, `implement-design`) don't need a subagent; "fresh" for them just means re-reading disk state and not assuming prior-run outcomes.

## Autonomous operation

File-modifying skills (`initialize-project`, `design-prompt`, `implement-design`, `execute-techspec`, `execute-debug`, `execute-techreview`, `report-bug`, `request-feature`) run to completion without further prompts after the user's initial approval. No confirmations, no diff previews, no per-file prompts — source control is the review mechanism.

Exception: if the skill genuinely cannot continue without information that cannot be inferred (missing API key, unavailable external resource), it may ask one focused question. Stylistic choices and "which approach is better" are never grounds to prompt.

## Failure handling

When an autonomous step fails (build break, type error, test failure, dependency install failure, fix that cannot be completed):

1. Do not revert. Leave the partial change in place; source control is the rollback mechanism.
2. Log the failure to `issues.log.md` per `ISSUE_FORMAT.md`: `CATEGORY: BUG, SEVERITY: HIGH` (or `CRITICAL` if the project no longer builds). For a fix that couldn't be completed, leave the originating issue `[ACTIVE]` and append ` (blocked: {short reason})` to its description.
3. Continue with the next item. Surface all failures in the final report.

Skills that don't write to `issues.log.md` still follow rules 1 and 3 — they just have no log to write to.
