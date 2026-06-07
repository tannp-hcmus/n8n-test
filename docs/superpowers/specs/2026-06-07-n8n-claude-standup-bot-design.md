# Design — Week 3: N8N + Claude Code Smart Daily Standup Bot

**Date:** 2026-06-07
**Author:** tannp (tannp@zigexn.vn)
**Task path:** Default — Smart Daily Standup Bot (both directions: N8N→Claude and Claude→N8N via MCP)

## Goal

Build an N8N workflow that gathers a developer's activity from GitHub, Redmine,
Slack, and Gmail, asks the Claude Code CLI node to write a standup summary
(Yesterday / Today / Blockers), and posts it to a dedicated Slack test channel.
Then expose that workflow as an MCP tool (`run_standup_workflow`) callable from
the local Claude Code CLI, so a single phrase ("run my standup for today")
triggers the whole pipeline. Memory must persist across at least two runs.

## Verified environment constraints (drive the whole design)

These were checked directly in the current environment on 2026-06-07:

1. **Both containers run and are healthy** — `n8n` (port 5678) and
   `claude-code-runner` are up via the `docker/production/n8n-with-claude-code`
   compose stack. n8n is reachable at http://localhost:5678.

2. **Claude Code is NOT logged in inside the container.** `docker exec
   claude-code-runner claude --print "say OK"` returns `Not logged in · Please
   run /login`, and `/root/.claude` is empty. Every Claude Code CLI node will
   fail until login completes. Login is an interactive (browser) step the user
   must run. **This is prerequisite Step 0.**

3. **The Claude Code CLI node is a regular action node, not an AI Agent
   sub-node.** Source (`nodes/ClaudeCode/ClaudeCode.node.ts`) declares
   `inputs: ["main"]`, `outputs: ["main"]`, `group: ["transform"]`,
   `usableAsTool: true`. It has no `ai_memory` input. Therefore a **Window
   Buffer Memory node cannot be connected to it** — Window Buffer Memory only
   attaches to AI Agent / Chat nodes via the `ai_memory` connection.

4. **The node supports sessions natively.** Operations are `executePrompt`,
   `executeWithContext`, `continueSession`, `resumeSession`; the output
   includes a `sessionId`. This is the real mechanism for cross-run memory.

## Memory approach: two-track (chosen)

Because of constraint 3, we satisfy the rubric honestly with two tracks:

- **Track A — real persistence (primary path):** the standup pipeline uses the
  node's native **Session ID** to carry context across runs. First run uses
  `executePrompt`/`executeWithContext`; subsequent runs use **Resume Session**
  with a fixed Session ID `standup-tannp`. This is what genuinely makes the bot
  "remember yesterday" and is what the "memory persists across 2 runs" criterion
  actually tests.

- **Track B — rubric-literal demo (secondary branch):** a small, separate
  branch on the canvas containing an **AI Agent node + Window Buffer Memory
  node** (`contextWindowLength: 10`, fixed Session ID `standup-tannp`). Its only
  purpose is to make the Window Buffer Memory node visible/working on the canvas
  as the assignment diagram shows, and to be able to answer "What did I report
  yesterday?". The PR will document that the main pipeline uses session-based
  memory because the Claude Code CLI node cannot accept a memory connection.

This gives both: working persistence (Track A) and the literal diagram element
(Track B).

## Architecture

### Part 1 — N8N → Claude (the standup workflow)

```
[Manual/Chat Trigger]   [Schedule Trigger: daily 09:00]   [Webhook Trigger (MCP entry)]
            \                    |                          /
             \___________________|_________________________/
                                 │  (any trigger fans out to the 4 sources)
        ┌────────────┬───────────┼───────────┬──────────────┐
        ▼            ▼           ▼            ▼              ▼
  [GitHub:      [Redmine:     [Slack:      [Gmail:        ]
   commits      issues        read msgs    emails today
   since 24h]   assigned,     today]       (optional)]
                updated today]
        └────────────┴───────────┼───────────┴──────────────┘
                                 ▼
                 [Code node: merge + dedupe + format]
                   → single "activity" string
                                 ▼
                 [Claude Code CLI node]
                   Connection: Docker, container claude-code-runner, cwd /workspace
                   Operation: executePrompt (run 1) / resumeSession (run 2+)
                   Session ID: standup-tannp
                   System: "You are a standup assistant."
                   Prompt: activity → Yesterday / Today / Blockers
                                 ▼
                 [Slack: post message → #standup-test]
                                 │
                                 └──(Track B demo branch)──▶
                                      [AI Agent node] ◀─ai_memory─ [Window Buffer Memory]
                                      (Session ID standup-tannp, last 10)
```

### Part 2 — Claude → N8N (MCP)

```
You (local Claude Code CLI): "run my standup for today"
        │
        ▼
Claude calls MCP tool: run_standup_workflow()
        │
        ▼
N8N MCP endpoint → fires the standup workflow (via Webhook Trigger / MCP Server Trigger)
        │
        ▼
Workflow runs → standup posted to #standup-test
        │
        ▼
Claude replies: "Done. Standup posted to #standup-test ✅"
```

## Components and contracts

| Component | Purpose | Input | Output |
|-----------|---------|-------|--------|
| Triggers | Start the run (manual test, daily 9am, or MCP webhook) | — / cron / HTTP | trigger item |
| GitHub source | Commits in last 24h | repo + token | commit list |
| Redmine source | Issues assigned to me, updated today | base URL + API key (HTTP Request) | issue list |
| Slack source (read) | Today's messages from a channel | Slack creds | message list |
| Gmail source (opt) | Emails received today | Gmail OAuth | email list |
| Code node | Merge all sources, dedupe, build one activity string | 4 source arrays | `{ activity: string }` |
| Claude Code CLI node | Generate standup text | activity string + session | standup markdown + `sessionId` |
| Slack post | Publish standup | standup text | Slack message ts |
| AI Agent + Window Buffer Memory (Track B) | Demonstrate memory node + answer "what did I report yesterday" | chat input | answer referencing prior run |
| MCP exposure | Make workflow callable from Claude | tool call | run confirmation |

## Data flow detail (Code node)

- Normalize each source to `{ source, title, url, timestamp }`.
- Concatenate, dedupe by `url`/`title`.
- Emit a single human-readable `activity` string grouped by source, e.g.:
  `GitHub:\n- <commit msgs>\nRedmine:\n- <issues>\nSlack:\n- <msgs>\nGmail:\n- <subjects>`
- Empty sources degrade gracefully (section omitted, never errors the run).

## Claude Code CLI node config

- Connection mode: **Docker**, Container: `claude-code-runner`, identifier: name,
  Working Directory: `/workspace`.
- Run 1: **Execute Prompt**, capture returned `sessionId`.
- Run 2+: **Resume Session** with Session ID `standup-tannp`.
- System prompt: `You are a standup assistant.`
- User prompt: `Based on this activity, write a standup with three sections —
  Yesterday, Today, Blockers. Activity:\n{{ $json.activity }}`
- Model: Default (inherit container config) or Sonnet for speed.

## Error handling

- Each source node set to **Continue On Fail** so one missing/empty source does
  not abort the standup.
- Claude node: if `Not logged in`, the run fails loudly — Step 0 must be done
  first; PR documents this.
- Slack post: verify message ts in execution log = success.
- MCP: if Claude can't reach the endpoint, fall back to manual trigger and note
  it in the PR (per assignment's fallback guidance).

## Testing / acceptance (Default path rubric)

1. N8N running with Claude integration (screenshot of canvas).
2. Workflow executes with Claude Code CLI node (green execution log).
3. GitHub commits fetched and included in Claude's prompt.
4. Redmine issues fetched and included.
5. Slack messages fetched and included.
6. (Bonus) Gmail also feeds data.
7. MCP tool `run_standup_workflow` configured and callable from local Claude CLI.
8. Standup generated from real data (not hardcoded).
9. Standup posted to #standup-test.
10. Memory persists across ≥2 runs (Track A session resume; Track B Window Buffer demo).
11. Both directions tested: N8N→Claude AND Claude→N8N.

## Deliverable

A PR titled `[tannp] Week 3 — N8N + Claude Code Exercise` using the assignment's
template, with the canvas screenshot, the exported workflow JSON, the vibe-coding
log, and the reflection — including an explicit note on the Window Buffer Memory
vs. action-node constraint and how the two-track memory design resolves it.

## Out of scope (YAGNI)

- Path A / Path B alternatives.
- Production scheduling/monitoring beyond the daily 9am trigger.
- Multi-user sessions (single fixed Session ID `standup-tannp`).
- claude.ai web MCP connection (we use local Claude Code CLI only).
