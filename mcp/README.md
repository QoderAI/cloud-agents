# Qoder Cloud Agents 接入指南

QCA 通过远程 MCP 提供云端 Agent 工具，也支持直接调用 HTTP API。本页集中说明区域地址、连接参数、鉴权及验证步骤。

## 服务地址

| 用途 | Global | CN |
| --- | --- | --- |
| QCA MCP | `https://mcp.qoder.com/api/v1/mcp/servers/qca` | `https://mcp.qoder.cn/api/v1/mcp/servers/qca` |
| Managed HTTP | `https://api.qoder.com/api/v1/cloud` | `https://api.qoder.com.cn/api/v1/cloud` |
| Forward HTTP | `https://api.qoder.com/api/v1/forward` | `https://api.qoder.com.cn/api/v1/forward` |
| SA Key → SAT | `https://openapi.qoder.sh/api/v1/serviceToken/exchange` | `https://openapi.qoder.com.cn/api/v1/serviceToken/exchange` |
| PAT 管理 | `https://qoder.com/cloud/pat-keys` | `https://qoder.com.cn/cloud/pat-keys` |

默认版本连接 Global，`-cn` 版本连接 CN。账号、凭证与资源必须属于同一区域；文档语言不决定服务区域。

## 配置 MCP

| 参数 | 配置 |
| --- | --- |
| 传输方式 | Streamable HTTP |
| URL | 上表中所属区域的 QCA MCP 完整地址 |
| 鉴权方式 | 静态 Bearer Token，使用同区域 PAT |
| 请求头 | `Authorization: Bearer <PAT>` |
| 连接名称 | 可自定义，例如 `qca`；同时连接两区时使用不同名称 |

在客户端 MCP 设置中填写这些参数。具体文件格式、变量引用语法、启用方式和凭证存储由客户端决定。`<PAT>` 是占位符；公开配置中不包含真实令牌。

若客户端提供 Bearer Token 输入框，只填写原始 PAT；若填写完整请求头，则包含 `Bearer ` 前缀。不要重复添加前缀，也不要假设所有客户端都支持 `${pat}` 替换语法。

## 验证连接

1. 启用连接，完成 MCP 初始化和工具发现。
2. 确认返回 QCA 工具列表。
3. 调用 `list_models` 或 `list_agents`，检查返回内容、业务状态和 `isError`。

工具名称可能带客户端命名空间，实际名称和参数以工具发现结果为准。鉴权信息属于连接配置，不属于工具参数。仅建立连接不能证明业务调用成功。

遇到 401/403，检查令牌有效性、区域及账号权限。不要因调用失败切换区域，或通过另一种传输重放结果未知的写操作。

## 使用 HTTP API

直接调用 HTTP API 支持同区域的 PAT 或 Service Account Token（SAT）：

```http
Authorization: Bearer <PAT_OR_SAT>
```

Service Account API Key（SA Key）先通过同区域的令牌交换接口获取 SAT，不能直接作为业务 Bearer Token。HTTP 的 PAT/SAT 支持不代表 MCP 支持 SAT。

HTTP 凭证需要单独在本地配置，不会自动继承 MCP 的凭证。请求示例和接口说明见 [Global Skill](../skills/qoder-cloud-agents/) 或 [CN Skill](../skills/qoder-cloud-agents-cn/)，完整 API 使用流程见[官方快速开始](https://docs.qoder.com/cloud-agents/quickstart)。

## 配合 Skill 使用

通用 Skill 优先使用对应区域 MCP；未配置该 MCP，或明确要求 HTTP 时，使用 HTTP API。缺少单个工具、401/403、业务失败或超时不等于 MCP 未配置。

## Registry metadata

| Region | Server definition | Registry name |
| --- | --- | --- |
| Global | [server.json](qoder-cloud-agents/server.json) | `io.github.QoderAI/qoder-cloud-agents` |
| CN | [server.json](qoder-cloud-agents-cn/server.json) | `io.github.QoderAI/qoder-cloud-agents-cn` |

These definitions describe the hosted MCP endpoints and required PAT input using the [official MCP Registry format](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/generic-server-json.md). The client substitutes the user's raw PAT into `Bearer {pat}`; both the input and resulting header are marked secret. No token is included in these files.

`websiteUrl` points to this public connection guide. The optional `repository` field is omitted because this repository contains integration content, not the hosted MCP implementation's source code. The initial metadata version is `0.6.0`.

Adding these files does not publish a Registry listing. Publication requires verification of the `io.github.QoderAI` namespace and a separate Registry submission; see the [official publishing guide](https://github.com/modelcontextprotocol/registry/blob/main/docs/modelcontextprotocol-io/quickstart.mdx).
