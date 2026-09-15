# MCP credential setup

Use the endpoint in [regional endpoints](region.md) and the client's supported local credential settings. Reuse an existing configured connection.

1. Obtain a Global PAT from https://qoder.com/cloud/pat-keys if needed.
2. Configure the client to send `Authorization: Bearer <PAT>` on MCP requests. The stored PAT is the raw token without the Bearer prefix; environment-variable substitution syntax depends on the client.
3. Discover QCA tools and call their declared arguments. PAT and authentication headers belong in connection settings, not business tool arguments.

For 401/403, check the configured token, permissions and environment. Do not interpret an authentication failure as missing MCP or replay an uncertain write over HTTP.

MCP credential settings configure MCP only. HTTP requires its own local credential source described in [HTTP setup](../shared/guide.md); do not extract tokens from private client storage. HTTP PAT/SAT support does not imply that the MCP endpoint accepts SAT. Keep credentials out of chat, Skill files and Git.
