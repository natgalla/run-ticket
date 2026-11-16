---
name: grill-autonomous
description: Autonomous AC interpretation for pipeline use. Identifies ambiguities, routes each to a Skye subagent for resolution, and produces CONTEXT.md without user interaction. BLOCKs only when Skye cannot reach a conclusion from code and standard practice.
argument-hint: "<ticket-body-or-ac>"
model: sonnet
effort: high
---

# Grill Autonomous

Non-interactive AC interpretation for the `/run-ticket` and `/run-epic` pipelines. Produces the same `CONTEXT-<BRANCH>.md` output contract as `/grill-with-docs` but routes all ambiguity resolution to a Skye subagent rather than the user. The user is not consulted.

## Setup

Run `git branch --show-current` to get the branch name (fall back to `default` if empty). The output file is `CONTEXT-<BRANCH>.md` in the working directory root. If it already exists, read it — its resolved entries are settled and must not be re-asked or re-resolved.

## Brief

Write a 3–5 sentence plain-English summary of what is being built: the problem, the user-facing change, and the main moving parts. Label it **What we're building:** and render it as a blockquote. This is for the pipeline log, not for the user.

## Scan

Read the seed and any existing CONTEXT file. Find every ambiguity — vague nouns, implicit scope, conflicting signals, unmeasured criteria, one-way decisions hiding in plain sight.

For each candidate, apply a reasoning pass first. Resolve it immediately (without Skye) if it can be answered from:
- The seed itself
- The existing CONTEXT file
- Codebase reading (files, types, existing conventions)
- Standard engineering practice with no genuine alternative

Log every self-resolved item in an internal inference list with a one-line rationale. Only ambiguities that survive the reasoning pass — those where two or more reasonable conclusions exist and the codebase does not clearly prefer one — proceed to Skye.

## Resolve via Skye

For each surviving ambiguity, spawn a `skye` subagent with the following prompt template. Skye is context-blind — pass only what is in the template. Do not pass session history or prior conversation. See README for Skye configuration.

---

```
You are Skye — an adversarial reviewer operating as a decision-resolver, not a critic. You are context-blind: you know only what is in this prompt.

Your job is to resolve the ambiguity below by reading the codebase and applying product reasoning. Reach a definite conclusion wherever one is justified. Do not hedge when the evidence is clear.

## Resolution criteria

Resolve by inference when any of the following holds:
- The codebase has an established pattern that clearly applies
- Standard engineering practice has a conventional answer and no project-specific reason to deviate
- One option is reversible and the other is not — prefer the reversible one
- One option is consistent with existing conventions and the other introduces a new pattern

BLOCK only when all of the following are true:
- No reasonable conclusion can be reached from code + standard practice
- The decision is non-trivial to reverse
- It requires the user's domain knowledge, business context, or explicit authorization

## Output format

First line must be one of:
  DECISION: <the resolved answer, stated precisely>
  BLOCK: <why no conclusion is reachable without human judgment>

Second line (always present):
  REASONING: <one or two sentences citing the specific evidence or principle that grounds the decision>

Nothing else. No questions. No alternatives unless the reasoning requires naming what was rejected.

## Project directory

{{PROJECT_DIR}}

## Ticket body

{{TICKET_BODY}}

## Current CONTEXT file (resolved entries are settled — do not contradict them)

{{CONTEXT_MD}}

## Ambiguity to resolve

{{QUESTION}}
```

---

Parse Skye's first line:

- `DECISION:` — the ambiguity is resolved. Record the decision and Skye's reasoning in the inference list.
- `BLOCK:` — escalate to the user with the question, the ticket body, and Skye's full output. Stop the pipeline.

## Write CONTEXT.md

After all ambiguities are resolved, write `CONTEXT-<BRANCH>.md`. The file has three sections:

### `## Language`

One entry per term that required definition — either self-resolved or Skye-resolved:

```
**<Term>**:
<One or two sentence definition grounded in the resolution.>
_Avoid_: <synonyms to reject, comma-separated>
```

Only create an entry when a term's meaning was genuinely in question. Do not manufacture glossary entries for terms that were always unambiguous.

### `## Decisions`

One-line reference per ADR created:

```
**<Topic>**: See [ADR-<NNN>](docs/adr/ADR-<NNN>-<kebab-title>.md)
```

Create an ADR only when all three conditions hold: the decision is hard to reverse, it would surprise a future reader without context, and it was a real trade-off between genuine alternatives. Use the same `docs/adr/ADR-<NNN>-<kebab-title>.md` path convention and increment from the highest existing number.

### `## Inference`

Every decision made without asking the user — whether self-resolved or Skye-resolved:

```
**<Topic>**: <Decision reached.> — <one-line rationale citing evidence or principle>
```

This section is the audit trail. It must be complete: every ambiguity from the scan that did not surface to the user appears here.

## Output

Return the absolute path to `CONTEXT-<BRANCH>.md`. Confirm it is ready to pass to the proposal stage.

If any ambiguity resulted in a BLOCK, do not write the CONTEXT file. Surface the block to the user with the full Skye output and halt.
