# N8N + Claude Code Standup Bot — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an N8N workflow that gathers GitHub/Redmine/Slack/Gmail activity, has the Claude Code CLI node write a Yesterday/Today/Blockers standup, posts it to Slack `#standup-test`, persists memory across runs, and is callable from the local Claude Code CLI as the MCP tool `run_standup_workflow`.

**Architecture:** A single N8N workflow fans a trigger out to four source nodes → a Code node merges/dedupes into one activity string → the Claude Code CLI node (Docker mode, session `standup-tannp`) generates the standup → Slack posts it. A small AI Agent + Window Buffer Memory branch demonstrates the memory node literally. The workflow is exposed over MCP and driven from the local Claude Code CLI.

**Tech Stack:** N8N (browser config), `n8n-nodes-claude-code-cli` (Docker exec into `claude-code-runner`), Slack/GitHub/Gmail nodes, Redmine HTTP API, N8N MCP Server Trigger, local Claude Code CLI MCP config.

**Note on verification:** This plan configures N8N in the browser; there is no unit-test runner. "Verify" steps mean: run the node/workflow and confirm a **green execution** in N8N's execution log with the expected output shape. Each task ends by exporting the workflow JSON and committing it so progress is tracked in git.

**Conventions used throughout:**
- Session ID (fixed): `standup-tannp`
- Slack target channel: `#standup-test`
- Claude container: `claude-code-runner`, working dir `/workspace`
- Workflow JSON export path in repo: `docs/superpowers/artifacts/standup-workflow.json`
- Create the artifacts dir once: `mkdir -p docs/superpowers/artifacts`

---

## Task 0: Prerequisite — authenticate Claude Code in the container

**Why first:** Verified on 2026-06-07 that `claude --print` in the container returns `Not logged in`. Every Claude node fails until this is done. This is an interactive browser step.

- [ ] **Step 1: Confirm containers are up**

Run:
```bash
docker compose -f docker/production/n8n-with-claude-code/docker-compose.yml ps
```
Expected: both `n8n` and `claude-code-runner` show `Up` (claude-code-runner `healthy`).

- [ ] **Step 2: Log in to Claude inside the container (interactive)**

In your terminal, run (the `-it` flags are required for the browser prompt):
```bash
docker exec -it claude-code-runner claude login
```
Follow the browser prompts with your team account. Tip: you can run this from the Claude Code prompt by typing `! docker exec -it claude-code-runner claude login`.

- [ ] **Step 3: Verify login succeeded**

Run:
```bash
docker exec claude-code-runner sh -c 'cd /workspace && claude --print "reply with the single word READY"'
```
Expected: output contains `READY` (NOT `Not logged in`).

- [ ] **Step 4: Confirm session config persisted**

Run:
```bash
docker exec claude-code-runner ls -la /root/.claude
```
Expected: non-empty (credentials/session files present). This persists via the `./claude-config` volume.

---

## Task 1: Create the workflow shell + n8n credentials

**Files:** N8N (browser). Artifact: `docs/superpowers/artifacts/standup-workflow.json`

- [ ] **Step 1: Create the workflow**

In n8n (http://localhost:5678): **Workflows → Add workflow**. Rename it to `Week3 Standup Bot`. Save.

- [ ] **Step 2: Create the Claude Code CLI credential**

Add a Claude Code CLI node temporarily → **Credentials → Create New**:
- Connection Type: **Docker**
- Container Name: `claude-code-runner`
- Container identifier: **name**
- Working Directory: `/workspace`
Save the credential as `claude-docker`.

- [ ] **Step 3: Smoke-test the Claude credential**

On that temporary node: Operation **Execute Prompt**, Prompt `reply with READY`. Click **Test step**.
Expected: green node, output text contains `READY` and a `sessionId` field. If it says `Not logged in`, go back to Task 0.

- [ ] **Step 4: Export + commit the shell**

In n8n: workflow menu → **Download** the JSON. Save it to `docs/superpowers/artifacts/standup-workflow.json`.
```bash
mkdir -p docs/superpowers/artifacts
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): workflow shell + claude docker credential"
```

---

## Task 2: Add triggers (manual + schedule + webhook)

**Files:** N8N nodes on `Week3 Standup Bot`.

- [ ] **Step 1: Add Manual Trigger**

Add **Manual Trigger** node (for test runs). Leave default.

- [ ] **Step 2: Add Schedule Trigger**

Add **Schedule Trigger** node: Trigger Interval = **Days**, at hour **9**, minute **0**. (Daily 09:00.)

- [ ] **Step 3: Add Webhook Trigger (MCP entry point)**

Add **Webhook** node: HTTP Method **POST**, Path `run-standup`, Respond **Using 'Respond to Webhook' node** (or "When last node finishes" for simplicity). This is the entry the MCP tool will call in Task 8.

- [ ] **Step 4: Verify triggers**

Click **Test step** on Manual Trigger → it emits one empty item (green). Save.

- [ ] **Step 5: Export + commit**

Download JSON → overwrite `docs/superpowers/artifacts/standup-workflow.json`.
```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): add manual, schedule, and webhook triggers"
```

---

## Task 3: GitHub source (commits in last 24h)

**Files:** N8N. Required source.

- [ ] **Step 1: Add GitHub credential**

Add **GitHub** node → Credentials → Create New → GitHub API, paste a Personal Access Token (scope: `repo` read). Save as `github-pat`.

- [ ] **Step 2: Configure the node**

GitHub node: Resource **Repository**, Operation **Get Commits** (or use an **HTTP Request** node to `GET /repos/{owner}/{repo}/commits?since={{ $now.minus(24,'hours').toISO() }}`). Set owner/repo to your repo.

- [ ] **Step 3: Wire and test**

Connect Manual Trigger → GitHub node. **Test step**.
Expected: green, output is an array of commit objects (may be empty if no commits in 24h — that is OK, not a failure).

- [ ] **Step 4: Set Continue On Fail**

Open node **Settings** → enable **Continue On Fail** (so an empty/erroring source never aborts the standup).

- [ ] **Step 5: Export + commit**

```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): add GitHub commits source"
```

---

## Task 4: Redmine source (issues assigned to me, updated today)

**Files:** N8N. Required source. Uses HTTP Request (no native Redmine node).

- [ ] **Step 1: Add HTTP Request node**

Add **HTTP Request** node named `Redmine Issues`. Method **GET**.
URL: `https://<your-redmine-host>/issues.json`
Query params:
- `assigned_to_id` = `me`
- `updated_on` = `>={{ $today.toFormat('yyyy-MM-dd') }}`
- `status_id` = `open`

- [ ] **Step 2: Add Redmine auth**

Authentication → **Generic / Header Auth** credential. Header name `X-Redmine-API-Key`, value = your Redmine API key. Save as `redmine-key`.

- [ ] **Step 3: Wire and test**

Connect Manual Trigger → `Redmine Issues`. **Test step**.
Expected: green, JSON body with an `issues` array. If 401 → fix API key; if 404 → fix host/path.

- [ ] **Step 4: Set Continue On Fail**

Node Settings → enable **Continue On Fail**.

- [ ] **Step 5: Export + commit**

```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): add Redmine issues source"
```

---

## Task 5: Slack read source + Gmail source

**Files:** N8N. Slack required (read), Gmail optional bonus.

- [ ] **Step 1: Add Slack credential**

Add **Slack** node → Credentials → Create New. Use a Slack token (OAuth or bot token) with scopes: `channels:history`, `channels:read`, `chat:write` (write needed in Task 7). Save as `slack-bot`.

- [ ] **Step 2: Configure Slack read**

Slack node named `Slack Read`: Resource **Message**, Operation **Get Many** (Search/Get), Channel = your team channel, filter to today (Options → Oldest = `{{ $today.toISO() }}`). Limit ~50.

- [ ] **Step 3: Test Slack read**

Connect Manual Trigger → `Slack Read`. **Test step**.
Expected: green, array of message objects. Enable **Continue On Fail** in Settings.

- [ ] **Step 4: Add Gmail (optional bonus)**

Add **Gmail** node → OAuth2 credential (save as `gmail-oauth`). Operation **Get Many Messages**, filter `after:{{ $today.minus(1,'day').toFormat('yyyy/MM/dd') }}`, limit ~20. Connect Manual Trigger → Gmail. Test → green. Enable **Continue On Fail**.

- [ ] **Step 5: Export + commit**

```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): add Slack read + Gmail sources"
```

---

## Task 6: Code node — merge, dedupe, format activity string

**Files:** N8N Code node. This is the one place with real logic, so the code is given in full.

- [ ] **Step 1: Add a Code node**

Add **Code** node named `Merge & Format`, mode **Run Once for All Items**, language **JavaScript**.

- [ ] **Step 2: Paste this exact code**

```javascript
// Pull each source by node name; tolerate missing/empty sources.
function safe(nodeName) {
  try { return $items(nodeName).map(i => i.json); } catch (e) { return []; }
}

const github  = safe('GitHub');
const redmine = (safe('Redmine Issues')[0]?.issues) || [];
const slack   = safe('Slack Read');
const gmail   = safe('Gmail');

// Normalize to {source, title, url}
const rows = [];
for (const c of github)  rows.push({ source: 'GitHub',  title: (c.commit?.message || c.message || '').split('\n')[0], url: c.html_url || '' });
for (const i of redmine) rows.push({ source: 'Redmine', title: `#${i.id} ${i.subject}`, url: '' });
for (const m of slack)   rows.push({ source: 'Slack',   title: (m.text || '').slice(0, 140), url: '' });
for (const g of gmail)   rows.push({ source: 'Gmail',   title: g.subject || g.snippet || '', url: '' });

// Dedupe by source+title
const seen = new Set();
const deduped = rows.filter(r => {
  const k = r.source + '|' + r.title;
  if (!r.title || seen.has(k)) return false;
  seen.add(k); return true;
});

// Build grouped activity string
const bySource = {};
for (const r of deduped) (bySource[r.source] ||= []).push(r);
let activity = '';
for (const s of ['GitHub', 'Redmine', 'Slack', 'Gmail']) {
  if (bySource[s]?.length) {
    activity += `${s}:\n` + bySource[s].map(r => `- ${r.title}${r.url ? ' (' + r.url + ')' : ''}`).join('\n') + '\n\n';
  }
}
if (!activity) activity = 'No activity found in the last 24h.';

return [{ json: { activity: activity.trim(), count: deduped.length } }];
```

- [ ] **Step 3: Wire all four sources into the Code node**

Connect GitHub, Redmine Issues, Slack Read, and Gmail outputs → `Merge & Format` input.

- [ ] **Step 4: Test the Code node**

**Test step** (run the whole chain from Manual Trigger first so upstream data exists).
Expected: green, output one item `{ activity: "<grouped string>", count: <n> }`.

- [ ] **Step 5: Export + commit**

```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): merge+dedupe+format activity in Code node"
```

---

## Task 7: Claude Code CLI node + Slack post

**Files:** N8N. Core of N8N→Claude direction.

- [ ] **Step 1: Configure the Claude node**

On the Claude Code CLI node (created in Task 1): Credential `claude-docker`. Operation **Execute Prompt**. Add Session ID option = `standup-tannp` if available; otherwise leave for Task 9 to switch to Resume Session.
- System Prompt: `You are a standup assistant.`
- Prompt:
```
Based on this activity, write a standup with exactly three sections — Yesterday, Today, Blockers. Be concise, use bullet points. Activity:
{{ $json.activity }}
```

- [ ] **Step 2: Wire and test**

Connect `Merge & Format` → Claude node. **Test step**.
Expected: green, output text contains `Yesterday` / `Today` / `Blockers`, plus a `sessionId`.

- [ ] **Step 3: Configure Slack post**

Add **Slack** node named `Slack Post`: credential `slack-bot`, Resource **Message**, Operation **Send**, Channel `#standup-test`, Text = `{{ $json.result || $json.text }}` (use whichever field holds Claude's output — check the node's output panel).

- [ ] **Step 4: Wire and test the post**

Connect Claude node → `Slack Post`. **Test step**.
Expected: green, and the standup actually appears in `#standup-test`. Confirm visually in Slack.

- [ ] **Step 5: Export + commit**

```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): generate standup with Claude + post to Slack"
```

---

## Task 8: Full end-to-end run (N8N → Claude)

- [ ] **Step 1: Run the whole workflow from Manual Trigger**

Click **Test workflow**.
Expected: every node green; standup posted to `#standup-test`; data is real (matches your actual commits/issues/messages), not hardcoded.

- [ ] **Step 2: Activate the workflow**

Toggle the workflow **Active** (so Schedule + Webhook triggers are live).

- [ ] **Step 3: Screenshot the canvas**

Take a screenshot of the full canvas (all nodes green) for the PR. Save to `docs/superpowers/artifacts/canvas.png`.

- [ ] **Step 4: Commit artifacts**

```bash
git add docs/superpowers/artifacts/standup-workflow.json docs/superpowers/artifacts/canvas.png
git commit -m "feat(standup): working end-to-end standup pipeline + canvas screenshot"
```

---

## Task 9: Memory — Track A (session resume) + Track B (Window Buffer demo)

**Why two tracks:** The Claude Code CLI node is an action node (`inputs/outputs: ["main"]`) and cannot accept a Window Buffer Memory connection. Track A gives real persistence; Track B satisfies the literal diagram.

- [ ] **Step 1: Track A — switch Claude node to Resume Session**

After the first successful run produced a `sessionId`, change the Claude node Operation to **Resume Session** with Session ID `standup-tannp` (fixed). This carries context across runs.

- [ ] **Step 2: Track A — run twice and verify memory**

Run the workflow once today (run 1). Run it again (run 2) — optionally with a follow-up prompt `What did I report in my previous standup?` via a Chat Trigger or a temporary Execute Prompt.
Expected: on run 2, Claude references content from run 1 (proves session persistence).

- [ ] **Step 3: Track B — add Window Buffer Memory demo branch**

Add an **AI Agent** node (Chat) named `Memory Demo` with a **Window Buffer Memory** sub-node connected via its memory input: Session Key `standup-tannp`, Context Window Length `10`. Give the AI Agent any chat model credential you have. This branch is standalone (driven by a Chat Trigger), purely to show the Window Buffer Memory node on the canvas.

- [ ] **Step 4: Track B — verify the memory node**

Open chat on `Memory Demo`, send "I merged PR #42 today", then in a second message ask "What did I report?".
Expected: the agent recalls PR #42 from Window Buffer Memory.

- [ ] **Step 5: Export + commit**

```bash
git add docs/superpowers/artifacts/standup-workflow.json
git commit -m "feat(standup): session-resume memory + Window Buffer Memory demo"
```

---

## Task 10: Expose as MCP tool + drive from local Claude Code CLI (Claude → N8N)

**Files:** N8N (MCP Server Trigger) + local `.mcp.json`/`claude mcp`.

- [ ] **Step 1: Expose the workflow over MCP**

Easiest path: in n8n add an **MCP Server Trigger** node; it gives you an MCP server URL and lets you register tools. Register the standup workflow (or its Webhook) as tool name **`run_standup_workflow`** with description "Run today's standup and post to Slack". Copy the MCP server URL.

(Alternative if MCP Server Trigger is unavailable: keep the Webhook from Task 2 and wrap it as a tool via an HTTP-based MCP server entry below.)

- [ ] **Step 2: Register the MCP server in local Claude Code CLI**

In your terminal:
```bash
claude mcp add --transport http n8n-standup <MCP_SERVER_URL_FROM_STEP_1>
```
(or add the equivalent entry to your local `.mcp.json`). Verify:
```bash
claude mcp list
```
Expected: `n8n-standup` listed and reachable.

- [ ] **Step 3: Call it from Claude**

In the local Claude Code CLI, type: `run my standup for today`.
Expected: Claude calls `run_standup_workflow` → N8N workflow fires → standup posted to `#standup-test` → Claude replies with a done/confirmation message.

- [ ] **Step 4: Capture evidence**

Screenshot the Claude CLI tool call + the N8N execution log entry for the MCP-triggered run. Save to `docs/superpowers/artifacts/mcp-call.png`.

- [ ] **Step 5: Commit**

```bash
git add docs/superpowers/artifacts/mcp-call.png
git commit -m "feat(standup): expose workflow as run_standup_workflow MCP tool"
```

---

## Task 11: PR with filled template

**Files:** PR description + artifacts.

- [ ] **Step 1: Create branch and push**

```bash
git checkout -b week3-standup-bot
git push -u origin week3-standup-bot
```

- [ ] **Step 2: Open the PR**

Title: `[tannp] Week 3 — N8N + Claude Code Exercise`. Paste the assignment's PR template and fill EVERY section using the real values from this build:
- Task path: Default (checked).
- Integration method: `n8n-nodes-claude-code-cli` production Docker stack (checked); Docker auth completed: Yes (Task 0).
- Data sources: GitHub, Redmine, Slack checked; Gmail bonus checked.
- Vibe coding log: the prompts used to generate the workflow JSON.
- Build Part 1: workflow name `Week3 Standup Bot`, node list, the exact system+user prompt from Task 7, execution = pass, attach `canvas.png`.
- Build Part 2: MCP tool `run_standup_workflow`, test prompt `run my standup for today`, result Yes, Claude's reply.
- Memory verification: Session ID `standup-tannp`, second-run result Yes (Track A) + Window Buffer demo (Track B). **Explicitly explain** the action-node-vs-memory-connection constraint and the two-track resolution.
- Acceptance criteria 1–11: check each.
- Reflection + 60-second share.

- [ ] **Step 3: Attach artifacts**

Attach `canvas.png` and `mcp-call.png` to the PR; the exported `standup-workflow.json` is already in the repo under `docs/superpowers/artifacts/`.

- [ ] **Step 4: Final verification pass**

Re-read the 11 acceptance criteria against what actually works. Anything you could not verify (e.g., a flaky source), state honestly in "What I would still verify manually".

---

## Acceptance criteria → task mapping (self-review)

1. N8N running w/ Claude integration → Task 1, Task 8 (canvas).
2. Workflow executes w/ Claude node (green) → Task 7, Task 8.
3. GitHub commits in prompt → Task 3 + Task 6.
4. Redmine issues in prompt → Task 4 + Task 6.
5. Slack messages in prompt → Task 5 + Task 6.
6. Gmail bonus → Task 5.
7. MCP tool callable → Task 10.
8. Standup from real data → Task 8 Step 1.
9. Posted to Slack channel → Task 7 Step 4.
10. Memory across 2 runs → Task 9.
11. Both directions tested → Task 8 (N8N→Claude) + Task 10 (Claude→N8N).

Prerequisite blocker (Claude login) → Task 0.
