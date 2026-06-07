# Import guide — Week3 Standup Bot

Import `standup-workflow.json` into n8n, then do the manual steps below (credentials + placeholders can't be baked into the file).

## 1. Import
n8n → **Workflows → Import from File** → pick `standup-workflow.json`.

> **Node versions matter.** This file targets n8n 2.23.x. The GitHub node has no
> "get commits" operation in this version, so commits are fetched via the HTTP
> Request node against the GitHub REST API. The Slack `message` resource uses
> `search` (not `getAll`) for reading.

## 2. Replace placeholders (search for `REPLACE_` in each node)
| Node | Field | Value |
|------|-------|-------|
| GitHub Commits | URL `REPLACE_OWNER/REPLACE_REPO` | your repo, e.g. `tannp-hcmus/myrepo` |
| GitHub Commits | `Authorization: Bearer REPLACE_GITHUB_TOKEN` | your GitHub PAT (better: use a GitHub credential / Header Auth) |
| Redmine Issues | URL host | your Redmine host |
| Redmine Issues | `X-Redmine-API-Key` header | your Redmine API key (better: move to a Header Auth credential) |
| Slack Read | query `in:REPLACE_SLACK_CHANNEL_NAME` | channel name, e.g. `in:#general` |
| Slack Post | channelId | `#standup-test` channel ID |

## 3. Attach credentials (each node → Credential → select/create)
- **GitHub Commits**: optional — replace inline Bearer token, or attach a Generic Header Auth credential (`Authorization: Bearer <PAT>`).
- **Slack Read**: Slack credential with a **USER token** + `search:read` scope (Slack's `search.messages` API requires a user token, NOT a bot token).
- **Slack Post**: Slack credential (bot token is fine here) with `chat:write`.
- **Gmail**: Gmail OAuth2 (optional bonus).
- **Claude Code**: Claude Code Docker API — Container `claude-code-runner`, identifier `name`, Working Directory `/workspace`.

## 4. Prerequisite (Task 0 — do once)
```bash
docker exec -it claude-code-runner claude login
docker exec claude-code-runner sh -c 'cd /workspace && claude --print "say READY"'   # must NOT say "Not logged in"
```

## 5. Run + verify
- Click **Test workflow** from Manual Trigger → all nodes green → standup posted to `#standup-test`.
- Check the Claude node output panel: if the standup text isn't under `result`, adjust **Slack Post → Text** (`$json.result` / `$json.text` / `$json.output`).

## 6. Memory (Task 9)
After the first successful run, change **Claude Code → Operation** to **Resume Session**, Session ID `standup-tannp`. Add the standalone AI Agent + Window Buffer Memory demo branch for the literal rubric element.

## 7. MCP (Task 10)
Expose via N8N **MCP Server Trigger** as tool `run_standup_workflow` (or wrap the `run-standup` webhook). Then locally:
```bash
claude mcp add --transport http n8n-standup <MCP_SERVER_URL>
claude mcp list
```
In Claude Code CLI: type `run my standup for today`.

## Notes on the file
- `onError: continueRegularOutput` is set on all four sources so an empty/failing source never aborts the run.
- All three triggers (Manual / Schedule 9am / Webhook) fan out to the same four sources.
