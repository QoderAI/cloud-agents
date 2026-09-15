# CN region

| Purpose | Production address |
| --- | --- |
| MCP | `https://mcp.qoder.cn/api/v1/mcp/servers/qca` |
| Managed HTTP | `https://api.qoder.com.cn/api/v1/cloud` |
| Forward HTTP | `https://api.qoder.com.cn/api/v1/forward` |
| SAT exchange | `https://openapi.qoder.com.cn/api/v1/serviceToken/exchange` |
| PAT settings | `https://qoder.com.cn/cloud/pat-keys` |
| PAT alternative | `https://qoder.com.cn/account/integrations` |
| Product | `https://qoder.com.cn/cloud-agents` |
| Documentation | `https://help.aliyun.com/zh/lingma/cloud-agents-cn` |

Use credentials, resources and endpoints from the same region/account/environment. Do not infer region from language, aliases, documentation language or token prefixes.

Preserve explicitly selected TEST/gray configurations; never carry their credentials or routing headers into production defaults. If connection region is ambiguous, resolve it before invoking tools. Do not try credentials against a different region after an authentication failure. Never repeat an uncertain MCP write through HTTP.

QCA MCP requires static bearer authentication: the user supplies a matching regional PAT through local plugin/client configuration; requests carry `Authorization: Bearer <PAT>`. Host login is not a substitute for this PAT. Direct QCA HTTP accepts PAT or SAT. Exchange an SA Key only at the matching regional exchange origin; never use the key directly for business APIs.

These endpoint values come from the supplied regional packages and existing regional configuration. They are not proof of live authentication or parity of every deployed operation. Use the selected operation's current contract.
