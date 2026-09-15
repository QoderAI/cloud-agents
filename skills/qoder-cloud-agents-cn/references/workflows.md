# MCP Workflows for Qoder Cloud Agents

These examples show logical Tool names and argument objects. The MCP host may prefix Tool names. Replace placeholder IDs with values returned by earlier calls and inspect the live input schema before calling.

## Layer selection

Use Managed for explicit Agent/Environment execution without a business Identity, and Deployment for standalone recurring jobs. Use [Forward workflows](forward-workflows.md) for an existing business Identity and Template or a Forward-owned Session/Schedule. Do not cross layers on permission or reference failures.

## Create an Agent and complete one turn

### 1. Select a model

Call `list_models` with `{}`. Choose an item where `is_enabled` is true. Use only that item's advertised `effort` and `available_context_windows` values. Omit `context_window` to use the default unless the task genuinely needs a different advertised window. For recurring or potentially expensive work, compare `price_factor` and make the quality/cost choice visible to the user.

### 2. Create an Environment

Call `create_environment`:

```json
{
  "name": "analysis-environment"
}
```

This is the minimum create request. Add `config` only when the task needs a specific type, package installation, or setup script; inspect the live schema for those optional fields.

### 3. Create an Agent

When the Agent needs an existing Skill, call `list_skills` first. Use `get_skill` to inspect its latest version. Omit the binding version to follow latest, or pin a known version obtained from approved context; version-history tools are not exposed.

Call `create_agent`:

```json
{
  "name": "repository-analyst",
  "model": "<model id from list_models>",
  "system": "Inspect the mounted repository and answer with evidence. Do not modify files unless asked.",
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "enabled_tools": ["Read", "Glob", "Grep", "Bash", "DeliverArtifacts"],
      "default_config": {
        "permission_policy": {"type": "always_ask"}
      },
      "configs": [
        {"name": "Read", "permission_policy": {"type": "always_allow"}},
        {"name": "Glob", "permission_policy": {"type": "always_allow"}},
        {"name": "Grep", "permission_policy": {"type": "always_allow"}}
      ]
    },
    {"type": "browser_toolset_20260714"}
  ],
  "skills": [
    {"type": "custom", "skill_id": "skill_..."}
  ],
  "metadata": {"purpose": "repository-analysis"}
}
```

`default_config` sets the baseline and `configs[]` supplies per-tool overrides. If `enabled_tools` is non-empty, include every built-in tool the task needs. Include `DeliverArtifacts` when the user expects downloadable output files. Keep the Browser Use entry only when browser automation or live preview is needed. Omit `skills` when no existing Skill is needed.

### 4. Create a Session

This creates the Session without sending the first message. The published `create_session` input has no `events` or `initial_events` argument; send messages separately in step 5 even if the discovered description suggests otherwise.

Call `create_session`:

```json
{
  "agent": "agent_...",
  "environment_id": "env_...",
  "title": "Analyze repository",
  "environment_variables": {
    "REPORT_FORMAT": "markdown"
  },
  "metadata": {"purpose": "repository-analysis"}
}
```

To attach an existing file:

```json
{
  "type": "file",
  "file_id": "file_...",
  "mount_path": "/data/input.txt"
}
```

To attach a repository, obtain its token through an approved secure channel and never echo it:

```json
{
  "type": "github_repository",
  "url": "https://github.com/example/project.git",
  "authorization_token": "<secure value>",
  "mount_path": "/workspace/project",
  "checkout": {"type":"branch","name":"main"}
}
```

### 5. Send the message

Call `send_session_events`:

```json
{
  "session_id": "sess_...",
  "events": [
    {
      "type": "user.message",
      "content": [
        {"type":"text","text":"Analyze the repository and summarize the main risks with file evidence."}
      ]
    }
  ]
}
```

### 6. Poll for completion

There is no SSE Tool. Use a bounded polling loop:

1. Call `get_session` and inspect `item.status`.
2. Call `list_session_event_summaries` with `session_id`, `order: "asc"`, `limit: 100`, and `after_id` only when a prior event ID exists. Read `items[].type` without pulling payloads, and advance the cursor to a non-empty `last_id`.
3. When a summary shows an event whose payload is needed, call `list_session_events` with a narrow `types` filter and small limit. Final Agent output uses `agent.message`, not `assistant.message`.
4. Stop at `idle` or `terminated`, then fetch the latest final output with `types:["agent.message"]`, `order:"desc"`, and `limit:1`.
5. Use a bounded delay and deadline. Do not hammer the Tool, use long fixed sleeps, or retry a write while waiting.

If the Session requests tool confirmation, show the requested tool/action. After authorization, answer with:

```json
{
  "session_id": "sess_...",
  "events": [
    {
      "type": "user.tool_confirmation",
      "tool_use_id": "<tool use id>",
      "result": "allow"
    }
  ]
}
```

For denial, use `result: "deny"` and optionally `deny_message`.

## List many Sessions safely

`list_sessions` returns compact summaries. Start with a filtered request and `limit: 20` or less, follow `next_page` only when the user needs more records, and call `get_session` for full details.

When aggregating pages, concatenate only their `items` arrays. Do not merge the response objects themselves: object merging can replace an earlier page's `items` with the final page.

## Add Browser Use to one existing Session

`update_session.agent.tools` changes only that Session's Agent snapshot. It does not update the reusable Agent resource, and the submitted `tools` array is a full replacement.

1. Call `get_session` and retain every existing `item.agent.tools` entry that must remain.
2. Add the Browser Use entry once.
3. Call `update_session`:

```json
{
  "session_id": "sess_...",
  "agent": {
    "tools": [
      {
        "type": "agent_toolset_20260401",
        "enabled_tools": ["Read", "Glob", "Grep", "Bash"]
      },
      {"type": "browser_toolset_20260714"}
    ]
  }
}
```

If QCA reports `feature_not_available`, report it and keep the requested Session configuration visible; do not silently remove Browser Use.

## Update an Agent safely

1. Call `get_agent` and keep `item.version` plus all array fields that must remain.
2. Call `update_agent` with `agent_id`, that exact version, and only intended scalar changes.
3. If changing `tools`, `mcp_servers`, or `skills`, send the complete replacement array, including entries to preserve.
4. On a version conflict, get the Agent again and ask before overwriting changes that were made concurrently.

Example:

```json
{
  "agent_id": "agent_...",
  "version": 3,
  "description": "Updated analysis instructions"
}
```

## Use an existing file

There is no file upload or Drive tool in the core surface. Use a file already available to the caller, or ask for it to be prepared through the QCA application.

1. Find an existing file with `list_files` or validate a known ID with `get_file`.
2. Inspect file readiness through metadata; download-link generation is temporarily unavailable.
3. Attach it during `create_session`, or call `add_session_resource` for an existing Session:

```json
{
  "session_id": "sess_...",
  "type": "file",
  "file_id": "file_...",
  "mount_path": "/data/input.txt"
}
```

4. Attachments remain supported; do not claim file bytes were downloaded. Direct the user to an authorized file-access interface if content transfer is required.

## Create and observe a Deployment

Call `create_deployment` for a scheduled job:

```json
{
  "name": "daily-status",
  "agent": "agent_...",
  "environment_id": "env_...",
  "schedule": {
    "type": "cron",
    "expression": "0 9 * * *",
    "timezone": "Asia/Shanghai"
  },
  "initial_events": [
    {
      "type": "user.message",
      "content": [
        {"type":"text","text":"Generate the daily status report and deliver it as an artifact."}
      ]
    }
  ],
  "environment_variables": "REPORT_FORMAT=markdown;REPORT_LANGUAGE=zh-CN",
  "metadata": {"purpose":"daily-status"}
}
```

Omit `schedule` for a manual-only Deployment. `user.define_outcome` is optional; add it only when iterative evaluation is part of the requested workflow and the user accepts its additional model cost.

Creating or updating a Deployment alone does not imply permission to test-run it. `run_deployment` starts a real Agent Session and consumes model credits. It is authorized when the user asks to run or validate the Deployment; otherwise obtain approval after stating the expected cost drivers.

Use the exact lifecycle argument shapes:

```json
{"deployment_id":"dep_..."}
```

The object above is the complete input for `run_deployment`, `pause_deployment`, `unpause_deployment`, and `archive_deployment`. Never rename the property to `jobId`.

```json
{"run_id":"drun_..."}
```

The object above is the complete input for `get_deployment_run`; do not add `deployment_id`.

To observe a run:

1. Use the `item.id` returned by `run_deployment`, or call `list_deployment_runs` with `deployment_id`.
2. Call `get_deployment_run`; read `item.session_id` and `item.error`.
3. When `session_id` exists, use `get_session` and `list_session_events` as in the interactive polling workflow.
4. Pause, unpause, or archive only when requested. Archive stops future scheduled execution. Read back `archived_at` to confirm archival; `status` can remain `active`.

## Cleanup disposable resources

Only clean resources explicitly established as disposable or covered by the user's cleanup request.

Recommended order:

1. If a Session is running, call `cancel_session` and poll until it settles. `canceling` acknowledges the request; an already idle Session can remain `idle`. Use state/events, not a required `canceled` status, to determine whether execution has stopped.
2. Call `delete_session` only if permanent event deletion is intended; otherwise `archive_session`.
3. Archive the Agent with `archive_agent`.
4. Archive a used Environment with `archive_environment`; do not expect `delete_environment` to work after a Session has referenced it.
5. Archive test Deployments before archiving their Agent.

## Bind existing resources without managing them

File, Skill and Vault list/get tools remain for discovery. Use `vault_ids` on Session/Deployment creation for already configured Vaults; do not claim that reading a Vault proves every credential is valid.

Existing memory stores can still be referenced in `create_session.resources` or Deployment resources when their IDs are supplied:

```json
{"type":"memory_store","memory_store_id":"memstore_existing","access":"read_write","instructions":"Keep project decisions concise."}
```

This is a retained binding parameter, not permission or a capability to create/manage stores. Session creation also accepts file/repository resources; `add_session_resource` attaches only a file later. `get_session_resource` and `list_session_resources` inspect mounts; `delete_session_resource` unmounts the selected attachment, not the source file. Repository-token rotation is outside the core tool surface.

Agent `skills` entries require `type:"qoder"` or `"custom"` and `skill_id`; optional version is a numeric string or `"latest"`. MCP server objects require `type:"url"`, `name` and `url`; their names are referenced by `mcp_toolset.mcp_server_name`. Preserve omitted fields and construct complete replacement arrays for intended changes.
