# QCA HTTP API guide

Use when no QCA MCP is available, or the user explicitly requests HTTP API work. These references incorporate the supplied older HTTP Skill; authentication and SSE startup were refreshed on 2026-09-12. Other examples are reference snapshots, not proof that every endpoint/field remains supported in every deployment. For missing or changed details, read [live sources](live-sources.md) and the target operation's current contract before executing.

Read [regional endpoints](../references/region.md) first; use this edition's defaults only when no explicit matching configuration exists.

## Authentication

Use existing secure local configuration. `QODER_ACCESS_TOKEN` contains a raw PAT or SAT without the Bearer prefix. If the user already configured PAT, QODER_PAT or QCA_MCP_PAT for the intended HTTP account/environment, reference that variable locally without printing it. Do not assume a token intended for another host/region is valid here. If missing, direct the user to Qoder's Personal Access Tokens settings (https://qoder.com/cloud/pat-keys) and ask them to set it locally.

Global production defaults:

```bash
export QCA_HTTP_BASE_URL="${QCA_HTTP_BASE_URL:-https://api.qoder.com/api/v1/cloud}"
export QCA_FORWARD_BASE_URL="${QCA_FORWARD_BASE_URL:-https://api.qoder.com/api/v1/forward}"
export QODER_OPENAPI_BASE_URL="${QODER_OPENAPI_BASE_URL:-https://openapi.qoder.sh}"
: "${QODER_ACCESS_TOKEN:?Configure a PAT or SAT securely first}"
```

Respect an already selected region/environment. These HTTP bases are not MCP server URLs; do not append REST paths to `/mcp/servers/qca` or silently switch TEST to production. If the requested deployment is unclear, resolve it before mutations.

PAT authenticates a user. For service/CI integrations, use the SAT exchange in [API authentication](api-reference.md#authentication), with an exchange origin matching the business region/environment. An SA Key must never be sent directly to business APIs. No Job Token exchange is required for the documented direct QCA PAT route; do not import AppHub-specific auth assumptions.

## Managed HTTP workflow

1. Read [Managed API reference](api-reference.md) for the required operation. Discover models with GET /models and reuse or create a suitable Environment and Agent.
2. POST /sessions with `agent` and `environment_id`. HTTP uses `agent`, not the MCP argument name `agent_id`.
3. POST /sessions/{id}/events with `{"events":[{"type":"user.message","content":[{"type":"text","text":"..."}]}]}`.
4. Use [SSE/events](events.md) and [client patterns](client-patterns.md) to observe the target turn. Without Last-Event-ID, SSE begins at the current tail; recover history via List Events. requires_action is paused, not successful completion.
5. For recurring jobs use Deployments; only run one immediately when requested. For files, Skills, Vaults and tool config, read [tools and resources](tools-and-resources.md). Use [curl recipes](../curl/recipes.md) selectively.

HTTP lists normally use `data` and pagination fields, not MCP's `items` or `item.data`. JSON bodies need Content-Type application/json; multipart file/Skill uploads use the client's generated boundary. Construct JSON with a serializer, not string interpolation. Check status/errors before extracting returned IDs.

The older HTTP examples include networking fields not exposed by the current MCP Environment schema. Do not transplant them into MCP calls. For HTTP environment creation, verify the target deployment's schema before using legacy networking settings. Honor requested network restrictions.

## Forward HTTP workflow

The imported legacy API catalog is Managed-only. Forward uses `/api/v1/forward`, with its own contracts; never derive a route by replacing an MCP tool-name prefix or send a Managed recipe to a Forward resource.

Read the relevant official operation through [live sources](live-sources.md) before constructing a Forward request: Template/Environment setup, Session creation and events, or Schedule/run management. Preserve the business Identity and Template bindings. Use supported identity_id from user/application context or documented reads; only create an Identity when the task includes that setup. If the required HTTP contract cannot be obtained, report that gap instead of guessing an endpoint or switching to Managed.

## HTTP-specific resources

HTTP may expose upload/download, Skill/Vault management, Memory and other APIs outside the core MCP tool set. Use them only when selected HTTP mode and the requested task call for them. Attach large inputs as files instead of embedding datasets in messages. Download artifacts using returned signed URLs without forwarding QCA Authorization headers to storage hosts.

An Agent's `mcp_servers` and Vaults configure downstream cloud execution, not this host's connection to QCA. Credentials for those servers are distinct from the QCA PAT/SAT.
