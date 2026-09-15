# Live Sources

When you need details not covered in this skill's files, fetch the latest official documentation from these URLs.

> **Docs host**: official Cloud Agents documentation lives on the Aliyun help portal under `help.aliyun.com/zh/lingma/cloud-agents-cn/`.

---

## Console & Account

- **Cloud Agents 介绍页**: https://qoder.com.cn/cloud-agents
- **PAT 管理（首选）**: https://qoder.com.cn/cloud/pat-keys
- **PAT 管理（备选入口）**: https://qoder.com.cn/account/integrations

---

## Overview & Concepts

- **Overview**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/overview
- **Quickstart**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/quickstart

## User Guide

- **Define an agent**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/define-agent
- **Environments**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/environments
- **Sessions**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/sessions
- **Events**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/events
- **Files**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/files
- **Memory Stores**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/memory-stores
- **Skills**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/user-guide/skills

## API Reference

- **Agents**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/agents
- **Environments**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/environments
- **Sessions**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/sessions
- **Send Events**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/send-events
- **Stream Events (SSE)**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/stream-events
- **Files**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/files
- **Memory Stores**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/memory-stores
- **Skills**: https://help.aliyun.com/zh/lingma/cloud-agents-cn/api-reference/skills


## Contracts not indexed in the supplied CN package

The supplied CN documentation index does not include a dedicated SAT exchange, Forward, Models, Deployments or Vault API page. Start from the CN overview and navigate/search the official CN documentation for the required operation and schemas. Do not manufacture a CN URL by replacing the host of a Global documentation URL, and do not infer a REST route from an MCP tool name.

For Forward HTTP, obtain the current CN contract for Templates/Identities, Sessions/events or Schedules/runs before constructing requests. If the required contract cannot be obtained, report that specific gap instead of guessing or changing business layer. The maintained general workflow remains MCP-first, with HTTP when a matching MCP is absent; the source package's older HTTP examples do not override it.

Service API and SAT exchange endpoints are listed in [regional endpoints](../references/region.md). Documentation URLs identify the CN documentation source; this import is not a live endpoint or authentication test.
