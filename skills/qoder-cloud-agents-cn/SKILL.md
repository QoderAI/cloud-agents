---
name: qoder-cloud-agents-cn
description: Run and manage Qoder Cloud Agents CN tasks, conversations and schedules from third-party agents. Use available QCA MCP tools or the HTTP API.
---

# Qoder Cloud Agents — CN

## Choose MCP or HTTP

- Prefer the configured QCA MCP tools for the requested region/environment. Discover their actual names and argument schemas; see [MCP calling guide](references/mcp-guide.md).
- If MCP is absent or unconfigured, or the user requests HTTP, follow [HTTP setup and workflows](shared/guide.md). Connected MCP errors and uncertain writes do not trigger automatic HTTP retries.
- Use this edition's [regional endpoints](references/region.md). Keep the selected account, environment and resource region consistent.
- For MCP credential setup, read [MCP authentication](references/mcp-auth.md). HTTP uses separately configured local PAT/SAT credentials; a plugin PAT form does not automatically supply them. Never ask for tokens in chat.

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
