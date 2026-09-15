# 内容维护

- `skills/qoder-cloud-agents/` 与 `skills/qoder-cloud-agents-cn/` 是两套 Skill 的唯一维护位置。保留 Global/CN 地址和鉴权差异，不增加客户端专用副本。
- MCP 服务地址、连接方式、鉴权和验证说明集中维护在 `mcp/README.md`；两份 `server.json` 与对应区域保持一致。
- README 面向用户介绍产品、内容和入口；客户端配置格式与安装方式由使用方自行处理。
- 修改后检查本地链接、Skill 引用及 Registry 元数据格式，并更新 CHANGELOG。
- 不提交真实凭据、个人配置、构建工具或压缩制品。
