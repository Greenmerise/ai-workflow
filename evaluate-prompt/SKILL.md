---
name: evaluate-prompt
description: "Assess project.prompt.md for clarity — identifies ambiguity, vague asks, and unclear definitions. Reports a certainty score and gaps by severity."
---

# Evaluate Prompt

Assess `project.prompt.md` for clarity, identifying ambiguity and vague requirements that would leave an AI guessing about what the user actually wants.

## Input

- Operates on `project.prompt.md` in the project root.
- Use Glob to locate it. If the file does not exist or is empty, report that to the user and stop.

## Execution policy

This skill follows `../EXECUTION_POLICY.md`. The analysis is delegated to a fresh subagent via the Agent tool per the "Fresh context" rule; the main skill then formats and presents the subagent's assessment.

## CRITICAL: Read-only skill

Do NOT create, edit, or write any files. Do NOT use Bash with output redirection. This skill only reads and analyzes — its sole output is text presented to the user.

## Clarity, not reproducibility

This skill evaluates whether the prompt clearly communicates what the user wants. It does NOT evaluate whether the output would be identical across regenerations — that is the concern of `execute-techspec` and `execute-techreview`. Do not flag gaps related to file structure consistency, naming stability, or generation determinism.

(The "fresh context" rule in `EXECUTION_POLICY.md` refers to Claude's conversation state — it does NOT mean the evaluation should consider whether the prompt will produce identical outputs.)

## Analysis process

1. **Read the prompt file** in full.
2. **Evaluate whether the prompt clearly communicates what the user wants built.** Consider the following dimensions:

   - **Purpose & Goals** — Is it clear what this project is for and what problem it solves? Or could it be interpreted multiple ways?
   - **Scope & Boundaries** — Is it clear what is in scope and what is out of scope? Or are the edges undefined?
   - **Functional Requirements** — Are the features and behaviors described concretely? Or are they hand-wavy ("make it fast", "handle errors well")?
   - **Technical Requirements** — Are technology choices stated or clearly implied? Or is the AI left to guess the stack?
   - **User Experience** — If there's a user-facing component, is the intended experience described? Or just "build a UI"?
   - **Domain Concepts** — Are domain-specific terms and concepts defined or self-evident? Or does the prompt assume knowledge the AI may not have?
   - **Priorities & Tradeoffs** — When requirements conflict, is it clear which wins? Or are all requirements presented as equal?
   - **Success Criteria** — Is it clear what "done" looks like? Or could the AI build something technically correct but completely wrong?

3. **For each dimension, ask:** "If I gave this prompt to a capable AI with no other context, how confident am I that it would understand what to build?" Flag anything where the answer is less than confident.

4. **Classify each gap** into one of these severity levels:

   | Severity | Meaning |
   |---|---|
   | **Critical** | Fundamentally unclear — the AI would have to guess what the user wants. Multiple valid but wildly different interpretations exist. |
   | **High** | Significantly ambiguous — the AI would make assumptions that have a strong chance of being wrong. |
   | **Medium** | Somewhat vague — the AI could produce reasonable output but might miss the user's actual intent. |
   | **Low** | Minor ambiguity — unlikely to cause a meaningful misunderstanding but could lead to a "that's not quite what I meant" moment. |
   | **Suggestion** | Not ambiguous, but could be stated more clearly or explicitly to reduce interpretation overhead. |

5. **Calculate a certainty score** from 0–100 representing: "How sure am I that I understand what you want?"

   The score falls within a band determined by the worst-severity gap found. Both the band and the cap are **inclusive** on both ends.

   | Worst gap severity | Score band (inclusive) | Label |
   |---|---|---|
   | None, or only Suggestions | 90–100 | **Clear** |
   | Low (no Medium/High/Critical) | 70–89 | **Mostly Clear** |
   | Medium (no High/Critical) | 50–69 | **Unclear** |
   | High (no Critical) | 30–49 | **Vague** |
   | Critical | 0–29 | **Opaque** |

   The band ensures the score and the severity tags stay coherent: a Critical gap cannot yield a "Clear" score. Within a band, apply judgment based on how many gaps exist at that severity and how much of the prompt is affected.

## Output format

Present the assessment as follows:

### Prompt Evaluation: `project.prompt.md`

**Certainty Score: {score}/100 — {label}**

A 1–2 sentence summary of the prompt's clarity.

Then list gaps grouped by severity (only include sections that have gaps):

### Critical
- **[dimension]:** What is ambiguous and what the competing interpretations are.

### High
- **[dimension]:** What is ambiguous and what assumptions the AI would have to make.

### Medium
- **[dimension]:** What is vague and how it could lead to a mismatch.

### Low
- **[dimension]:** What is slightly unclear.

### Suggestions
- How to make the prompt more explicit.

End with a total count: **X gaps found (N Critical, N High, N Medium, N Low, N Suggestions).**

Then produce the final report per `../FINAL_REPORT.md`. Use `N/A (read-only)` for all four counts.
