# Qoder Cloud Agents over MCP

Read the workflow selected in [the Skill entrypoint](../SKILL.md), then use this reference for tool arguments and result handling.

Use the installed connector's typed tools. Match logical names such as `list_models`, `create_session` and `create_forward_session` to the host's discovered callable names; do not guess a namespace. Verify needed tools and their live schemas before planning or calling. Pass the argument object defined by each tool's schema.

## Core scope and retained configuration

This tool set focuses on core resource lifecycle and execution. File, Skill and Vault list/get tools support discovery of existing resources; file attachments remain available.

Removing management tools does **not** remove configuration arguments. Keep supported `identity_id`, `resources`, `vault_ids`, `tools`, `mcp_servers`, `skills`, `multiagent`, environment configuration and Schedule delivery/execution options when the task needs them. Existing memory-store references may still be passed where declared; there is no memory-management workflow here.

Forward Session creation and Schedule creation/run listing still require the actual business `identity_id`. Obtain it from the user, approved application context, or an existing Forward Session/Schedule result. Use the actual business Identity rather than an inferred ID. If unavailable, ask for it or for setup in the QCA application; do not fabricate an ID, create an Identity through another interface, or silently switch to Managed.

No direct byte upload, file-content download tool or SSE stream is exposed. Use existing file IDs and bounded polling. If a prerequisite cannot be supplied through these tools, explain the gap and request the missing resource/setup.

## Calling and observing

- Use `list_models` for both layers. Choose enabled models and only their advertised efforts/context windows; omit a non-default window unless needed. Consider `price_factor` for recurring or expensive work.
- Reuse actual returned IDs. Check the selected tool's literal parameter names: Both layers use resource-specific IDs such as `agent_id`, `template_id`, `session_id` and `schedule_id`; inspect each selected tool rather than assuming generic `id`.
- Construct writes from the input schema, not entire read responses. Omitted fields and explicit nulls can have different meanings; supplied tool/server/Skill arrays replace existing arrays.
- A permissive schema is not proof the backend accepts every JSON value. For Template `files` and Schedule `execution`, follow the additional business constraints in [Forward workflows](forward-workflows.md).
- Managed lists use top-level `items`; Forward lists normally use `item.data`, with pagination inside `item`. Use small limits and returned cursors; concatenate data arrays rather than envelopes. Use at most one mutually exclusive pagination selector where required.
- Session messages in both layers use `{"type":"user.message","content":[{"type":"text","text":"..."}]}` inside `events`. Forward Schedule initial events instead use string `content`, with the same `user.message` type.
- Creation/submission is not completion. Poll the chosen layer's Session/run status with a deadline; inspect errors and final `agent.message` events. Stop on completion, failure, cancellation, deadline or user stop. There is no `assistant.message` final-event alias.
- Managed event summaries omit payloads; fetch filtered full events when output is needed. Forward has full event reads only. Limit payload size and avoid unnecessary thinking/tool-call output.
- On ambiguous writes, inspect existing state before retrying. A Forward `idempotency_key` is not proof every endpoint deduplicates writes.
- Check tool errors, `isError`, business status, error code and request ID.

## Resource lifecycle

Perform only mutations covered by the requested workflow. Creating/updating a Deployment or Schedule does not authorize an additional manual test run; execution consumes credits. Outcome evaluation is optional and may add paid iterations. Do not auto-approve tool confirmations.

Archive/delete only authorized exact objects. Managed Session deletion removes recorded events; Environment deletion is permanent and may fail when referenced. Archive used Environments instead. Agents, Deployments, Forward Templates/Sessions/Schedules have archive, not hard-delete tools here. Do not simulate missing deletion through another layer.

Verify Deployment/Schedule archival using a populated `archived_at`; their `status` may still be `active`. Forward Template uses `status:"archived"`; Forward Session archive may initially return `terminated` and a later read `archived`. Cancellation acknowledgement (`canceling`) is not completion: read back state/events, and do not wait for an invented persistent `canceled` status.

Report the chosen layer, affected IDs, actual final state and any error/request IDs. Distinguish saved configuration, accepted execution and completed output. Do not automatically clean up shared resources.
