---
name: run-ticket
description: Run the full development pipeline for a single ticket autonomously. Grills the AC, proposes, implements, polishes, and opens a PR — surfacing only on BLOCK or completion.
argument-hint: "<ticket-id> [--project path] [--base branch]"
model: sonnet
effort: xhigh
---

# Run Ticket

Autonomous single-ticket pipeline. Runs grill → critic-0 → propose → critic-1 → implement → polish → critic-2 → verify → pr-ready → archive without stopping. Surfaces to the user only on BLOCK or completion.

## Step 0 — Parse arguments

Parse `$ARGUMENTS` for:
- Positional ticket ID (required)
- `--project <path>` — absolute path to the project directory. Can also be an alias key from `~/.claude/project-aliases.json` if you maintain one (see README).
- `--base <branch>`
- `--branch <name>` — if provided, skip branch creation (pre-created by caller)
- `--sidecar <path>` — if provided, write atomic stage updates to this path (used by `/run-epic`)

If no ticket ID is present, stop and ask.

## Step 1 — Resolve project

If `--project` is given: treat it as an absolute path. If it matches a key in `~/.claude/project-aliases.json`, resolve through the alias map. If omitted: check cwd for a `CLAUDE.md`; if absent, ask for the project path and stop until answered. Set `PROJECT_DIR` to the resolved path.

Detect the provider: run `git -C $PROJECT_DIR remote get-url origin`. A URL containing `dev.azure.com` or `visualstudio.com` is ADO; otherwise GitHub.

Resolve `BASE_BRANCH`. If `--base` is given, use it. Otherwise: `git -C $PROJECT_DIR remote show origin | grep 'HEAD branch'`.

## Step 2 — Fetch ticket

GitHub:
```
gh issue view <TICKET_ID> --json number,title,body,url
```

ADO:
```
az boards work-item show --id <TICKET_ID> --org <ORG> -o json
```

Extract: `id`, `title`, `body`, `url`. Extract `ac` by scanning the body for a `## Acceptance Criteria` or `## AC` section or checkbox lines; if none found, use the full body.

## Step 3 — Branch setup

If `--branch` was provided: use it as `BRANCH`. Skip creation.

Otherwise: generate branch name `<id>-<slugified-title>` (lowercase, hyphens, max 60 chars). Then:
```
git -C $PROJECT_DIR fetch origin $BASE_BRANCH
git -C $PROJECT_DIR checkout -b <BRANCH> origin/$BASE_BRANCH
git -C $PROJECT_DIR push -u origin <BRANCH>
git -C $PROJECT_DIR checkout $BASE_BRANCH
```

Assign and record. GitHub: `gh issue develop <TICKET_ID> --name <BRANCH>`. ADO: add a comment with the branch name.

## Step 3c — Claim and track

Assign the ticket to the current user and move it to "In Progress" on the project board.

**GitHub:**

```bash
# Assign
gh issue edit <TICKET_ID> --add-assignee @me

# Resolve org and repo from remote
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
ORG=$(echo $REPO | cut -d/ -f1)
REPO_NAME=$(echo $REPO | cut -d/ -f2)
ISSUE_URL=$(gh issue view <TICKET_ID> --json url -q .url)

# Find projects linked to this repo
PROJECT_NUMBER=$(gh api graphql -f query='
  query($owner: String!, $name: String!) {
    repository(owner: $owner, name: $name) {
      projectsV2(first: 10) { nodes { number title } }
    }
  }' -f owner=$ORG -f name=$REPO_NAME \
  --jq '.data.repository.projectsV2.nodes[0].number')

# Add to project (no-op if already present)
gh project item-add $PROJECT_NUMBER --owner $ORG --url $ISSUE_URL

# Discover Status field ID and "In Progress" option ID
STATUS_FIELD_ID=$(gh project field-list $PROJECT_NUMBER --owner $ORG --format json \
  --jq '.fields[] | select(.name == "Status") | .id')
IN_PROGRESS_OPTION_ID=$(gh project field-list $PROJECT_NUMBER --owner $ORG --format json \
  --jq '.fields[] | select(.name == "Status") | .options[] | select(.name == "In Progress") | .id')

# Get the project item ID for this issue
PROJECT_ID=$(gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    organization(login: $owner) { projectV2(number: $number) { id } }
  }' -f owner=$ORG -F number=$PROJECT_NUMBER \
  --jq '.data.organization.projectV2.id')
ITEM_ID=$(gh project item-list $PROJECT_NUMBER --owner $ORG --format json \
  --jq --arg url "$ISSUE_URL" '.items[] | select(.content.url == $url) | .id')

# Set status
gh project item-edit --id $ITEM_ID --field-id $STATUS_FIELD_ID \
  --project-id $PROJECT_ID --single-select-option-id $IN_PROGRESS_OPTION_ID
```

If `projectsV2` returns more than one project, pick the first whose title contains the repo name; otherwise pick the first result.

**ADO:**

```bash
az boards work-item update --id <ID> --org <ORG> --assigned-to "<CURRENT_USER>"
az boards work-item update --id <ID> --org <ORG> --state "Active"
```

If the "Active" state update fails (some ADO process templates use "In Progress" instead), retry with `--state "In Progress"`.

## Step 3b — Re-entry check

Before running any pipeline stage, determine where to resume. Check in order:

1. If `--sidecar` was provided and the sidecar file exists: read `stage` from it. Resume from the next stage after the last completed one (`stageCompletedAt` is non-null).
2. Otherwise, inspect artifacts on `BRANCH`:
   - OpenSpec change is already archived (`openspec/changes/` has no active proposal for this branch) and PR exists → already at `archive`. Report and stop.
   - Open PR exists (`gh pr view` returns a result) → already at `pr-ready`. Resume from `archive`.
   - Commits exist on `BRANCH` beyond `BASE_BRANCH` → implement is done. Check for polish evidence (review pass commit messages). Resume from `critic-2` if polish is done, `polish` if not.
   - Proposal file exists in the repo (`openspec/changes/*/proposal.md` or equivalent) → propose is done. Resume from `critic-1`.
   - CONTEXT.md exists in the repo root → grill is done. Resume from `critic-0`.
   - None of the above → start from `grill`.

Surface the resume point in one line before proceeding: "Resuming from `<stage>`."

## Step 4 — Pipeline

Run each stage in order. For each stage, if `--sidecar` was provided write stage updates atomically (write to `<sidecar>.tmp`, then `mv`). Fields to update per stage: `stage`, `stageStartedAt`, `stageCompletedAt`, `verdict`, `blocks`.

**Critic invocations.** For each critic stage: read the matching section from `~/.claude/specs/critic-prompts.md` (Critic-0, Critic-1, or Critic-2) — this file is bundled as `critic-prompts.md` in this repo; install it to `~/.claude/specs/` per the README. Fill the template slots. Spawn a `critic` subagent with the filled prompt as its full prompt. The critic is context-blind — pass only the filled prompt. See README for the critic configuration. Parse the critic's first line for `VERDICT: PASS`, `VERDICT: REVISE`, or `VERDICT: BLOCK`. Write the critic's full output to a temp file (`/tmp/critic-<stage>-<TICKET_ID>.md`) immediately after the subagent returns. Hold only the VERDICT line in main context.

**REVISE logic.** On REVISE: increment the revise count for that critic (track locally). Revise the artifact, threading the critic's full prior output into the revision brief. Re-run the critic with `ITERATION_GT_1` true, `ITERATION_N`, and `PRIOR_CRITIC_OUTPUT` set. If the revise count for that critic reaches 2, treat the next REVISE as BLOCK.

**BLOCK action.** On BLOCK (or REVISE after cap): if `--sidecar` was provided, write `stage = "blocked"`, `blocks = [<full the critic output>]` atomically, then return. If standalone, surface to the user with the full the critic output and the blocked artifact. Stop and wait for direction before resuming.

---

### Grill

Spawn a `claude` subagent that executes the `/grill-autonomous` skill on the ticket AC. Pass `PROJECT_DIR`, the ticket body, and the branch name in the subagent prompt. The subagent resolves all ambiguities internally via the critic and returns the path to the completed CONTEXT file. A BLOCK from the critic is the only condition that surfaces to the user — in that case, run the BLOCK action and stop. Do NOT invoke grill-autonomous via the Skill tool; spawning it as an Agent ensures the pipeline continues in this turn after the subagent returns.

<!-- Session history save point. If using context-historian, spawn the historian agent here (mode: SAVE) with ticket ID, branch name, CONTEXT file path, and a 2–3 sentence summary of what grill found. See README. -->

[Save point 1: after grill.]

### Critic-0

Invoke the critic with the Critic-0 prompt. Inputs: `TICKET_BODY` = ticket body, `CONTEXT_MD` = grill output.

On PASS: proceed to propose.
On BLOCK: run BLOCK action.

<!-- Session history save point on BLOCK. See README. -->

### Best Practice

Scan CONTEXT.md for decision-bearing topics — library choices, architectural patterns, data modeling decisions, query strategies, auth approaches, or any phrase that implies a choice between competing approaches.

For each distinct topic identified (max 4 — pick the highest-stakes ones), spawn a `claude` subagent (model: sonnet) to run `/best-practice <topic> --project <PROJECT_DIR>`. Do NOT invoke `/best-practice` via the Skill tool inline — spawn each as an Agent so the pipeline continues in this turn after each subagent returns. If topics are independent (they almost always are), spawn them in parallel in a single message.

Collect all finding blocks and write them to `BEST_PRACTICES.md` in the repo root, prefaced with:

```
# Best Practice Findings
Generated by /best-practice during ticket <TICKET_ID>
```

If no decision-bearing topics are found, skip this stage silently (do not create the file).

Pass the existence of `BEST_PRACTICES.md` into the Propose brief so the proposal incorporates the findings.

### Propose

Spawn a `claude` subagent (model: opus) that executes the `/opsx:propose` skill with the approved CONTEXT file as input. If `BEST_PRACTICES.md` exists in the repo root, include its contents in the subagent brief. The subagent produces all planning artifacts (proposal, spec, design, tasks) and returns the change name and artifact paths. Do NOT invoke opsx:propose via the Skill tool — the skill has a built-in "stop and wait for user" planning gate that ends the turn when invoked inline; spawning it as an Agent keeps that gate contained within the subagent and allows the pipeline to continue after the subagent returns.

> **Note:** `/opsx:propose` is part of the OpenSpec system. If your project does not use OpenSpec, substitute your own proposal-generation skill or create the proposal artifact directly. The pipeline expects a `proposal.md` file on the branch when this stage completes.

### Critic-1

Invoke the critic with the Critic-1 prompt. Inputs: `CONTEXT_MD` = approved AC interpretation, `PROPOSAL` = proposal.

On PASS: proceed to implement.
On BLOCK: run BLOCK action.

<!-- Session history save point on BLOCK. See README. -->

### Implement

Spawn an opus worktree agent (`isolation: "worktree"`, `model: "opus"`, `subagent_type: "claude"`) to implement the approved proposal on `BRANCH`. Brief it with the proposal, the AC, the branch name, and `~/.claude/engineering-context.md` if it exists (see README). Tell it to run `/code-review` at low effort after each task.

After it completes, run the commit verification gate:
```
git -C $PROJECT_DIR log $BASE_BRANCH..$WORKTREE_BRANCH --oneline
```
If empty, merge or cherry-pick from the worktree branch onto `BRANCH` before continuing.

### Polish

Spawn a `claude` subagent to run `/polish` on `BRANCH`. Pass the branch name and `PROJECT_DIR`. The subagent returns only the completion summary (review passes, test result, QA result, any design forks). Do not run `/polish` via the Skill tool inline.

<!-- Session history save point after implement + polish. Include: ticket ID and title, branch name and commit hash, what was built, key decisions from CONTEXT and proposal, test result, review pass result. See README. -->

[Save point 4: after implement + polish.]

### Critic-2

Get the full diff:
```
git -C $PROJECT_DIR diff origin/$BASE_BRANCH...$BRANCH
```

Invoke the critic with the Critic-2 prompt. Inputs: `CONTEXT_MD` = approved AC interpretation, `PROPOSAL` = approved proposal, `DIFF` = full diff.

On PASS: proceed to verify.
On BLOCK: run BLOCK action.

### Verify

Spawn a `claude` subagent to run `/opsx:verify`. The subagent returns pass/fail and any CRITICAL or WARNING findings. Do not run `/opsx:verify` via the Skill tool inline. CRITICAL findings run the BLOCK action. WARNINGs surface to the user before proceeding.

> **Note:** If your project does not use OpenSpec, substitute your own verification step here (manual QA, Playwright, smoke test). The pipeline expects a pass/fail signal before continuing to pr-ready.

### PR-Ready

Spawn a `claude` subagent to run `/ready-for-review`. The subagent returns the PR URL. Do not run `/ready-for-review` via the Skill tool inline.

If `--sidecar` was provided: write `stage = "pr-ready"`, `pr = <URL>` atomically.

<!-- Session history save point after pr-ready. See README. -->

[Save point 5: after pr-ready.]

### Archive

If the repo uses OpenSpec (an `openspec/` directory exists at `PROJECT_DIR`) and an active proposal exists under `openspec/changes/<CHANGE_NAME>/` for this ticket: spawn a `claude` subagent to run `/opsx:archive` for that change. Do not run `/opsx:archive` via the Skill tool inline. The subagent returns the archive path and any follow-up commit it made to fold spec deltas into canonical specs.

If OpenSpec is not in use or no active proposal exists for this ticket, skip silently.

If `--sidecar` was provided: write `stage = "archive"` atomically after the subagent returns.

<!-- Session history save point after archive. Include: ticket ID and title, branch name and PR URL, one-sentence summary of what shipped, OpenSpec change name and archive path (or "no OpenSpec"), key decisions from the full pipeline. See README. -->

[Save point 6: after archive.]

## Step 5 — Completion

Clean up temp files: `rm -f /tmp/critic-*-<TICKET_ID>.md`

Surface to the user: PR URL, ticket number and title, wall time from branch creation to PR open.
