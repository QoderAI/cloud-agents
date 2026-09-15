# Forward workflows over MCP

Use Forward for Template-based assistants and their Sessions/Schedules. For direct Agent composition without business Identity prerequisites, use [Managed workflows](workflows.md). All examples use logical tool names and illustrative IDs; replace them with actual values.

## Prepare an existing Identity and a Template

Identity tools are not exposed. Obtain the actual `identity_id` from the user, approved application context, or an existing Forward Session/Schedule. Use the actual business Identity rather than an inferred ID. Without this prerequisite, ask for application setup; Template/Environment creation alone cannot produce a runnable Forward conversation.

Use `list_forward_templates` / `get_forward_template` to reuse an assistant. To create one, choose a model using shared `list_models` and an accessible Environment using Forward environment list/get. Only create new resources when the task requires them.

Unlike Managed, `create_forward_environment` requires config:

```json
{"name":"assistant-runtime","config":{"type":"cloud"}}
```

Cloud config permits type, optional packages (apt/npm/pip arrays of strings) and setup_script. Self-hosted permits type and optional setup_script only. Never send networking, singular package, packages.type or a copied read-response config. These rules also apply to updates; omit config to preserve it.

Call `create_forward_template` with name, model and non-blank environment_id; there is no implicit Environment default:

```json
{"name":"support-assistant","model":"model-from-discovery","system":"Answer using the customer's configured resources.","environment_id":"env_example"}
```

Read back the Template. Creation saves configuration without starting a Session. Template defaults and existing business configuration are resolved by Forward; no effective-config or Identity-personalization tool is exposed here. Do not promise all runtime settings equal Template defaults. Inspect the returned Session config after creation. Clone a Template for an independent variant instead of modifying a shared original.

### Update an existing Template

`update_forward_template` requires only `template_id`; read the current Template first and send only intended changes. Omitted fields are preserved. Successful updates persist a new Template version, not a Session run.

- If supplied, `name` must contain non-whitespace text and fit within **256 UTF-8 bytes**, not 256 Chinese characters. `model` may be omitted to keep it; when supplied, use a discovered model ID or the declared model object, not null.
- `environment_id` may be omitted to preserve the binding, set to an accessible ID to rebind, or set to null/empty string to clear it. This differs from creation, which requires a non-blank Environment ID. Clearing does not select a default or prove future execution will work; ensure effective runtime configuration before running.
- `tunnel_id` also supports null/empty string to clear. `vault_ids`, `environment_variables`, `github_repositories`, and `metadata` accept null to clear; omit them to keep existing values. Do not copy nulls from a read response into an unrelated update. Non-null metadata merges keys; do not assume all configuration fields share that merge behavior.
- `status` is `active` or `archived`. Sending `archived` invokes archival and must be intentional; prefer the dedicated `archive_forward_template` tool for that operation. An already archived Template cannot be updated or restored by sending `active`.

For a description-only change, no Environment or model needs to be resent:

```json
{"template_id":"tmpl_example","description":"Updated assistant description"}
```

Use an actual Template ID, then read back the changed fields and verify unrelated bindings remain unchanged.

### Template tools, MCP servers and Skills

The same object-array shapes apply to `create_forward_template` and `update_forward_template`. Do not pass arrays of tool names, server URLs or Skill IDs alone. On update, each supplied array replaces that entire field; omission preserves it and `[]` clears it.

- `tools` (at most 128): select an object branch by `type`: `agent_toolset_20260401`, `mcp_toolset`, `browser_toolset_20260714`, or `custom`. Built-in names go inside `enabled_tools`, not directly in tools. An MCP toolset requires `mcp_server_name`; a custom tool requires `name`, `description`, and an object `input_schema`. Forward per-tool `configs[].permission_policy.type` is `always_allow`, `always_ask`, or `always_deny`; do not copy Managed's `ask`/`default` aliases into these configs. Browser runtime availability is deployment/eligibility-dependent.
- `mcp_servers` (at most 20): each object has `type:"url"`, a unique non-empty `name`, and `url`. Reference that name from the MCP toolset. Never include `headers` or `authorization_token`; use Vault credentials.
- `skills` (at most 20): each object has `type:"qoder"` or `"custom"`, and `skill_id`. Optional `version` is `"latest"` or a numeric **string**; omission follows the latest version. Optional `enabled` is a boolean, default true; Forward omits disabled bindings when compiling the runtime Agent. Resolve actual Skill IDs/versions first.

Shape example for an explicitly requested Template configuration change (replace the example Template/Skill IDs and server URL with authorized actual values):

```json
{
  "template_id": "tmpl_example",
  "tools": [
    {"type":"agent_toolset_20260401","enabled_tools":["Bash","DeliverArtifacts"]},
    {"type":"mcp_toolset","mcp_server_name":"docs"}
  ],
  "mcp_servers": [{"type":"url","name":"docs","url":"https://example.com/mcp"}],
  "skills": [{"type":"custom","skill_id":"skill_example","version":"latest","enabled":true}]
}
```

### Template file bindings

On both create and update, `files` is an object keyed by existing File ID, or null—not a string, array, boolean, filename or URL. The MCP schema may be broader than this backend contract and let invalid values reach a validation error.

Each value must be an object. Put the identifier in the map key, not a nested `file_id`, `id` or `resource_id`. Optional `enabled` must be boolean. Example shape (replace the key with a returned, accessible File ID):

```json
{"files":{"file_example":{"enabled":true}}}
```

Omit `files` on update to preserve existing bindings; `{}` or null clears them. Build the complete intended map for a replacement. This binds existing files; it does not upload bytes. Other nested fields must be supported by the deployed backend—do not invent them because the schema is permissive.

## Run and continue a Forward Session

After obtaining an existing business identity_id and choosing a valid Template, call `create_forward_session`:

```json
{"identity_id":"idn_example","template_id":"tmpl_example","title":"Support case analysis"}
```

This is not Managed `create_session`: do not pass `agent`, `agent_id`, or top-level `environment_id`. Optional `config` contains backend-supported per-session overrides; do not copy a Managed Agent patch into it. Optional resources are existing file references; no upload bytes.

Reuse the returned Session ID. Call `send_forward_session_events`:

```json
{"session_id":"sess_example","events":[{"type":"user.message","content":[{"type":"text","text":"Summarize the open support questions."}]}]}
```

The events schema accepts six typed user event variants; it rejects system/assistant events and malformed user messages. Ordinary messages use content blocks. A requested tool confirmation requires the user's authorization before responding; never auto-approve it.

Observe through Forward:

1. `get_forward_session` reports business/runtime state. `list_forward_sessions` supports Identity/Template/source/time filters; use it for discovery, not whole-account scans.
2. `list_forward_session_events` returns full event payloads in **`item.data`**. Start with a narrow types filter and small limit; `agent.message` identifies Agent messages. There is no Forward summary-only tool.
3. Stop polling when the returned state says the turn is finished, or on failure/cancellation/deadline/user stop. Use bounded delays, not rapid polling or a permanently open call.
4. Continue the same Session after it is ready for another turn. On uncertain send completion, inspect state/events before repeating the write.

Call `list_forward_session_events`:

```json
{"session_id":"sess_example","types":["agent.message"],"order":"desc","limit":5}
```

Use `update_forward_session` for title/metadata/config changes. Use `add_forward_session_resource` for a file attachment, not a Managed resource mutation. `cancel_forward_session` requests turn cancellation; verify readback. `archive_forward_session` also updates Forward directory and Channel conversation lifecycle. Do not replace these with similarly named Managed calls.

## Forward Schedule, not Managed Deployment

Use a Schedule when the recurring task belongs to an Identity and Template. Use Managed Deployment for a standalone Agent/environment job.

Call `create_forward_schedule`:

```json
{"identity_id":"idn_example","template_id":"tmpl_example","name":"Daily support summary","environment_id":"env_example","initial_events":[{"type":"user.message","content":"Summarize today's support cases."}],"trigger_policy":{"type":"cron","expression":"0 9 * * *","timezone":"Asia/Shanghai"}}
```

Important differences:

- Forward uses `trigger_policy`, not Managed `schedule`.
- On both create and update, every Forward Schedule initial_events entry requires `type:"user.message"` and **string content**, unlike Session/Managed Deployment message blocks. No other event type is supported; do not use `assistant.message` or copy the nested Session shape.
- Optional `execution` controls attempt/concurrency/session policies; use the constraints below even when the live schema is permissive. Optional `sinks` controls delivery; use existing approved sink configuration instead of guessing an external recipient/channel.
- Creating/updating a Schedule does not authorize a separate immediate test run, and runs can consume credits.

When an immediate run is requested, call `run_forward_schedule` with schedule_id. To recover/locate the run, call `list_forward_schedule_runs` with the owning Identity and optional Schedule filter, e.g. `{"identity_id":"idn_...","schedule_id":"sched_..."}`. identity_id is required even when schedule_id is supplied; retrieve the owning Identity from the Schedule if only its ID is known, never invent it. Then use `get_forward_schedule_run` to inspect status/error/session association. If a Session ID is available, continue with Forward Session/event tools. Do not use a Managed `deployment_id` or `get_deployment_run`.

### Schedule execution options

For both create and update:

- `session_mode`: `new_session` or `reuse_session`. Omit it for normal default/retention behavior; an empty string follows backend default/merge handling, not a third execution mode. For independent runs use `new_session`; choose reuse only when the existing-session lifecycle is intended.
- `max_attempts`: integer 1–2 when supplied.
- `max_concurrent_runs`: positive integer when supplied.
- `timeout_ms`: integer at least 10 when supplied. This is a backend minimum in milliseconds, not a recommended practical task timeout.
- With effective `session_mode:"reuse_session"`, effective `max_concurrent_runs` must be 1. Consider existing values on partial updates; send both fields when switching to reuse if concurrency was greater than 1.

For ordinary creation, omit `execution` unless customization is needed. A new-session example is `{"execution":{"session_mode":"new_session","max_attempts":1,"max_concurrent_runs":1,"timeout_ms":60000}}`. Do not send arbitrary mode strings merely because the MCP schema accepts strings; unsupported modes return validation errors.

Inspect before `update_forward_schedule`; omit unchanged fields. Pause/unpause controls future scheduling; archive retires the Schedule. Confirm retirement through `archived_at`, even if `status` remains `active`. Do not assume those operations cancel an already running Session—inspect the run and request the appropriate cancellation if authorized.

## Resource queries, parameters and cleanup

- File/Skill/Vault list/get tools remain to discover existing resources and bind their IDs. Forward directory/ownership checks still apply; do not substitute Managed reads/writes after a Forward failure.
- `get_forward_file`, `get_forward_skill`, `get_forward_environment`, `get_forward_vault` take `id`. Environment update/delete also take `id`; `archive_forward_environment` specifically takes `environment_id`. Read the selected tool's schema, not a sibling's.
- Resource `page` values are opaque cursors. Forward results normally wrap lists in `item.data`; read pagination inside item. Use only the selected tool's declared filters, not removed search-tool arguments.
- File IDs are opaque existing IDs, not filenames or URLs. No byte upload, Drive URL generation or resource registration is available in this core set. Ask for missing files to be prepared in the application.
- Skill list/get returns IDs and latest-version information; use `include_content` only when needed. Follow latest or pin an actual known version; do not invent versions or call history/import tools.
- Vaults must already be configured. Bind `vault_ids` where declared; list/get does not prove credential validity. Vault mutations are outside this tool set.
- Configuration parameters remain intact, including existing resource references and Schedule execution/sinks. Their presence does not expose management of Identity, Channel, Memory or other referenced systems.
- On uncertain writes, inspect current state before retrying. For authorized cleanup, cancel active execution and inspect it, archive the selected Session/Schedule/Template, and archive a used Environment. Environment hard deletion is permanent and may fail on references; do not bypass Forward guards or use Managed deletion for a Forward Session.
