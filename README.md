# Qoder Cloud Agents

**将 AI Agent 作为云服务，接入你的应用与工作流。**

[产品官网](https://qoder.com/zh/cloud-agents) · [官方文档](https://docs.qoder.com/cloud-agents/overview) · [快速开始](https://docs.qoder.com/cloud-agents/quickstart) · [Skills](skills/README.md) · [MCP](mcp/README.md)

## 什么是 QCA？

Qoder Cloud Agents（QCA）是面向企业开发者的云端 Agent Harness 全托管平台，以 Agent as a Service 提供生产级运行时。开发者可通过 API 将 Agent 嵌入现有产品与业务系统，通过 Skill、MCP 连接企业数据与工具，也可接入 IM 提供服务。QCA 托管任务编排、工具执行、会话状态与弹性调度，支持多智能体协同、跨会话记忆、定时与批量任务，并提供身份隔离、凭证管理、工具权限控制及运行观测。企业专注业务目标与交付标准，QCA 承接底层运行与规模运维，让 Agent 从一次调用走向持续服务。

你可以用 QCA：

- **执行研发任务**：将代码分析、重构或测试生成等任务交给云端 Agent，持续查看执行过程。
- **为应用接入 Agent**：通过 API 创建会话、发送任务和接收事件，将执行结果接入自己的产品流程。
- **自动化日常工作**：结合调度与业务系统，完成周期性检查、报告生成和批量处理。

一次典型的使用流程是：定义 Agent → 配置运行环境 → 创建会话 → 发送任务 → 获取进度与结果。Agent 描述“由谁来做、能用什么工具”，会话承载一次具体的任务执行。详见[核心概念与工作流程](https://docs.qoder.com/cloud-agents/overview)。

## 这个仓库提供什么？

本仓库是 Qoder Cloud Agents 的公开接入与使用内容入口，集中维护 两套独立 Skills、MCP 发布元数据、统一接入文档和更新记录。QCA 服务运行在云端；这里提供连接和使用它所需的内容。

| 内容 | 用途 | 入口 |
| --- | --- | --- |
| **Skills** | 指导 Agent 如何使用 QCA，包括工具选择、任务执行、进度查询和结果处理 | [Skills](skills/README.md) |
| **MCP** | 让支持 MCP 的客户端连接 QCA 的远程工具服务 | [MCP 接入](mcp/README.md) |
| **文档** | 说明服务地址、区域选择、鉴权与接入方式，并提供 HTTP API 使用参考 | [统一接入指南](mcp/README.md) |
| **Changelog** | 记录本仓库中 Skills 和接入内容的变化 | [更新记录](CHANGELOG.md) |

Skill 提供使用指导，MCP 提供可调用的工具。使用方按自己的客户端配置连接，也可以独立安装 Skill。

## 如何接入

### 在 Agent 客户端中使用

在支持远程 MCP 的客户端中添加 QCA 连接，选择所属区域的服务地址，使用 **Streamable HTTP** 传输，并配置 `Authorization: Bearer <PAT>` 请求头。连接参数与验证步骤统一见[MCP 接入指南](mcp/README.md)。配置文件格式、设置入口和凭证存储方式由所用客户端决定。

连接成功后，可以先执行只读操作：

> 使用 Qoder Cloud Agents，列出我当前的云端 Agents。

配合本仓库的 Skill，客户端可以按 QCA 工作流程选择工具、执行任务并获取结果。

### 单独使用 Skill

如果客户端支持 Skills，也可以直接安装独立 Skill。将选定的**整个目录**放入客户端支持的 Skills 位置，保留其中的引用文档和示例：

- **Global**：[skills/qoder-cloud-agents](skills/qoder-cloud-agents/)
- **CN**：[skills/qoder-cloud-agents-cn](skills/qoder-cloud-agents-cn/)

独立 Skill 优先使用已安装的对应区域 QCA MCP 工具，并通过 PAT 鉴权；没有对应 MCP，或你明确要求使用 HTTP 时，则按 Skill 中的说明使用 HTTP API，支持 PAT 或 Service Account Token（SAT）。HTTP 凭证需要单独配置，客户端的 MCP 凭证设置不会自动传给 HTTP 调用。

### 在应用或自动化流程中调用 API

如果你要将 QCA 集成到自己的应用、后端服务或自动化流程，可以直接调用 HTTP API，通过 PAT 或 SAT 鉴权。

从[官方快速开始](https://docs.qoder.com/cloud-agents/quickstart)了解如何创建 Agent、配置环境并运行第一个会话；本仓库还提供 [HTTP API 参考](skills/qoder-cloud-agents/shared/api-reference.md)和 [curl 调用示例](skills/qoder-cloud-agents/curl/recipes.md)。区域地址与凭证配置见[统一接入指南](mcp/README.md)。

## Global 与 CN 版本

本仓库统一使用以下命名：

- **`qoder-cloud-agents`**：默认版本，连接 Global 服务。
- **`qoder-cloud-agents-cn`**：CN 版本，连接 CN 服务。

请根据自己的账号所属服务区域选择版本，并使用同一区域的凭证。中文文档不代表 CN 服务；使用中文客户端也不影响区域选择。Skill 不会在调用失败时自动切换区域。

## 目录组织

```text
cloud-agents/
├── skills/                         可独立安装的 Skills
│   ├── qoder-cloud-agents/          Global Skill、API 参考与调用示例
│   └── qoder-cloud-agents-cn/       CN Skill、API 参考与调用示例
├── mcp/                            远程 MCP 接入说明与 Registry 元数据
│   ├── README.md                   统一接入指南
│   ├── qoder-cloud-agents/          Global server.json
│   └── qoder-cloud-agents-cn/       CN server.json
├── CHANGELOG.md                    本仓库更新记录
└── CONTRIBUTING.md                 内容贡献与维护说明
```

每个区域的 Skill 只维护一份，安装时保留整个 Skill 目录及引用文件。MCP 连接、鉴权和验证说明集中在 `mcp/README.md`。

## 文档与反馈

- [QCA 官方文档](https://docs.qoder.com/cloud-agents/overview)：产品概念、使用指南与 API 文档。
- [快速开始](https://docs.qoder.com/cloud-agents/quickstart)：运行第一个云端任务。
- [更新记录](CHANGELOG.md)：查看本仓库的内容变化。
- [GitHub Issues](https://github.com/QoderAI/cloud-agents/issues)：反馈 Skill、MCP 接入或仓库文档的问题。
- [贡献说明](CONTRIBUTING.md)：了解如何改进本仓库的公开内容。
