---
name: polish
description: >-
  Quality loop for worktree-scoped changes (3+ files). Runs tests, review-report,
  applies all findings via a worktree agent, then runs playwright QA. Use this
  after implementation is complete and before pushing — whenever you'd otherwise
  manually chain tests, review, fix, and QA as separate steps. Trigger phrases:
  "polish", "run polish", "do the review loop", "full check before pushing",
  "review and QA".
---

# Polish

Autonomous quality loop: review → apply → repeat until only nits remain → tests → QA. Run after implementation, before push. Intended for worktree-scoped changes (3+ files). For single-file fixes, skip this and push directly.

Do not surface intermediate findings to the user. The loop runs to completion on its own. The only stops are a design fork (a finding that requires a human decision) or a test/QA failure.

## Loop: review → apply (repeat until nits only)

### Review

Spawn the `code-review` agent and, conditionally, `/security-review` in parallel.

The `code-review` agent runs the full review pipeline in a fresh context (no implementation bias) and returns only the consolidated findings list. Spawn it via:

```
Agent(subagent_type: "code-review")
```

**Security pass:** also run `/security-review` when the diff touches any of:
- Auth boundaries (JWT handling, session management, permission checks)
- Cross-user data flows (queries that could leak data between users)
- Service-role or elevated-privilege client usage
- Any new route or data access function returning sensitive data

```
Skill("security-review")   // only when the above conditions are met
```

Merge findings from both into a single consolidated list. Do not pause or present findings to the user. Consume the output and proceed immediately.

### Classify

Review the findings:
- If nothing remains, or only nits remain — exit the loop and proceed to Tests.
- If any finding above nit severity exists, check each one: is it a mechanical fix, or does it require a design decision?
  - **Design fork** — stop and surface the specific decision to the user. Do not apply anything else until it is resolved. After resolution, re-enter the loop.
  - **Mechanical** — proceed to Apply.

### Apply

Spawn a worktree agent to apply all findings — including nits. Pass the full findings list as context.

Spawn with:
- `isolation: "worktree"`
- `model: "sonnet"`
- `subagent_type: "claude"`

Brief the agent:

> A review-report was just completed on this branch. Apply all findings from
> the consolidated list below. Make the changes directly — do not ask for
> confirmation on individual items. Follow the existing code conventions.
> Engineering context is at `~/.claude/engineering-context.md` if it exists.
>
> [paste full findings list here]

After the agent completes, always loop back to Review — even if the previous pass had only medium or low findings. The loop exits only when a full pass returns nits only (no medium or higher findings).

## Tests

Once the review loop exits (nits only), delegate to the **test-runner** agent. Ask it to run the full suite and return a pass/fail result with any failure details.

If tests fail, stop — surface the failures and wait for direction.

## Coverage

After tests pass, check whether the project has coverage configured. Look for, in order: a `coverage` script in `package.json`, a `pytest --cov` config, or a coverage threshold in `jest.config.*`, `vitest.config.*`, or `.nycrc`. If none found, skip this step silently.

If coverage is configured, run it via Bash using the project's own coverage command. Then get the changed line list:

```bash
git diff $(git merge-base HEAD main)...HEAD --unified=0 | grep '^+' | grep -v '^+++' | wc -l
```

Check whether the project's configured threshold is met for the changed files. If coverage falls below the threshold, surface it as a finding — do not hard-stop. Report the shortfall in the completion summary so the user can decide whether to address it before pushing.

## Playwright QA

Required (not optional) when the diff touches any E2E spec file (`tests/e2e/**`). Also run when the change touches user-facing flows, UI components, or anything where visual or interaction correctness matters. Skip only when the change is purely backend, logic-only, and no E2E specs are in the diff.

If running: delegate to the `/verify` skill to exercise the changed functionality in the running app.

```
Skill("verify")
```

## Critic Review

After QA (or after Tests if QA was skipped), run a final critic pass. See README for the critic configuration.

Get the full branch diff:

```bash
git diff $(git merge-base HEAD main)...HEAD
```

Before spawning the critic, collect the following:
- The full branch diff (from the command above)
- The approved plan or ticket AC, if one exists. Check in order: a PLAN.md in the repo root, the proposal file from the propose stage, or the ticket AC extracted during ticket setup. If none is available, note that no plan reference exists.

Spawn the `critic` subagent. The critic is context-blind — pass only what it needs:

> Here is the full diff for this branch. [If a plan exists: Here is the approved plan / ticket AC it was built against.] Ask the questions that surface what's being missed — gaps in the implementation, intent drift from the plan, unchecked assumptions, or design choices that could bite us later.
>
> [paste full diff here]
>
> [paste approved plan / ticket AC here, or note "no plan reference available"]

Collect the critic's output. Do not act on it — it is advisory. Carry it into the completion summary.

## Completion

Surface to the user once, with:
- Number of review passes completed and total findings applied
- Remaining nits (list them — the user can decide if any are worth addressing)
- Test result (pass/fail, count)
- Whether QA ran, and if so, whether it passed or flagged anything
- Any design forks that were surfaced and how they were resolved
- the critic's findings, if any (advisory — the user decides what to act on)

If tests or QA flag a failure, stop — do not push. Surface the failure and wait for direction.
