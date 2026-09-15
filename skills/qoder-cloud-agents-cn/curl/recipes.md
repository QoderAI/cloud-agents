# curl Recipes

HTTP mode only: read [setup and transport selection](../shared/guide.md) first. These Managed examples are separate from the MCP tool schemas; consult current operation docs for deployment-specific fields.

Managed HTTP examples: select the recipe for the requested operation, not the whole file. Read [HTTP setup](../shared/guide.md) first. Check HTTP status and business errors before consuming IDs; stop if an ID is empty or null. Cleanup, credential writes and manual scheduled runs are separate authorized actions.

## Setup

```bash
# Required: set these before running any recipe
export BASE="${QCA_HTTP_BASE_URL:-https://api.qoder.com.cn/api/v1/cloud}"
: "${QODER_ACCESS_TOKEN:?Set the token securely in the local environment first}"
```

> Every recipe below also works with a Service Account Token (SAT) — same `Authorization: Bearer` header. For CI / service callers use an SAT instead of a PAT; see `../shared/api-reference.md` § Authentication.

### Windows / Git-Bash note

On Windows Git-Bash (MSYS2 / MINGW), two quirks can corrupt the recipes below. Apply these and they run unchanged:

- **Disable POSIX path conversion.** MSYS rewrites any `/`-leading value passed to a native `.exe` (e.g. `jq.exe`, `curl.exe`) into a Windows path — so a mount path like `/mnt/session/uploads/file_xxx` arrives as `C:/Program Files/Git/mnt/...`. Export `MSYS_NO_PATHCONV=1` (or `MSYS2_ARG_CONV_EXCL='*'`) once per shell to turn this off.
- **Avoid fragile backslash line-continuations.** When a script is pasted with CRLF line endings, a trailing `\` stops continuing the line and the next line (`-F ...`, `--arg ...`) runs as its own command. Prefer single-line commands, and for JSON bodies write the payload to a file and send it with `curl -d @body.json` instead of inlining a `jq -n '{...}'` program as a curl argument.

```bash
# Recommended on Git-Bash:
export MSYS_NO_PATHCONV=1
# Build a JSON body in a file, then post it (robust against quoting/continuation issues):
jq -n --arg a "$AGENT_ID" --arg e "$ENV_ID" --arg f "$FILE_ID" \
  '{agent:$a, environment_id:$e, resources:[{type:"file", file_id:$f}]}' > body.json
curl --fail-with-body -sS -X POST "$BASE/sessions" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" -d @body.json | jq -r '.id'
```

---

## Recipe 1: Verify Connectivity

```bash
curl --fail-with-body -sS "$BASE/agents" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .
```

Expected output (empty account):
```json
{"data": [], "first_id": null, "last_id": null, "has_more": false}
```

If you get 401, your PAT is invalid. If connection timeout, check network.

---

## Recipe 2: Create Environment

> Skip if you already have an environment.

**Option A** — Quick check (prints full response):
```bash
curl --fail-with-body -sS -X POST "$BASE/environments" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-env", "config": {"type": "cloud", "networking": {"type": "unrestricted"}}}' | jq .
```

**Option B** — Save the ID to a variable (use this for subsequent recipes):
```bash
ENV_ID=$(curl --fail-with-body -sS -X POST "$BASE/environments" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-env", "config": {"type": "cloud", "networking": {"type": "unrestricted"}}}' | jq -r '.id')
echo "Environment: $ENV_ID"
```

> Pick **one** of the above — running both creates two environments.

> **Note**: `config` is optional (omit it and you get a default cloud env), but **always pass `networking` explicitly** — when it's absent the env silently defaults to `limited` (no package managers, no MCP egress), which only shows up later as install/network failures. `config: {}` or `config: null` is rejected. Optional extras inside `config`: `packages` (`{"apt": ["curl"]}`) and `setup_script` (idempotent bash, runs at sandbox preparation; non-zero exit aborts session startup).

A new environment is immediately usable (no `status` field to poll) — provisioning happens when a session starts:
```bash
curl --fail-with-body -sS "$BASE/environments/$ENV_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '{id, name}'
```

---

## Recipe 3: Create Agent

```bash
AGENT_ID=$(curl --fail-with-body -sS -X POST "$BASE/agents" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg name "my-agent" \
    --arg system "You are a helpful assistant. Reply concisely." \
    '{name: $name, system: $system, model: "ultimate", tools: [{type: "agent_toolset_20260401"}]}'
  )" | jq -r '.id')
echo "Agent: $AGENT_ID"
```

> Use `jq -n --arg` for safe JSON construction. Never interpolate shell variables directly into JSON strings.

To discover available models and their tuning options (`effort`, `context_window`):
```bash
curl --fail-with-body -sS "$BASE/models" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.data[] | {id, efforts, available_context_windows}'
```

---

## Recipe 4: Create Session + Send Message + Stream

```bash
# Create session
SESS_ID=$(curl --fail-with-body -sS -X POST "$BASE/sessions" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg agent "$AGENT_ID" \
    --arg env "$ENV_ID" \
    '{agent: $agent, environment_id: $env}'
  )" | jq -r '.id')
echo "Session: $SESS_ID"

# Send message — content MUST be an array of content blocks, not a plain string
MESSAGE="Hello! What can you do?"
EVENT_ID=$(curl --fail-with-body -sS -X POST "$BASE/sessions/$SESS_ID/events" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg text "$MESSAGE" \
    '{events: [{type: "user.message", content: [{type: "text", text: $text}]}]}'
  )" | jq -r '.data[0].id')
echo "Event sent: $EVENT_ID"

# Stream response (timeout: use `timeout` on Linux/Git Bash, `gtimeout` on macOS with coreutils, or omit and Ctrl+C manually)
timeout 90 curl --fail-with-body -sS -N "$BASE/sessions/$SESS_ID/events/stream" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Accept: text/event-stream" \
  -H "Last-Event-ID: $EVENT_ID"
```

The stream outputs SSE events. Look for `event: agent.message` for the response and `event: session.status_idle` as a turn boundary; inspect stop_reason (requires_action is paused, not completed).

To render output incrementally, request event deltas (repeatable param, values `agent.message` / `agent.thinking`):
```bash
timeout 90 curl --fail-with-body -sS -N -G "$BASE/sessions/$SESS_ID/events/stream" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Accept: text/event-stream" \
  -H "Last-Event-ID: $EVENT_ID" \
  --data-urlencode "event_deltas[]=agent.message"
```
You'll see `event: event_start` then `event: event_delta` frames (same event ID throughout), followed by the authoritative buffered `agent.message`.

---

## Recipe 5: Multi-Turn Conversation

After Recipe 4, send another message to the same session (wait for idle first):

```bash
MESSAGE2="Now explain that in more detail."
EVENT2_ID=$(curl --fail-with-body -sS -X POST "$BASE/sessions/$SESS_ID/events" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg text "$MESSAGE2" \
    '{events: [{type: "user.message", content: [{type: "text", text: $text}]}]}'
  )" | jq -r '.data[0].id')
echo "Event sent: $EVENT2_ID"

# Stream again — resume after already-received events with the Last-Event-ID header
timeout 60 curl --fail-with-body -sS -N "$BASE/sessions/$SESS_ID/events/stream" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Accept: text/event-stream" \
  -H "Last-Event-ID: $EVENT2_ID"
```

> **Important**: Resume with the `Last-Event-ID` **header** (standard SSE), set to the last SSE `id:` you received. Without it, the stream starts at the current tail and does not replay history. Read historical events through List Events.

---

## Recipe 6: Cancel a Running Session

```bash
# Cancel (acknowledged immediately; safe to call even if already idle)
curl --fail-with-body -sS -X POST "$BASE/sessions/$SESS_ID/cancel" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .

# Wait for idle confirmation via stream or poll:
curl --fail-with-body -sS "$BASE/sessions/$SESS_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.status'
# → "idle" (when settled)
```

---

## Recipe 7: Explicitly Requested Cleanup

Only target the exact resources authorized for cleanup. Pause/archive dependent Deployments first and wait for cancellation to settle before deletion. Do not run this as an automatic final step.

```bash
# 1. Cancel if running (safe to call even if already idle)
curl --fail-with-body -sS -X POST "$BASE/sessions/$SESS_ID/cancel" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .

# Read back state and wait with a deadline until cancellation settles before proceeding.
# 2. Delete session (archive is optional)
curl --fail-with-body -sS -X DELETE "$BASE/sessions/$SESS_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .

# 3. Delete agent
curl --fail-with-body -sS -X DELETE "$BASE/agents/$AGENT_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .

# 4. Archive the test environment. A deleted session may retain its environment reference,
#    which makes DELETE /environments/{id} return 409.
curl --fail-with-body -sS -X POST "$BASE/environments/$ENV_ID/archive" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .

echo "Cleanup complete."
```

> Sessions and agents can be deleted directly without archiving first. Environment deletion is stricter: `DELETE /environments/{id}` succeeds only when no session references it, and deleted sessions may retain that reference. Archive an environment that hosted a session; delete only an unused test environment.

---

## Recipe 8: List & Filter Resources

```bash
# List all agents (non-archived)
curl --fail-with-body -sS "$BASE/agents" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.data[] | {id, name, version}'

# List all sessions for inspection
curl --fail-with-body -sS "$BASE/sessions" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.data[] | {id, status}'

# List environments (environment objects have no status field)
curl --fail-with-body -sS "$BASE/environments" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.data[] | {id, name, networking: .config.networking.type, archived_at}'

# Credits spent by a session (cumulative snapshot, floored to 2 decimals)
curl --fail-with-body -sS "$BASE/sessions/$SESS_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.usage'
# → {"total_credits": 5.94}

# Paginate: pass the previous response's next_page as the page cursor
NEXT=$(curl --fail-with-body -sS "$BASE/agents?limit=20" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq -r '.next_page')
curl --fail-with-body -sS "$BASE/agents?limit=20&page=$NEXT" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .
```

> `page`, `after_id`, and `before_id` are mutually exclusive — sending more than one returns 400.

---

## Recipe 9: Upload File and Attach to a Session

Upload a file once via `POST /files`, then attach it to the session as a `type: "file"` resource. Use this whenever the agent needs to process a non-trivial dataset (CSV, JSON, log, transcript, source code) — pass the data through the filesystem, not through the message body.

```bash
# 1) Upload the file. Response ID field is `id`. No `purpose` field — it no longer exists.
FILE_ID=$(curl --fail-with-body -sS -X POST "$BASE/files" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -F "file=@./feedback.csv" | jq -r '.id')
echo "File: $FILE_ID"

# 2) Create a session with the file in `resources`. Control the path with mount_path
#    (default is /mnt/session/uploads/<file_id>).
SESSION_ID=$(curl --fail-with-body -sS -X POST "$BASE/sessions" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg agent "$AGENT_ID" --arg env "$ENV_ID" --arg fid "$FILE_ID" \
        '{agent: $agent, environment_id: $env, resources: [{type: "file", file_id: $fid, mount_path: "/data/inputs/feedback.csv"}]}')" | jq -r '.id')
echo "Session: $SESSION_ID"

# 3) Tell the agent where to read it (the mount_path you chose).
curl --fail-with-body -sS -X POST "$BASE/sessions/$SESSION_ID/events" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
        '{events:[{type:"user.message",content:[{type:"text",text:"Read /data/inputs/feedback.csv and report the row count excluding the header."}]}]}')"

# Then stream as in Recipe 5 and wait for session.status_idle.
```

You can also attach a file to an existing session: `POST /sessions/{id}/resources` with `{"type":"file","file_id":"...","mount_path":"..."}`.

### Retrieving the agent's output

Agents deliver output files via the built-in `DeliverArtifacts` tool (enabled by default). Watch the stream for **`agent.artifact_delivered`** — it carries the `file_id` directly, plus `original_filename` / `content_type` / `size`:

```bash
# Take the file_id straight from the agent.artifact_delivered event:
ART_ID=$(curl --fail-with-body -sS "$BASE/sessions/$SESSION_ID/events?limit=100&order=desc" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  | jq -r '[.data[] | select(.type=="agent.artifact_delivered")][0].file_id')

# Step 1: get a presigned download URL.
URL=$(curl --fail-with-body -sS "$BASE/files/$ART_ID/content" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq -r '.url')

# Step 2: fetch the file (no Authorization header — signature is in the URL).
curl -sL -o result.csv "$URL"
```

---

## Recipe 10: Memory Store Operations

```bash
# Create store
STORE_ID=$(curl --fail-with-body -sS -X POST "$BASE/memory_stores" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "project-kb", "description": "Project knowledge base"}' | jq -r '.id')
echo "Store: $STORE_ID"   # prefix is memstore_

# Add memory entry — `path` is required and unique within the store
curl --fail-with-body -sS -X POST "$BASE/memory_stores/$STORE_ID/memories" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg content "Production uses PostgreSQL 15 in us-west-2" \
    '{path: "infra/database.md", content: $content, metadata: {category: "infra"}}'
  )" | jq .

# List memories
curl --fail-with-body -sS "$BASE/memory_stores/$STORE_ID/memories" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.data'

# Attach the store to a session so the agent can use it:
#   resources: [{"type": "memory_store", "memory_store_id": "<STORE_ID>", "access": "read_write"}]
```

---

## Recipe 11: Skill Upload + Bind to Agent

```bash
# Create a zip with SKILL.md at root (frontmatter name/description/version wins over form fields)
zip -r my-skill.zip SKILL.md references/

# Upload skill (multipart, max 50 MiB)
SKILL_ID=$(curl --fail-with-body -sS -X POST "$BASE/skills" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -F "file=@./my-skill.zip" | jq -r '.id')
echo "Skill: $SKILL_ID"

# Bind to existing agent — update is POST /agents/{id} (NOT PUT), version required (optimistic lock).
# NOTE: `skills` replaces the whole binding list; merge with existing bindings if any.
AGENT_VERSION=$(curl --fail-with-body -sS "$BASE/agents/$AGENT_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.version')
curl --fail-with-body -sS -X POST "$BASE/agents/$AGENT_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg skill_id "$SKILL_ID" \
    --argjson version "$AGENT_VERSION" \
    '{version: $version, skills: [{type: "custom", skill_id: $skill_id}]}'
  )" | jq '{id, version, skills}'

# Optional, only when deletion is requested; do not delete the Skill just uploaded and bound.
curl --fail-with-body -sS -X DELETE "$BASE/skills/$SKILL_ID" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN"
echo "Skill deleted."
```

> Skills use `multipart/form-data` for upload, NOT JSON. Updating an existing skill is JSON: `PUT /skills/{id}` (optionally `content` + `content_encoding: "base64"` for a new zip).

---

## Recipe 12: Scheduled Deployment (cron)

Run an agent on a schedule — each run creates a session and delivers `initial_events`.

```bash
# Create: daily at 09:00 Asia/Shanghai
DEP_ID=$(curl --fail-with-body -sS -X POST "$BASE/deployments" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg agent "$AGENT_ID" --arg env "$ENV_ID" \
    '{name: "daily-report",
      agent: $agent,
      environment_id: $env,
      schedule: {type: "cron", expression: "0 9 * * *", timezone: "Asia/Shanghai"},
      initial_events: [{type: "user.message", content: [{type: "text", text: "Generate today'"'"'s status report"}]}]}'
  )" | jq -r '.id')
echo "Deployment: $DEP_ID"   # prefix is dep_

# Optional: trigger now only if the user requested an immediate run; creation does not authorize this.
curl --fail-with-body -sS -X POST "$BASE/deployments/$DEP_ID/run" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq .

# Inspect runs; each run points at the session it created
curl --fail-with-body -sS "$BASE/deployments/$DEP_ID/runs" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.data'

# Pause / resume the schedule
curl --fail-with-body -sS -X POST "$BASE/deployments/$DEP_ID/pause" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.status'
curl --fail-with-body -sS -X POST "$BASE/deployments/$DEP_ID/unpause" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" | jq '.status'
```

---

## Recipe 13: Session with a GitHub Repository

Mount a repo so the agent can read/edit code and open PRs (the `gh` CLI ships in the runtime image):

```bash
SESSION_ID=$(curl --fail-with-body -sS -X POST "$BASE/sessions" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg agent "$AGENT_ID" --arg env "$ENV_ID" --arg token "$GITHUB_PAT" \
        '{agent: $agent, environment_id: $env,
          resources: [{type: "github_repository", url: "https://github.com/your-org/your-repo", mount_path: "/app/your-repo", authorization_token: $token}]}')" | jq -r '.id')
```

- `authorization_token` is a GitHub PAT (fine-grained recommended, least privilege). It is never returned on reads.
- Repos cannot be swapped on a running session; container disk is ephemeral — preserve results and commit/push only when authorized.

---

## Recipe 14: Tool Confirmation (always_ask -> allow/deny)

When a built-in tool is configured with `permission_policy: always_ask`, the agent pauses instead of running it: the turn settles at `session.status_idle` with `stop_reason.type = requires_action`, and `stop_reason.event_ids` points at the pending `agent.tool_use`. You resolve it with a `user.tool_confirmation` event.

```bash
# 1) Agent that must ask before running Bash.
AGENT_ID=$(curl --fail-with-body -sS -X POST "$BASE/agents" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d "$(jq -n '{name:"ask-agent", model:"ultimate", system:"Use bash when asked.",
        tools:[{type:"agent_toolset_20260401",
                configs:[{name:"Bash", permission_policy:{type:"always_ask"}}]}]}')" | jq -r '.id')

SESS_ID=$(curl --fail-with-body -sS -X POST "$BASE/sessions" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg a "$AGENT_ID" --arg e "$ENV_ID" '{agent:$a, environment_id:$e}')" | jq -r '.id')

# 2) Ask it to run something. It will pause.
curl --fail-with-body -sS -X POST "$BASE/sessions/$SESS_ID/events" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"events":[{"type":"user.message","content":[{"type":"text","text":"Run in bash: echo hello. Report the output."}]}]}' > /dev/null

# 3) Wait for idle, then read the pending tool_use id from stop_reason.event_ids.
#    (poll GET /sessions/$SESS_ID until status=idle, then:)
EVENTS=$(curl --fail-with-body -sS "$BASE/sessions/$SESS_ID/events?limit=100&order=asc" -H "Authorization: Bearer $QODER_ACCESS_TOKEN")
TOOL_USE_ID=$(echo "$EVENTS" | jq -r '[.data[] | select(.type=="agent.tool_use")][-1].id')
echo "pending tool_use: $TOOL_USE_ID"

# 4a) Only after the required user approval: allow the exact pending tool. Never auto-approve.
curl --fail-with-body -sS -X POST "$BASE/sessions/$SESS_ID/events" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg id "$TOOL_USE_ID" '{events:[{type:"user.tool_confirmation", tool_use_id:$id, result:"allow"}]}')"

# 4b) …or deny it (optional deny_message, allowed only with result=deny):
#   -d "$(jq -n --arg id "$TOOL_USE_ID" '{events:[{type:"user.tool_confirmation", tool_use_id:$id, result:"deny", deny_message:"Not allowed here."}]}')"
```

> Client-side `custom` tools work the same way: the agent emits `agent.custom_tool_use` and pauses at `requires_action`; you run the tool yourself and reply with `user.custom_tool_result` (`custom_tool_use_id` + `content`) instead of `user.tool_confirmation`.

---

## Recipe 15: Authenticated MCP via a Vault

An MCP server that needs a bearer token: put the token in a vault, attach the vault to the session. The agent's `mcp_servers` never carries the token.

```bash
# 1) Vault + static_bearer credential (mcp_server_url must match the agent's MCP url).
VAULT_ID=$(curl --fail-with-body -sS -X POST "$BASE/vaults" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" -d '{"display_name":"mcp-creds"}' | jq -r '.id')

export MCP_TOKEN='<the server token>'
curl --fail-with-body -sS -X POST "$BASE/vaults/$VAULT_ID/credentials" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg u "https://my-mcp.example.com/mcp" --arg t "$MCP_TOKEN" \
        '{auth:{type:"static_bearer", mcp_server_url:$u, token:$t}}')" > /dev/null
unset MCP_TOKEN   # secret is stored server-side and never echoed back

# 2) Agent that references the same MCP server.
AGENT_ID=$(curl --fail-with-body -sS -X POST "$BASE/agents" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n '{name:"mcp-agent", model:"ultimate", system:"Use the MCP tools when asked.",
        tools:[{type:"agent_toolset_20260401"},{type:"mcp_toolset", mcp_server_name:"my-mcp"}],
        mcp_servers:[{name:"my-mcp", type:"url", url:"https://my-mcp.example.com/mcp"}]}')" | jq -r '.id')

# 3) Session with the vault attached — MCP discovery runs here, using the credential.
curl --fail-with-body -sS -X POST "$BASE/sessions" -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg a "$AGENT_ID" --arg e "$ENV_ID" --arg v "$VAULT_ID" \
        '{agent:$a, environment_id:$e, vault_ids:[$v]}')" | jq '{id, vault_ids}'
```

> If session creation fails with an MCP `Unauthorized`/`initialize` error, the vault credential is missing, mismatched (`mcp_server_url` ≠ the agent's url), or the server itself rejected the token. Discovery happens at create time, not at first tool call.
