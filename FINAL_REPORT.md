# Final Report Template

Every skill ends with a Summary block in this shape:

```
## {skill-name} — Summary

- Issues fixed: {n}
- Issues remaining ACTIVE: {n}
- New issues logged during this run: {n}
- Files changed: {n created, n modified, n removed}

{If any failures occurred, list them with a one-line reason each.}
```

Rules:

- `{skill-name}` is the skill directory name verbatim (e.g., `execute-debug`).
- Always include all four count lines. Use `0` for a true zero; use `N/A (read-only)` or `N/A (no issue log writes)` when the count is structurally inapplicable to the skill.
- Skills MAY add skill-specific lines above the four core counts (e.g., `evaluate-prompt` prints its certainty score and gap list; `execute-techspec` notes the number of sections written).
- If the skill stops early (file not found, user declined, no findings), still produce the Summary with a single sentence above it explaining why it stopped.
