# Tools and Resources

HTTP mode only: read [setup and transport selection](guide.md) first. These Managed examples are separate from the MCP tool schemas; consult current operation docs for deployment-specific fields.

How to configure agent tools, manage environments, and work with files, repos, memory, and scheduled deployments.

---

## Agent Tools

Tools are configured on the Agent object (not the session). `tools[]` is a union discriminated by `type`: `agent_toolset_20260401`, `browser_toolset_20260714`, `mcp_toolset`, `custom`.

### agent_toolset_20260401 (Built-in Tools)

```json
{
  "name": "my-agent",
  "system": "You are a coding assistant.",
  "model": "ultimate",
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "enabled_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep", "WebFetch", "WebSearch", "DeliverArtifacts"]
    }
  ]
}
```

Built-in tool names (exact values, also used in event streams): `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebFetch`, `WebSearch`, `ImageGen`, `ImageSearch`, `DeliverArtifacts`.

Semantics that bite:
- `enabled_tools` omitted or `[]` → **all** built-in tools enabled (including `DeliverArtifacts`).
- Non-empty `enabled_tools` → strict allowlist. Want artifact delivery with a custom list? Include `DeliverArtifacts` explicitly.
- Unknown tool names return 400 (`"unknown tool name 'Foo'"`).
- `disallowed_tools` hides/denies specific tools; a tool cannot appear in both lists.
- Omitting the whole `tools` field or `[]` → the model gets **no tools at all**.
- Per-tool permissions go in `configs`: `{"name": "Bash", "permission_policy": {"type": "always_ask"}}`. Policies: `always_allow`, `always_ask` (pauses for a `user.tool_confirmation` event), `always_deny`.
- The old per-tool-object schema (e.g. `{"type": "bash_20250124"}`) is no longer supported.

### Browser Use (Beta)

Browser Use ships as its **own toolset entry** — it is not selectable through `agent_toolset_20260401.enabled_tools`:

```json
{
  "tools": [
    {"type": "agent_toolset_20260401", "enabled_tools": ["Bash", "Read", "Write"]},
    {"type": "browser_toolset_20260714"}
  ]
}
```

- Enables all `browser_*` tools plus session live preview; accepts only `type` (no per-tool allowlist).
- **Writing this entry into `tools` needs the header `x-qoder-beta: browser-use-2026-07-14`** — on agent create and agent update. Without it those writes return 400. Session create against an already-configured agent does not need the header. `503 feature_not_available` = temporarily unavailable.
- Beta: behavior and limits may change. Agent updates apply only to sessions created afterwards.

### MCP Tools (Model Context Protocol)

MCP servers are declared in the **top-level `mcp_servers` field** of the agent (max 20); a `mcp_toolset` tool entry references one by name:

```json
{
  "name": "my-agent",
  "system": "You are a research assistant.",
  "model": "ultimate",
  "tools": [
    {"type": "agent_toolset_20260401"},
    {"type": "mcp_toolset", "mcp_server_name": "my-tools"}
  ],
  "mcp_servers": [
    {"name": "my-tools", "type": "url", "url": "https://my-mcp-server.example.com/mcp"}
  ]
}
```

- `mcp_server_name` must match an `mcp_servers[].name`, or the request fails.
- Only `type: "url"` (streamable HTTP) is supported; the server must be publicly reachable from the cloud environment.
- **Authentication is configured through Vaults**, not inline tokens: create a vault, add a credential with `auth.type: "static_bearer"` plus `mcp_server_url` (matching the agent's MCP URL) and `token`, then pass `vault_ids: ["vault_xxx"]` when creating the session. Secrets are never echoed back on reads.
- **MCP tool discovery runs at session create, not at first tool call.** An unreachable, expired, or unauthenticated MCP server makes `POST /sessions` fail outright with the upstream reason in the message (e.g. `Unauthorized` when the vault credential is missing). Treat a failing session create as "fix the MCP server or its credential first".
- MCP calls surface as `agent.mcp_tool_use` / `agent.mcp_tool_result` events.

### Custom Tools (client-side)

```json
{"type": "custom", "name": "lookup_order", "description": "Look up an order by ID", "input_schema": {"type": "object", "properties": {"order_id": {"type": "string"}}}}
```

When the agent calls it, the stream emits `agent.custom_tool_use`; your client executes the tool and returns `user.custom_tool_result` with the `custom_tool_use_id`.

### Model configuration

`model` accepts `"ultimate"` (string) or an object for tuning: `{"id": "ultimate", "effort": "high", "context_window": 200000}`. Query `GET /models` for each model's `efforts` and `available_context_windows`.

### Cost tracking

Model spend is reported as **credits**, not tokens: `GET /sessions/{id}` returns a cumulative `usage.total_credits` snapshot, and each `span.model_request_end` event carries `model_usage.credits` for that one call. Both are floored to 2 decimals. Overwrite your stored value per session ID rather than summing repeatedly.

---

## Environments

Environments define the execution context for sessions — the container where the agent's tools run.

### Creating an Environment

```json
POST /environments
{
  "name": "dev-env",
  "config": {
    "type": "cloud",
    "networking": {"type": "unrestricted"},
    "packages": {"apt": ["curl"], "npm": ["typescript@5.0.0"], "pip": []},
    "setup_script": "set -euo pipefail\n[ -d /workspace/.git ] || git clone https://github.com/me/repo /workspace"
  }
}
```

> **No default environment exists.** Every new account must create at least one environment before starting sessions. `config` itself is optional (omit it → default cloud), but **always set `networking` explicitly**: when it's absent the environment silently defaults to `limited` (no package managers, no MCP egress), which surfaces only later as install/network failures. A present-but-empty `config: {}` / `config: null` is rejected.

- `networking.type`: `unrestricted` or `limited` (the default when omitted).
- `packages`: package-manager → package spec strings, installed at sandbox preparation.
- `setup_script`: runs via `/bin/bash -lc` **after** packages install. Non-zero exit aborts session startup (error includes exit code + stderr excerpt). Make it idempotent.
- `type: "self_hosted"` environments exist for BYOC workers — they don't support session resources or environment variables.

### Environment Lifecycle

A new environment is **immediately usable** — the environment object has no `status` field. Container provisioning (package install, `setup_script`) happens when a session starts, so budget the cold-start time there.

- Reusable across multiple sessions; archive with `POST /environments/{id}/archive` when a used environment is no longer needed
- Archived environments cannot be used for new sessions
- `DELETE /environments/{id}` succeeds only when no session references the environment. Deleted sessions may retain the reference, so archive an environment that has hosted a session.

---

## Session Resources

Three resource types can be attached at `POST /sessions` via `resources[]`:

| Type | Required | Optional | Mounted at |
|---|---|---|---|
| `file` | `file_id` | `mount_path` | `/mnt/session/uploads/<file_id>` by default |
| `github_repository` | `url`, `authorization_token` | `mount_path`, `checkout` | path derived from repo name by default |
| `memory_store` | `memory_store_id` | `access` (`read_only`/`read_write`), `instructions` | injected into agent context |

Files can also be added mid-session: `POST /sessions/{id}/resources` (only `type: "file"`). GitHub repos and memory stores must be attached at creation. A given file can be attached to a session only once — re-attaching the same `file_id` is rejected as a conflict, so pick the `mount_path` you want on the first attach.

### Files

Upload once, attach per session:

```bash
FILE_ID=$(curl -s -X POST "$BASE/files" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -F "file=@./data.csv" | jq -r '.id')
```

- No `purpose` field — that's gone. Optional form fields: `name`, `metadata` (JSON string).
- Text-based files only; the response `id` is the file ID.
- Tell the agent the mount path explicitly in your message (default `/mnt/session/uploads/<file_id>`, or set `mount_path` yourself).

### GitHub Repositories

```json
{"type": "github_repository", "url": "https://github.com/org/repo", "mount_path": "/app/repo", "authorization_token": "ghp_xxx"}
```

- The platform clones the repo into the container at session start; the `gh` CLI ships in the runtime image with the token preconfigured, so the agent can push branches and open PRs.
- `authorization_token` (a GitHub PAT, fine-grained recommended) is write-only — never returned on reads. Use least-privilege tokens and revoke after use.
- Mounted repos cannot be swapped on a running session — create a new session to change URL/path.
- Container disk is ephemeral. Preserve outputs before the environment expires; commit and push only within the user-authorized workflow.

### Agent output → files (DeliverArtifacts)

Agents deliver produced files via the built-in `DeliverArtifacts` tool (write the file under `/data/`, then deliver). Each delivery emits an **`agent.artifact_delivered`** event with `file_id`, `original_filename`, `content_type`, `size` — watch for it and download the `file_id` via `GET /files/{id}/content` (presigned URL). Delivered artifacts are File objects with `downloadable: true`, scoped to the session.

---

## Memory Stores

Persistent knowledge that survives across sessions. Useful for:
- Shared project context across multiple agent sessions
- Accumulating learnings over time
- Cross-agent collaboration (multiple agents reading the same store)

### Setup

Entries are **path-addressed** (`path` required, unique per store):

```json
POST /memory_stores
{"name": "project-knowledge", "description": "Included in the agent's system prompt when attached"}

POST /memory_stores/{store_id}/memories
{
  "path": "infra/database.md",
  "content": "Production database is PostgreSQL 15 on us-west-2.",
  "metadata": {"category": "infra", "confidence": "high"}
}
```

### Attaching to Sessions

Attach via session `resources` (NOT via agent config):

```json
{"type": "memory_store", "memory_store_id": "memstore_xxx", "access": "read_write", "instructions": "Use this memory for long-lived project context."}
```

`read_write` lets the agent create/update entries during the session.

> **Only the store's `description` reaches the system prompt — entry *contents* do not.** Attaching a store does not put its entries in context; the agent has to go look them up with its own tools. Ask a bare "what do you remember?" and it may answer "nothing". Make the store's `description` and the resource's `instructions` say what lives in there and when to consult it, and in the turn itself tell the agent to check its memory before answering.

---

## Skills

Reusable capability packages uploaded as zip files and bound to agents.

### Uploading a Skill

Skills must be uploaded as a `.zip` (max 50 MiB) via `multipart/form-data`, with `SKILL.md` at the zip root (or one directory below). `SKILL.md` needs YAML frontmatter (`name`, `description`, `version`) — frontmatter wins over form fields:

```bash
zip -r my-skill.zip SKILL.md references/

curl -X POST "$BASE/skills" \
  -H "Authorization: Bearer $QODER_ACCESS_TOKEN" \
  -F "file=@./my-skill.zip"
```

> This is NOT a JSON endpoint. Sending `{"name":"..."}` as JSON returns an error. (Updating an existing skill IS JSON: `PUT /skills/{id}` with `content` + `content_encoding: "base64"` for a new zip.)

### Binding to an Agent

Bind via agent update — `POST /agents/{id}` (not PUT), `version` required for the optimistic lock:

```json
POST /agents/{id}
{
  "version": 1,
  "skills": [
    {"type": "custom", "skill_id": "skill_xxx"}
  ]
}
```

`skills` is a **full replacement** of the stored binding list — include existing bindings you want to keep. The agent object returns `skills` on GET, so read it back to verify.

### Lifecycle

- Skills can be directly deleted (`DELETE /skills/{id}`) without archiving first
- Deleting a skill that is bound to an agent does not automatically unbind it — update the agent separately

---

## Deployments (Scheduled / Manual Runs)

Deployments make an agent run on a cron schedule (or on demand) without you creating sessions by hand. Each run creates a session (with `deployment_id` set) and delivers `initial_events` to it.

```json
POST /deployments
{
  "name": "daily-report",
  "agent": "agent_xxx",
  "environment_id": "env_xxx",
  "schedule": {"type": "cron", "expression": "0 9 * * *", "timezone": "Asia/Shanghai"},
  "initial_events": [
    {"type": "user.message", "content": [{"type": "text", "text": "Generate today's status report"}]}
  ]
}
```

- Omit `schedule` for a manual-only deployment; trigger runs with `POST /deployments/{id}/run`.
- `pause` / `unpause` control the schedule; runs are inspectable via `GET /deployments/{id}/runs`.
- `resources`, `vault_ids`, `environment_variables`, `metadata` are supported the same way as on sessions.
- To fetch a run's output, get the run's session and read its events.

For push-based automation the other direction (platform → your service), register **Webhooks**: HTTP POST callbacks on lifecycle events (session created/updated/archived, agent changes, etc.) with HMAC-SHA256 signatures and at-least-once delivery. See the official Webhook API reference via `live-sources.md`.
