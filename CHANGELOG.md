# Changelog

## 2026-09-15

- Keep only Global/CN standalone Skills and MCP Registry metadata; remove client-specific plugins and duplicate Skills.
- Merge connection and authentication documentation into `mcp/README.md`.

- Consolidate connection documentation around regional MCP endpoints, Streamable HTTP and PAT authentication; leave client configuration details to consumers.

- Add Global and CN MCP Registry server definitions with regional Streamable HTTP endpoints and required secret PAT inputs.

## 2026-09-14

- Expose standalone Skills directly as `skills/qoder-cloud-agents` and `skills/qoder-cloud-agents-cn`; keep the Qoder integrated Skill within Qoder plugins.

- Name Global plugins `qoder-cloud-agents` and CN plugins `qoder-cloud-agents-cn`, including directory names and client manifests.

- Provide complete plugin adapters for Global and CN.
- Include all required Skills and references directly in each plugin.
- Provide standalone Global/CN general Skills and the Qoder integrated Skill.
- Document MCP connections, regional authentication and HTTP API usage.
