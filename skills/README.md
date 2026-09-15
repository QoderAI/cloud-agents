# Qoder Cloud Agents Skills

本仓库只维护两套独立 Skill：

- [Qoder Cloud Agents](qoder-cloud-agents/SKILL.md)：默认 Global 版本。
- [Qoder Cloud Agents CN](qoder-cloud-agents-cn/SKILL.md)：CN 版本，使用对应区域的服务地址和凭证。

将选定的整个目录放入客户端支持的 Skills 位置，保留引用文档和调用示例。具体安装方式由使用方按客户端说明处理。

Skill 优先使用对应区域的 MCP（PAT 鉴权）；没有配置该 MCP，或明确要求 HTTP 时，使用 HTTP API（PAT/SAT 鉴权）。连接与凭证配置统一见[MCP 接入指南](../mcp/README.md)。
