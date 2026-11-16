---
name: best-practice
description: >-
  Research a decision-bearing topic before committing to an approach.
  Returns a structured Best Practice Finding with conclusion, sources, and
  decision guidance. Use when a design or implementation choice warrants
  grounding in external knowledge before committing to an approach.
argument-hint: "<topic> [--project <path>]"
model: sonnet
effort: medium
---

# Best Practice

Given a topic, return a structured finding that can drive a decision or be threaded into a proposal.

## Step 0 — Parse arguments

Parse `$ARGUMENTS` for:
- Positional topic (required) — a concise phrase describing the decision-bearing question (e.g. "React Query cache invalidation", "Postgres RLS policy ordering", "date-range overlap queries")
- `--project <path>` — if provided, resolve to `PROJECT_DIR` and derive the project name

If no topic is present, ask and stop.

## Step 1 — Bibliography check (optional)

If you maintain a project bibliography at `~/.claude/history/<PROJECT_NAME>/BIBLIOGRAPHY.md` or a global one at `~/.claude/history/bibliography.md`, scan them for entries whose `Subject` or `Conclusion` semantically overlaps the topic.

A hit qualifies if:
- Influence ≥ 2 with a clear, non-empty conclusion, **and**
- Not stale (accessed within 6 months for fast-moving topics — pricing, versioned APIs, framework behavior; no staleness threshold for stable topics — RFCs, database fundamentals, language specs)

**Bib-sufficient condition:** at least one hit from `official-docs` or `academic` type with a clear conclusion, **or** two or more hits of any type with consistent conclusions. If the condition is met, skip Step 2 and proceed directly to Step 3.

If no bibliography is configured, skip this step and proceed directly to Step 2.

## Step 2 — Researcher

Spawn the `researcher` subagent with `model: "haiku"` unless the topic is architectural or security-bearing, in which case use `model: "sonnet"`.

Brief:

> Research the following topic for use in a software implementation decision: **{{TOPIC}}**
>
> Return:
> - The current best practice or recommended approach, with a one-sentence conclusion
> - The primary authoritative source (official docs, RFC, or peer-reviewed paper preferred)
> - One or two secondary sources if they materially add to the conclusion
> - Any known failure modes, caveats, or version-specific behavior
>
> Emit CITE tags for every source used (slug, url, accessed date).

Collect the researcher's output. Extract all CITE tags and conclusions.

## Step 3 — Format finding

Produce a **Best Practice Finding** block:

```
## Best Practice: <topic>

**Conclusion:** <one sentence — the recommended approach>

**Guidance:** <one to three sentences — what this means for the current implementation; specific enough to act on>

**Caveats:** <any failure modes, version constraints, or context-specific exceptions; omit section if none>

**Sources:**
- [<slug>](<url>) — <type> — <accessed> — <one-sentence conclusion from that source>
```

## Step 4 — Output

Return the formatted finding block. If called standalone (not from another skill), also print a one-line summary: what the conclusion is and whether it came from the bibliography or fresh research.

If called from `/run-ticket` or `/run-epic`: return the finding block only (no summary line). The caller appends it to `BEST_PRACTICES.md`.
