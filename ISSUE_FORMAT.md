# Issue Log Format

Shared schema for `issues.log`. All skills that read or write `issues.log` MUST follow this format exactly.

## Line schema

```
[YYYY-MM-DD] [STATUS] [SEVERITY] [CATEGORY] {location} — {description}
```

- **Date:** `YYYY-MM-DD` of the entry's most recent update (creation, edit, or status change).
- **`{location}`:** a short identifier for where the issue lives. The expected form depends on category — see the table below. If nothing meaningful applies, use `-`.
- **`{description}`:** one line, concise. State what is wrong and why. If the location is `-`, lead the description with the thing being described (e.g., section number, concept name) so entries remain greppable.
- **Em-dash separator:** The ` — ` (space, em-dash, space) between `{location}` and `{description}` is the field separator. Descriptions MUST NOT contain ` — ` — use a colon, hyphen, or comma instead. This keeps every line machine-parseable by splitting on the first ` — `.

### Location by category

| Category | Typical location form | Example |
|---|---|---|
| `BUG` | `path/to/file.ext:line` (always file-specific; include the line). | `src/auth/session.ts:42` |
| `DESIGN` | `techspec.md#{section}` or `-` for cross-cutting concerns. Do NOT point at source files; design issues are about the spec. | `techspec.md#4.2` |
| `REFACTOR` | `path/to/file.ext` (line optional; the whole file was touched). | `src/auth/session.ts` |
| `FEATURE` | `path/to/file.ext` if modifying existing code, otherwise `-`. | `-` |
| `DOC` | `path/to/doc.md` or `path/to/file.ext:line` if inline (line required for inline). | `README.md` |

The line number is only meaningful for `BUG` and inline `DOC` entries. Everywhere else, omit it.

## STATUS

| Value | Meaning |
|---|---|
| `ACTIVE` | Open — not yet addressed. |
| `FIXED` | Resolved. Description may be appended with a short note of the fix. |
| `WONTFIX` | Acknowledged but intentionally not fixed. Append a short justification. |

Never remove entries. Status changes are made in place.

## SEVERITY

| Value | Meaning |
|---|---|
| `CRITICAL` | Will cause a crash, data loss, security vulnerability, or makes the project unbuildable/unworkable. Must be fixed. |
| `HIGH` | Causes incorrect behavior in common code paths or significant deviation from intent. Should be fixed. |
| `MEDIUM` | Causes incorrect behavior in edge cases, or leads to suboptimal results. Worth fixing. |
| `LOW` | Minor — unlikely to cause user-visible problems but technically incorrect. Fix if convenient. |
| `SUGGESTION` | Not strictly a defect. An improvement that would strengthen the code or design. |

Use the all-caps form in log lines. Use title-case (**Critical**, **High**, **Medium**, **Low**, **Suggestion**) in prose and headings.

## CATEGORY

| Value | Source skill / meaning |
|---|---|
| `BUG` | Code defect. Logged by `execute-debug` and `report-bug`. |
| `DESIGN` | Techspec/architecture flaw. Logged by `execute-techreview`. |
| `REFACTOR` | Change made for continuity/consistency (not directly from a user-selected finding, but required to keep the codebase coherent). |
| `FEATURE` | Missing capability relative to intent. Logged by `request-feature`. |
| `DOC` | Documentation gap or inaccuracy. |

## Section headers

Group new entries from a single skill run under a dated header using the skill's exact name:

```
## {skill-name} — YYYY-MM-DD
```

Examples:
- `## execute-debug — 2026-04-13`
- `## execute-techreview — 2026-04-13`

The skill name must match the skill directory name verbatim so `issues.log` is mechanically greppable by origin.

Multiple runs on the same day may share a header or use separate ones; do not merge entries from different runs retroactively.

## Deduplication

When a skill is about to write a new entry, check existing entries first:

An existing entry matches a new finding when ALL of the following hold:

1. **Same `CATEGORY`.**
2. **Same specific identifier** — the new finding references the same function, class, file, module, section number, or concept name that the existing entry does. Line numbers may differ; file renames may have occurred. Identify the "specific identifier" as the narrowest named thing both entries reference.
3. **Same underlying problem class** — both describe the same kind of defect (e.g., both describe a null-dereference at that identifier; both describe a missing error-handling strategy for that section). Different problems at the same identifier are NOT matches.

When a judgment call is genuinely uncertain, **err toward matching** — a false match produces one slightly-wrong entry; a false non-match pollutes the log with duplicates that accumulate over runs.

Behavior by match result:

- **Match exists and is `ACTIVE`:** update the existing entry in place. Refresh date, location (line/section may have moved), description, and severity if they've changed. Do NOT create a duplicate.
- **Match exists and is `FIXED` or `WONTFIX`:** leave it untouched and append a new entry. The issue has regressed or a new variant has appeared.
- **No match exists:** append a new entry under the run's section header.

## Writing fixes

When a skill fixes an issue:
- Change `[ACTIVE]` → `[FIXED]` in place.
- Update the date to today.
- Update the location if it shifted (e.g., line moved, file renamed).
- Optionally append ` (fix: {short note})` to the description.

Never delete entries. The log is an append-and-update history.
