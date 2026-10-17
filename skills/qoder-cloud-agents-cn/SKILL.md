---
name: qoder-cloud-agents-cn
description: Use Qoder Cloud Agents to create and configure cloud agents, run tasks, continue conversations, and automate work on a schedule. Manage environments and resources, and track progress and results.
---

# Qoder Cloud Agents — CN

## Choose MCP or HTTP

- Prefer the configured QCA MCP tools for the requested region/environment. Discover their actual names and argument schemas; see [MCP calling guide](references/mcp-guide.md).
- If MCP is absent or unconfigured, or the user requests HTTP, follow [HTTP setup and workflows](shared/guide.md). Connected MCP errors and uncertain writes do not trigger automatic HTTP retries.
- Use this edition's [regional endpoints](references/region.md). Keep the selected account, environment and resource region consistent.
- For MCP credential setup, read [MCP authentication](references/mcp-auth.md). HTTP uses separately configured local PAT/SAT credentials; a plugin PAT form does not automatically supply them. Never ask for tokens in chat.

## Select a personal or enterprise workspace

Managed/CAS MCP tools can operate in personal spaces and enterprise spaces the signed-in user is authorized to access. Before selecting a space, check the live schemas for `list_accessible_workspaces` and workspace parameters; older installations may not expose them. If enterprise selection is unavailable, report that limitation instead of silently operating in the personal space.

Call `list_accessible_workspaces` with `{}` when discovering or changing the target space. Its `items` contain `workspace_id`, `name`, `type`, `organization_id` (enterprise only), `can_write`, and `is_default`. CAS discovers spaces from the authenticated user; no organization argument is needed. Results include the personal space and ordinary enterprise spaces available to organization administrators; non-administrators receive only their personal space. Enterprise listing initializes a default enterprise space when needed. Internal business spaces are excluded. Reuse returned IDs; do not infer an organization ID from a workspace ID.

| Target | Managed/CAS tool arguments | Forward tool arguments |
| --- | --- | --- |
| Default personal space | Omit `workspace_id` | Use existing arguments; no workspace selector |
| Selected personal space | Pass its `workspace_id` | Only default personal access is supported |
| Enterprise space | Pass its `workspace_id`; no `organization_id` needed | Not supported; do not call Forward tools for enterprise operations |

Managed/CAS selection is per call, not a persistent switch. Forward tools do not accept `workspace_id` or `organization_id` and remain in the personal space. Keep the same target on subsequent resource discovery, creation, updates and status/event polling. Honor the user's selected space; ask which one when the target is ambiguous. For writes, check `can_write`; the backend still authorizes each operation and can reject stale permissions. Do not retry a denied enterprise operation in the personal space.

These are MCP argument fields, not credentials or HTTP business-body fields. See [MCP calling guide](references/mcp-guide.md#workspace-argument-examples) for examples.

## Choose the workflow

| Task | Workflow |
| --- | --- |
| Custom Agent or one-off cloud task | Managed: Agent + Environment → Session |
| Agent running on demand or on a schedule | Managed Deployment |
| Business assistant conversation | Forward: Template + Identity → Session |
| Recurring Template task for an Identity | Forward Schedule |

Keep resources in their original business layer. Forward needs an actual `identity_id` from user/application context or existing resources.

## Run and observe

1. Discover supported models and options. Reuse suitable Agent/Template and Environment resources, or create them as needed.
2. Create a Session in the selected layer and send the user's message separately.
3. Observe status and final `agent.message` output using MCP event tools or HTTP SSE/polling. Stop on completion, failure, cancellation, required input or a deadline.
4. Report resource IDs, actual state/output and relevant error/request IDs. An accepted message is not a completed task.

Inspect existing configuration before updates and preserve requested tools, Skills and resource bindings when replacing arrays. Inspect state before retrying uncertain writes. Creating a scheduled task does not itself request an immediate run; cleanup applies to the resources the user selected.

## Read as needed

| Need | Reference |
| --- | --- |
| Managed MCP examples and arguments | [Workflows](references/workflows.md), [tool catalog](references/tool-catalog.md) |
| Forward MCP examples and arguments | [Workflows](references/forward-workflows.md), [tool catalog](references/forward-tool-catalog.md) |
| HTTP operation contracts | [API reference](shared/api-reference.md); [live sources](shared/live-sources.md) for Forward contracts |
| HTTP events, completion and reconnect | [Events](shared/events.md), [client patterns](shared/client-patterns.md) |
| HTTP files, Skills, Vaults and execution config | [Tools and resources](shared/tools-and-resources.md) |
| HTTP request examples | [curl recipes](curl/recipes.md) |

MCP argument/result schemas and HTTP routes/bodies are separate contracts. Use the reference for the selected calling method rather than translating tool names into API paths.
