# run-ticket

Autonomous ticket pipeline for Claude Code. Given a GitHub or Azure DevOps ticket ID, runs the full development cycle — AC interpretation, proposal, implementation, quality loop, and PR — surfacing only on BLOCK or completion.

```
/run-ticket <ticket-id> [--project <path>] [--base <branch>]
/run-epic   <epic-id>   [--project <path>] [--max-parallel N]
```

## Pipeline stages

```
grill → critic-0 → best-practice → propose → critic-1
      → implement → polish → critic-2 → verify → pr-ready → archive
```

| Stage | What happens |
|---|---|
| **grill** | Interprets the AC, resolves ambiguities via Skye, produces `CONTEXT-<branch>.md` |
| **critic-0** | Adversarial review of the AC interpretation |
| **best-practice** | Research pass on decision-bearing topics found in the AC |
| **propose** | Produces a proposal (integrates with OpenSpec if present) |
| **critic-1** | Adversarial review of the proposal |
| **implement** | Opus worktree agent executes the proposal |
| **polish** | Quality loop: review → apply → tests → QA |
| **critic-2** | Adversarial review of the diff |
| **verify** | Manual QA pass on the running app |
| **pr-ready** | Pushes and opens a PR |
| **archive** | Archives the OpenSpec change (skipped if OpenSpec not in use) |

A BLOCK at any stage surfaces to the user and stops. Resume by running the same command — the pipeline detects where it left off and picks up from there.

## Skills included

| Skill | Description |
|---|---|
| `run-ticket` | Single-ticket pipeline |
| `run-epic` | Fan-out orchestrator for multi-ticket epics |
| `grill-autonomous` | Non-interactive AC interpreter, used internally by the pipeline |
| `polish` | Quality loop: review → apply → tests → Playwright QA |
| `best-practice` | Research a decision-bearing topic before committing to an approach |

## Installation

Copy the skills into your Claude Code skills directory:

```bash
cp -r skills/* ~/.claude/skills/
cp critic-prompts.md ~/.claude/specs/critic-prompts.md
mkdir -p ~/.claude/specs
```

## Configuration

### Project aliases (optional)

The `--project` flag accepts an absolute path directly. For multi-project setups, you can also maintain a `~/.claude/project-aliases.json` map:

```json
{
  "my-app": "/path/to/my-app",
  "api": "/path/to/api"
}
```

If `--project` is omitted and no alias map is configured, the pipeline falls back to the current working directory.

### Engineering context (optional)

The implement stage looks for `~/.claude/engineering-context.md` to inject into the opus agent brief. Create this file with any project-wide conventions, tech stack notes, or constraints you want the agent to carry. If the file is absent, the agent uses only the proposal and the project's own `CLAUDE.md`.

### Skye — the adversarial reviewer

The critic stages and `grill-autonomous` spawn a subagent named `skye`. Skye is a context-blind adversarial reviewer — it receives only what the caller passes and has no session history.

Define it in `~/.claude/CLAUDE.md` as a custom agent type, or register it in your harness's agent definitions. The critic prompts are in `critic-prompts.md` — they form the full system prompt. Skye's job is to find gaps, not to agree.

If your harness does not support named agent types, replace `subagent_type: "skye"` with `subagent_type: "claude"` and prepend the relevant critic prompt as a system message.

### Session history (optional)

The pipeline emits save points for the [context-historian](https://github.com/developertown/context-historian) agent. If you use context-historian, the pipeline will spawn it at key milestones (after grill, after implement+polish, after PR open). If you do not, the save point calls are no-ops — comment them out or ignore them.

### OpenSpec (optional)

The propose, verify, and archive stages integrate with [OpenSpec](https://github.com/developertown/openspec) if an `openspec/` directory is present in the project root. If it is absent, the pipeline skips those integrations and proceeds without them.

## Dependencies

- [Claude Code](https://claude.ai/code) CLI
- `gh` CLI (GitHub projects) or `az` CLI (Azure DevOps)
- Git

## Run-epic

`/run-epic` fans out per-ticket pipeline agents in parallel, up to `--max-parallel N` (default 4). It manages a dependency graph, holds tickets that share files with in-flight tickets, and batches BLOCK escalations into single human interruptions.

State is written to `~/.claude/history/<project>/pipeline-state.json`. On restart, the orchestrator reads the state file and resumes from where each ticket left off.

```
/run-epic 12 --project /path/to/repo --max-parallel 3
/run-epic --tickets 42,43,44 --project /path/to/repo
```
