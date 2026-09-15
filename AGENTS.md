# Public distribution content

- Maintain only two standalone Skills in `skills/qoder-cloud-agents` and `skills/qoder-cloud-agents-cn`. Do not add client-specific plugin directories or duplicate Skill copies.
- Preserve regional endpoints and credentials. Prefer matching regional MCP with PAT; use HTTP with PAT/SAT when MCP is absent or HTTP is requested. Do not retry uncertain writes across transports or regions.
- Keep connection, authentication and verification instructions in `mcp/README.md`; clients manage their own configuration format and installation.
- Maintain the two regional MCP Registry definitions under `mcp/`. Do not add build/packaging steps, ZIP archives or release.json metadata.
- Keep README user-facing. Check local links, Skill references, Registry JSON and git diff --check before committing. Never include credentials or personal settings.
- Do not push without explicit user authorization. Do not claim marketplace approval from file preparation.
