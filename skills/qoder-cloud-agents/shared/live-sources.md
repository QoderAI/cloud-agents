# Live Sources

When you need details not covered in this skill's files, fetch the latest official documentation from these URLs.

> **Access note**: `docs.qoder.com` may return 403 for non-browser user agents (e.g., WebFetch/curl). If WebFetch fails, use Browser MCP (`navigate` + `get_page_text`) as fallback to retrieve the content.

---

## Overview & Concepts

- **Overview**: https://docs.qoder.com/cloud-agents/overview
- **Quickstart**: https://docs.qoder.com/cloud-agents/quickstart
- **Authentication (PAT + Service Account Token)**: https://docs.qoder.com/cloud-agents/api/conventions/authentication
- **Agent tools / Browser Use (Beta)**: https://docs.qoder.com/cloud-agents/tools
- **Container reference** (runtime image, file persistence): https://docs.qoder.com/cloud-agents/container-reference
- **Permission policies**: https://docs.qoder.com/cloud-agents/permission-policies
- **Access GitHub**: https://docs.qoder.com/cloud-agents/github-repositories
- **Webhooks**: https://docs.qoder.com/cloud-agents/webhooks
- **Vaults**: https://docs.qoder.com/cloud-agents/vaults

## Managed HTTP API Reference

- **Agents**: https://docs.qoder.com/cloud-agents/api/agents/list
- **Agent schemas (tools / MCP / skills / model)**: https://docs.qoder.com/cloud-agents/api/agents/schemas
- **Models**: https://docs.qoder.com/cloud-agents/api/models/list
- **Environments**: https://docs.qoder.com/cloud-agents/api/environments/list
- **Sessions**: https://docs.qoder.com/cloud-agents/api/sessions/list
- **Session schemas (event types, delta frames)**: https://docs.qoder.com/cloud-agents/api/sessions/schemas
- **Send Events**: https://docs.qoder.com/cloud-agents/api/sessions/send-event
- **Stream Events (SSE)**: https://docs.qoder.com/cloud-agents/api/sessions/stream-events
- **Deployments**: https://docs.qoder.com/cloud-agents/api/deployments/list
- **Files**: https://docs.qoder.com/cloud-agents/api/files/list
- **Memory Stores**: https://docs.qoder.com/cloud-agents/api/memory-stores/list
- **Skills**: https://docs.qoder.com/cloud-agents/api/skills/list
- **Vaults API**: https://docs.qoder.com/cloud-agents/api/vaults/list

> If a URL 404s, the docs may have been reorganized — start from the Overview page and navigate, or search the docs site.

## Documentation Index

The canonical current-page index is [llms.txt](https://docs.qoder.com/llms.txt). Use it to find a replacement when a page is reorganized.

## Forward HTTP contracts

Read the current operation and linked schemas, not the MCP argument catalog:

- [Create Forward Session](https://docs.qoder.com/cloud-agents/api/forward/sessions/create.md)
- [Send Forward events](https://docs.qoder.com/cloud-agents/api/forward/sessions/send-events.md)
- [List Forward events](https://docs.qoder.com/cloud-agents/api/forward/sessions/list-events.md)
- [Create Forward Schedule](https://docs.qoder.com/cloud-agents/api/forward/schedules/create.md)
- [List Forward Schedule runs](https://docs.qoder.com/cloud-agents/api/forward/schedule-runs/list.md)
- [Forward Environment schema](https://docs.qoder.com/cloud-agents/api/forward/environments/schemas.md)
- [Forward Identities](https://docs.qoder.com/cloud-agents/api/forward/identities/list.md)
- Use the documentation index above for Templates and any other operation.
- [SSE initial connection change](https://docs.qoder.com/cloud-agents/sse-initial-connection-behavior-change)
