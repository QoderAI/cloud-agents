# Managed HTTP API Reference

HTTP mode only: read [setup and transport selection](guide.md) first. These Managed examples are separate from the MCP tool schemas; consult current operation docs for deployment-specific fields.

## Contents

- [Agents](#agents)
- [Authentication](#authentication)
- [Models](#models)
- [Environments](#environments)
- [Sessions](#sessions)
- [Events](#events)
- [Files](#files)
- [Memory Stores](#memory-stores)
- [Skills](#skills)
- [Vaults](#vaults)
- [Deployments](#deployments)
- [Common Response Codes](#common-response-codes)

---

All endpoints are under `https://api.qoder.com.cn/api/v1/cloud`.

Every request requires:
```
Authorization: Bearer <PAT or SAT>
```

Add `Content-Type: application/json` only when the request body is JSON. Multipart uploads (`POST /files`, `POST /skills`) must use `multipart/form-data` with the client-generated boundary. Bodyless GET/DELETE requests do not need a Content-Type header.

Two token types are accepted — see [Authentication](#authentication).

Pagination: most list endpoints are cursor-based. Prefer `page` (pass the previous response's `next_page` value); `after_id` / `before_id` remain as compatibility cursors. The three cursor params are **mutually exclusive** — sending more than one returns 400. Response shape: `{data: [...], next_page, first_id, last_id, has_more}`. `GET /models` is the explicit non-paginated exception and returns `{data, has_more}`.

Resource ID prefixes: `agent_`, `sess_`, `env_`, `evt_`, `file_`, `memstore_`, `mem_`, `skill_`, `dep_`, `sthr_`, `vault_`.

---

## Authentication

| Token | Identity | Use case | How to get it |
|---|---|---|---|
| **PAT** (`pt-` prefix) | A user | Personal development and testing | Created in the Qoder console (see guide.md § Authentication) |
| **SAT** (JWT) | A Service Account | Server-side integrations, CI, automation | Exchange a Service Account API Key (SA Key) for a short-lived token |

Both go in the same header. Docs use `QODER_ACCESS_TOKEN` as the neutral variable name for "whichever token you hold".

### Getting an SAT

An org admin supplies an SA Key. Request scopes at token exchange, within the key's permissions. Use matching region/environment endpoints; the CN production pair is `https://openapi.qoder.com.cn` for exchange and `https://api.qoder.com.cn` for business APIs. Never mix exchange and business endpoints from different regions.

```bash
: "${QODER_SA_KEY:?Configure the SA Key securely}"
: "${QODER_OPENAPI_BASE_URL:?Set the matching regional exchange origin}"
SAT_RESPONSE=$(curl --fail-with-body -sS "$QODER_OPENAPI_BASE_URL/api/v1/serviceToken/exchange" \
  -H "Authorization: Bearer $QODER_SA_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"grant_type":"client_credentials","audience":"qoder","scope":"qca.access","ttl_seconds":3600}')
QODER_ACCESS_TOKEN=$(printf '%s' "$SAT_RESPONSE" | jq -er '.access_token | select(type == "string" and length > 0)')
export QODER_ACCESS_TOKEN
unset SAT_RESPONSE
```

Stop on exchange/parsing failure. `qca.access` covers Managed; request `qca.access forward.access` only when both API families are needed and permitted. An SAT expires in at most 43200 seconds; exchange again rather than refresh it. Never use the SA Key as the business bearer token or print token responses.

For CN operation documentation, use [CN live sources](live-sources.md). The SAT origin follows the existing CN production configuration; the supplied CN reference package does not document SAT exchange.

---

## Agents

| Method | Path | Description |
|--------|------|-------------|
| POST | `/agents` | Create agent |
| GET | `/agents` | List agents (excludes archived) |
| GET | `/agents/{id}` | Get agent (`?version=N` returns a version snapshot) |
| GET | `/agents/{id}/versions` | List version snapshots |
| POST | `/agents/{id}` | Update agent (OCC, requires `version`) |
| POST | `/agents/{id}/archive` | Archive agent |
| DELETE | `/agents/{id}` | Delete agent (archive optional) |

### POST /agents

```json
{
  "name": "my-agent",
  "description": "Optional description",
  "system": "You are a helpful assistant.",
  "model": "ultimate",
  "tools": [{"type": "agent_toolset_20260401"}]
}
```

- `model` accepts a string ID or an object `{"id": "ultimate", "effort": "high", "context_window": 200000}`. Discover valid values via [GET /models](#models).
- `tools` is a union by `type`: `agent_toolset_20260401` (built-in tools, optional `enabled_tools` allowlist / `disallowed_tools` / per-tool `configs[].permission_policy`), `browser_toolset_20260714` (Browser Use, Beta — see below), `mcp_toolset` (requires `mcp_server_name` referencing a top-level `mcp_servers[].name`), and `custom` (client-side tool: `name`, `description`, `input_schema`).
- `mcp_servers` is a **top-level agent field**: `[{"name": "my-tools", "type": "url", "url": "https://..."}]`. MCP auth goes through Vaults (`static_bearer` credential + `vault_ids` on the session), not inline tokens — see `tools-and-resources.md` § MCP Tools.
- `skills` binds uploaded skills: `[{"type": "custom", "skill_id": "skill_xxx"}]` (optional `version` string).

On success returns the agent object with `type: "agent"`, `version: 1`, `archived_at`, `multiagent`, `metadata` fields.

### Browser Use (Beta)

`browser_toolset_20260714` enables the platform-managed `browser_*` tools plus session live preview. It is a **separate tools entry**, not a member of `agent_toolset_20260401.enabled_tools`, and accepts only the `type` field (no per-tool selection):

```json
{"tools": [{"type": "agent_toolset_20260401"}, {"type": "browser_toolset_20260714"}]}
```

Any request that **writes** this into `tools` — agent create and agent update — must carry the Beta header:

```
x-qoder-beta: browser-use-2026-07-14
```

Omitting it on those writes returns 400 (`tools containing browser_toolset_20260714 require the 'x-qoder-beta: browser-use-2026-07-14' header.`); `503 feature_not_available` means Browser Use is temporarily off. Creating a session against an agent that already has the toolset does **not** need the header (the toolset lives in the agent snapshot; session create doesn't re-validate it). Being Beta, its behavior and limits may change. Updating an agent affects only sessions created afterwards — existing sessions keep their agent snapshot.

### POST /agents/{id} (update)

> Update is **POST to the agent path**.

Partial update with optimistic concurrency control — `version` is **required** and must match the current value (fetch via GET first). Stale version returns 409 `Version conflict`. On success `version` increments.

```json
{
  "name": "updated-name",
  "version": 1
}
```

Semantics per field: scalar fields (`name`, `system`, `description`, `model`) — omitted fields are preserved. **Array fields (`tools`, `mcp_servers`, `skills`) are full replacements** of the stored list. `metadata` is a patch: string values upsert keys, `null` values delete keys.

### POST /agents/{id}/archive

No body. Returns the agent with `archived_at` set.

### DELETE /agents/{id}

No body. Can be called directly without archiving first.

---

## Models

| Method | Path | Description |
|--------|------|-------------|
| GET | `/models` | List models enabled for the account |

Response (not paginated, `has_more` always false; values below are illustrative — query live for current models/windows):

```json
{
  "data": [
    {
      "id": "ultimate",
      "type": "model",
      "display_name": "Ultimate",
      "is_enabled": true,
      "efforts": ["low", "medium", "high", "xhigh", "max"],
      "default_effort": "high",
      "default_context_window": 200000,
      "available_context_windows": [200000, 400000, 1000000]
    }
  ],
  "has_more": false
}
```

Use `id` as the agent's `model`, and `efforts` / `available_context_windows` to fill the object form's `effort` / `context_window`. Note: some models (e.g. `auto`) expose no `efforts`/`context_windows` — they only accept the string form.

---

## Environments

| Method | Path | Description |
|--------|------|-------------|
| POST | `/environments` | Create environment |
| GET | `/environments` | List environments |
| GET | `/environments/{id}` | Get environment |
| POST | `/environments/{id}` | Update environment |
| POST | `/environments/{id}/archive` | Archive environment |
| DELETE | `/environments/{id}` | Delete environment only when no session references it |

### POST /environments

```json
{
  "name": "my-env",
  "config": {
    "type": "cloud",
    "networking": {"type": "unrestricted"},
    "packages": {"apt": ["curl"], "npm": [], "pip": []},
    "setup_script": "set -euo pipefail\n[ -d /workspace/.git ] || git clone https://github.com/me/repo /workspace"
  }
}
```

> **Always set `networking` explicitly.** `config` is optional — omit it and you get a default cloud environment — but when `networking` is absent it resolves to **`limited`** (no package managers, no MCP egress), and that choice is silent until the agent later fails to install packages or reach the network. A present-but-empty `config: {}` or `config: null` is rejected; either omit the key entirely or give it a valid object with `type`.

- `config.type`: `"cloud"` or `"self_hosted"`. The other config fields are only valid for `cloud`.
- `networking.type`: `limited` (default when omitted) or `unrestricted`.
- `packages`: maps package managers (`apt`, `npm`, `pip`) to arrays of package spec strings.
- `setup_script`: shell script run via `/bin/bash -lc` during sandbox preparation, after `packages` install. Non-zero exit aborts session startup (error includes exit code + stderr excerpt). Make it idempotent — it may run on every sandbox provision.

A new environment is **immediately usable** — there is no `status` field on the environment object. Actual container provisioning (package install, `setup_script`) happens when a session starts, which is part of why first turns are slower.

Environment deletion is stricter than session deletion: `DELETE /environments/{id}` succeeds only when no session references the environment. Deleted sessions may retain that reference, so an environment used by a test session should normally be archived with `POST /environments/{id}/archive` instead of deleted.

---

## Sessions

| Method | Path | Description |
|--------|------|-------------|
| POST | `/sessions` | Create session |
| GET | `/sessions` | List sessions |
| GET | `/sessions/{id}` | Get session |
| POST | `/sessions/{id}` | Update session (title/metadata) |
| POST | `/sessions/{id}/archive` | Archive session |
| DELETE | `/sessions/{id}` | Delete session |
| POST | `/sessions/{id}/cancel` | Cancel running turn |
| POST | `/sessions/{id}/resources` | Add a file resource mid-session |
| GET | `/sessions/{id}/resources` | List session resources |
| GET | `/sessions/{id}/threads` | List threads (managed-agent sessions) |

### POST /sessions

```json
{
  "agent": "agent_xxx",
  "environment_id": "env_xxx",
  "title": "Optional title",
  "environment_variables": "FEATURE_FLAG=on;LOG_LEVEL=debug",
  "resources": [
    {"type": "file", "file_id": "file_xxx", "mount_path": "/data/inputs/data.csv"},
    {"type": "github_repository", "url": "https://github.com/org/repo", "authorization_token": "ghp_xxx", "mount_path": "/app/repo"},
    {"type": "memory_store", "memory_store_id": "memstore_xxx", "access": "read_write"}
  ],
  "vault_ids": []
}
```

> **Critical**: field name is `agent`, NOT `agent_id`. It accepts a string ID or an object `{"id": "agent_xxx", "type": "agent", "version": 2}` to pin an agent version (object form must include `type: "agent"`).

- `environment_id` is **required**; the environment must exist and not be archived.
- If the agent declares `mcp_servers`, the platform performs **MCP tool discovery during this call** — an unreachable or unauthorized MCP server fails session creation. Attach the credential vault via `vault_ids` for authenticated servers.
- `resources[]` entries carry a `type` discriminator: `file` (optional `mount_path`, defaults to `/mnt/session/uploads/<file_id>`), `github_repository` (requires `url` + `authorization_token`; cloned at `mount_path`), `memory_store` (optional `access`: `read_only`/`read_write`, optional `instructions`).
- `environment_variables` is a **string** of `KEY=VALUE` pairs separated by `;` or newlines. Names must match `[A-Za-z_][A-Za-z0-9_]*`; reserved names (`SERVER_ENDPOINT`, `USER_ID`, `WORK_DIR`, prefixes `CAW_`/`QODER_`) are rejected. The response echoes it as a JSON object. Note: the variables are exported to **login shells** — the agent's Bash tool runs non-login commands by default, so have the agent use `bash -lc '...'` when it needs to read them.
- Legacy fields `environment`, `vaults`, `memory_store_ids` are **not supported**.
- Self-hosted environments do not support session resources or environment variables.

On success returns the session object with `type: "session"`, `status` (`idle` initially), embedded `agent` snapshot, `resources`, `stats`, `usage`, `deployment_id`. Legacy fields `agent_id`, `turn_status` are gone.

Session `status` values: `rescheduling`, `running`, `idle`, `canceling` (transient), `terminated`.

`usage` is a **cumulative credits snapshot** for the session: `{"total_credits": 5.94}` (`0` on a fresh session, field omitted when no usage data exists). Values are **floored** to at most 2 decimals (`7.6681` → `7.66`), and JSON drops trailing zeroes (`1.20` → `1.2`). Treat it as a snapshot — overwrite your local value per session ID, never accumulate it yourself.

> If the agent uses `browser_toolset_20260714`, that toolset is captured in the agent snapshot at agent create/update time — session create needs no extra header.

### POST /sessions/{id}/cancel

No body. Acknowledged immediately and safe to call even when the session is already idle. The session passes through the transient `canceling` status and settles asynchronously — watch the stream for `session.status_idle` or poll `GET /sessions/{id}`.

---

## Events

| Method | Path | Description |
|--------|------|-------------|
| POST | `/sessions/{id}/events` | Send event(s) |
| GET | `/sessions/{id}/events` | List events (paginated, `order=asc|desc`) |
| GET | `/sessions/{id}/events/stream` | SSE stream |

### POST /sessions/{id}/events

```json
{
  "events": [
    {
      "type": "user.message",
      "content": [{"type": "text", "text": "Hello, what can you do?"}]
    }
  ]
}
```

> **Critical ×2**: (1) the body must wrap events in an `events` array — a bare event object returns 400. (2) `content` must be a **non-empty array of content blocks** — a plain string returns 400.

Accepted client event types:

| Type | Required fields | Notes |
|---|---|---|
| `user.message` | `content` | array of content blocks |
| `user.interrupt` | — | optional `session_thread_id` |
| `user.tool_confirmation` | `tool_use_id`, `result` | `result`: `allow` / `deny`; optional `deny_message` |
| `user.tool_result` | `tool_use_id` | self-hosted worker tool results |
| `user.custom_tool_result` | `custom_tool_use_id` | client-side custom tool results |
| `user.define_outcome` | `description`, `rubric` | `rubric`: `{"type":"text","content":"..."}` or `{"type":"file","file_id":"..."}` |
| `system.message` | `content` | max one per request, must be the final event |

Accepted asynchronously; the response body is `{data: [...]}` containing the accepted event objects.

### GET /sessions/{id}/events/stream

SSE endpoint. See `events.md` for event types, `event_deltas[]` incremental streaming, and `Last-Event-ID` reconnection.

```
GET /sessions/{id}/events/stream
Authorization: Bearer <PAT>
Accept: text/event-stream
Last-Event-ID: evt_xxx        # optional, resume after this event
```

Optional query param `event_deltas[]` (repeatable; values `agent.message`, `agent.thinking`) turns on incremental output for this connection.

### GET /sessions/{id}/events

Query params: `page` / `after_id` / `before_id` (mutually exclusive), `limit` (default 20, max 100), `order` (`asc`/`desc`).

---

## Files

| Method | Path | Description |
|--------|------|-------------|
| POST | `/files` | Upload file (multipart) |
| GET | `/files` | List files |
| GET | `/files/{id}` | Get file metadata |
| GET | `/files/{id}/content` | Get a presigned download URL |
| DELETE | `/files/{id}` | Delete file |

### POST /files

Multipart form upload — text-based files only:
```
Content-Type: multipart/form-data
- file: <binary>            # required
- name: "my-data.csv"       # optional, defaults to uploaded filename
- metadata: '{"k":"v"}'     # optional JSON string, max 8 KB
```

> The legacy `purpose` form field has been removed — do not send it.

Response `200`:
```json
{
  "id": "file_xxx",
  "type": "file",
  "filename": "my-data.csv",
  "mime_type": "text/csv",
  "size_bytes": 1024,
  "downloadable": false,
  "scope": null,
  "metadata": {},
  "created_at": "2026-01-01T00:00:00Z"
}
```

> The ID field is `id` (not `file_id`). To let an agent read the file, attach it as a session resource: `{"type": "file", "file_id": "file_xxx"}` on `POST /sessions` (or `POST /sessions/{id}/resources` mid-session). Default mount path is `/mnt/session/uploads/<file_id>`; pass `mount_path` to control it.

### GET /files/{id}/content

Only works when the file object's `downloadable` is `true` (agent-delivered artifacts are; your own uploads generally are not). Returns a short-lived presigned URL:

```json
{"url": "https://...", "expires_at": "2026-01-01T01:00:00Z"}
```

Fetch with `curl -L "$url"` — no Authorization header needed. Mint a fresh URL by calling the endpoint again after expiry.

### Agent-produced files

Agents deliver output files via the built-in `DeliverArtifacts` tool (enabled by default; must be listed explicitly if you set a non-empty `enabled_tools` allowlist). Each delivery emits an **`agent.artifact_delivered`** event carrying `file_id`, `original_filename`, `content_type`, and `size` — that is the signal to consume. Delivered artifacts are File objects with `downloadable: true` and `scope` set to the session; download via `GET /files/{id}/content`.

---

## Memory Stores

| Method | Path | Description |
|--------|------|-------------|
| POST | `/memory_stores` | Create memory store |
| GET | `/memory_stores` | List memory stores |
| GET | `/memory_stores/{id}` | Get memory store |
| POST | `/memory_stores/{id}` | Update store (name/description/metadata) |
| POST | `/memory_stores/{id}/archive` | Archive store |
| DELETE | `/memory_stores/{id}` | Delete store |
| POST | `/memory_stores/{id}/memories` | Create memory entry |
| GET | `/memory_stores/{id}/memories` | List memories |
| GET | `/memory_stores/{id}/memories/{mem_id}` | Get memory |
| POST | `/memory_stores/{id}/memories/{mem_id}` | Update memory |
| DELETE | `/memory_stores/{id}/memories/{mem_id}` | Delete memory |

ID prefix is `memstore_` for stores, `mem_` for entries.

### POST /memory_stores

```json
{
  "name": "project-knowledge",
  "description": "Included in the agent's system prompt when attached",
  "metadata": {"team": "backend"}
}
```

### POST /memory_stores/{id}/memories

Entries are **path-addressed** — `path` is required and unique within the store (duplicate path returns 409):

```json
{
  "path": "infra/deployment-target.md",
  "content": "The deployment target is us-west-2.",
  "metadata": {"category": "infra"}
}
```

### Attaching to sessions

Attach a store via session `resources`: `{"type": "memory_store", "memory_store_id": "memstore_xxx", "access": "read_write"}`. Only the store's `description` is injected into the system prompt — **entry contents are not auto-loaded into context**; the agent reads/writes them through its own tools at runtime. So set a `description`/`instructions` that tells the agent what's in the store and when to consult it, and prompt it to check its memory in the turn itself — otherwise a bare "what do you remember?" can come back empty. See `tools-and-resources.md` § Memory Stores.

---

## Skills

| Method | Path | Description |
|--------|------|-------------|
| POST | `/skills` | Upload skill (multipart/form-data) |
| GET | `/skills` | List skills |
| GET | `/skills/{id}` | Get skill |
| GET | `/skills/{id}/versions` | List skill versions |
| PUT | `/skills/{id}` | Update skill (JSON) |
| DELETE | `/skills/{id}` | Delete skill (no archive needed) |

### POST /skills

Multipart form upload — a `.zip` (max 50 MiB) with `SKILL.md` at the zip root (or one directory below):
```
Content-Type: multipart/form-data
- file: <skill.zip>
- type: "custom" (optional, default)
- metadata: '{"k":"v"}' (optional JSON string)
```

`name` / `description` form fields are accepted but **overridden by the SKILL.md YAML frontmatter** (`name`, `description`, `version`).

Returns the created skill object (use its `id` for binding and deletion).

### PUT /skills/{id}

JSON body (this one IS a PUT, unlike agents): `name`, `description`, `metadata`, and optionally new `content` — plain text, or a base64-encoded zip with `content_encoding: "base64"`.

### DELETE /skills/{id}

No body. Immediate — no archive step.

### Binding skills to agents

Bind via agent create or update (`POST /agents/{id}`, requires `version`):

```json
{
  "version": 1,
  "skills": [{"type": "custom", "skill_id": "skill_xxx"}]
}
```

> `skills` is a full-replacement array on update. The agent object now returns `skills` on GET, so you can verify the binding by reading the agent.

---

## Vaults

Vaults hold credentials the runtime injects — most importantly **auth for MCP servers** (the agent's `mcp_servers` never carry inline tokens). Attach a vault to a session with `vault_ids: ["vault_xxx"]`.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/vaults` | Create vault |
| GET | `/vaults` | List vaults |
| GET | `/vaults/{id}` | Get vault |
| POST | `/vaults/{id}/archive` | Archive vault |
| DELETE | `/vaults/{id}` | Delete vault |
| POST | `/vaults/{id}/credentials` | Add a credential |
| GET | `/vaults/{id}/credentials` | List credentials |
| DELETE | `/vaults/{id}/credentials/{cred_id}` | Delete a credential |

ID prefixes: `vault_` for vaults, `vcred_` for credentials.

### POST /vaults

```json
{"display_name": "mcp-creds", "metadata": {"team": "docs"}}
```

Returns the vault with `type: "vault"`, `credentials: []`, `archived_at`, timestamps.

### POST /vaults/{id}/credentials

The credential is a wrapper around an `auth` object; the `auth.type` decides the shape:

```json
{"auth": {"type": "static_bearer", "mcp_server_url": "https://my-mcp.example.com/mcp", "token": "<secret>"}}
```

| `auth.type` | Required fields | Use |
|---|---|---|
| `static_bearer` | `mcp_server_url`, `token` | A fixed bearer token for an MCP server. `mcp_server_url` must match the agent's `mcp_servers[].url`. |
| `environment_variable` | `secret_name`, `secret_value` | Inject a secret as an env var into the container (`secret_name` must match `[A-Za-z_][A-Za-z0-9_]*`). |

> **Secrets are write-only.** The response and both list/get echo the credential *shape* (`id`, `auth.type`, `auth.mcp_server_url`, `secret_name`) but **never** the `token` / `secret_value`. To rotate a secret, delete the credential and add a new one; there is no in-place update.

### Using a vault with MCP

1. Create the vault, add a `static_bearer` credential whose `mcp_server_url` equals the agent's MCP server URL.
2. Create the session with `vault_ids: ["vault_xxx"]`.
3. On session create the platform runs MCP tool discovery **using that credential** — a missing/mismatched credential surfaces as an upstream `Unauthorized` and fails session creation (see `tools-and-resources.md` § MCP Tools).

---

## Deployments

Run an agent on a cron schedule (or manually) — each run creates a session and delivers `initial_events` to it.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/deployments` | Create deployment |
| GET | `/deployments` | List deployments |
| GET | `/deployments/{id}` | Get deployment |
| POST | `/deployments/{id}` | Update deployment |
| POST | `/deployments/{id}/pause` | Pause schedule |
| POST | `/deployments/{id}/unpause` | Resume schedule |
| POST | `/deployments/{id}/run` | Trigger a run now |
| GET | `/deployments/{id}/runs` | List runs of a deployment |
| GET | `/deployment_runs` | List runs across all deployments (top-level path) |
| POST | `/deployments/{id}/archive` | Archive deployment |

### POST /deployments

```json
{
  "name": "daily-report",
  "agent": "agent_xxx",
  "environment_id": "env_xxx",
  "schedule": {"type": "cron", "expression": "0 9 * * *", "timezone": "Asia/Shanghai"},
  "initial_events": [
    {"type": "user.message", "content": [{"type": "text", "text": "Generate today's status report"}]}
  ],
  "resources": [],
  "vault_ids": []
}
```

- `schedule` omitted ⇒ manual-only deployment (trigger with `POST /deployments/{id}/run`).
- `initial_events`: 1-50 events, types `user.message` / `user.define_outcome` / `system.message`, content-block format.
- `environment_variables` (string, same format/validation as sessions) and `metadata` are also supported.
- ID prefix `dep_`. Newly created deployments are `active` and fire immediately per schedule.

Each run creates a session with `deployment_id` set — find run output by streaming/listing that session's events.

---

## Common Response Codes

| Code | Meaning |
|------|---------|
| 200 / 201 | Success (create, get, update, archive, delete) |
| 202 | Accepted (async operations) |
| 400 | Bad request (missing/invalid fields, plain-string content, multiple cursors) |
| 401 | Unauthorized (invalid/missing PAT) |
| 404 | Resource not found |
| 409 | Conflict (version mismatch on agent update, duplicate memory path) |
| 429 | Rate limited — honor `Retry-After`, back off |
| 500 | Internal server error |
