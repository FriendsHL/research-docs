# 用 MCP 构建可触达生产系统的 Agent

> 原文：[Building agents that reach production systems with MCP](https://claude.com/blog/building-agents-that-reach-production-systems-with-mcp)
> 发布日期：2026年4月22日
> 翻译日期：2026年5月7日

![MCP 与 Agent 集成](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/6903d22bed4b18b6703cd710_e750c875fbd7f08ffb6495efa180a8ed60de3611-1000x1000.svg)

## 概述

本文探讨了将 AI Agent 连接到外部系统的三种方式：直接调用 API、通过命令行工具（CLI）、以及使用 Model Context Protocol（MCP）。我们会逐一分析它们各自适合的场景、为什么生产级 Agent 越来越倾向 MCP，以及构建这类集成的有效模式。

---

## 三种连接方式

### 直接调用 API

Agent 可以在沙箱中直接通过 HTTP 请求调用 API，或通过 function-calling 工具完成调用。这种方式适用于"单 Agent 对单 Service"的场景，但当扩展开来时就会遇到 **M×N 集成问题**——每对 Agent 与服务的组合都需要单独定制集成。

### 命令行工具（CLI）

Agent 在 shell 中执行 CLI 工具。这种方式轻量、能复用已有工具链，但在缺少容器访问权限的移动端、Web 端和云托管平台上就难以为继。认证依赖凭据文件，因此最适合权限相对宽松的本地环境。

### Model Context Protocol（MCP）

MCP 提供了一个标准化的协议层，让 Agent 能够连接到暴露系统能力的服务器，自带认证、发现和丰富的语义。**一个远程服务器即可触达所有兼容客户端，覆盖任何部署环境。**

---

## 生产级 Agent 与云端部署

生产级 Agent 越来越多地运行在云端，以便扩展和持续运行。MCP SDK 的月下载量已经突破 **3 亿次**（年初这一数字还是 1 亿），在企业以及 Claude、ChatGPT、Cursor、VS Code 等平台中被广泛采用。

---

## 构建高效的 MCP Server

### 用远程 Server 实现最大覆盖

> "远程 Server……可以触达所有兼容客户端，覆盖任何部署环境。"

让你的能力能够分发到 Web、移动端和云端各类平台。

### 按意图组织工具，而非镜像 API

数量更少、描述更清晰的工具，比把整个 API 完整包装一遍效果要好得多。一个 `create_issue_from_thread` 工具，胜过多个原语级别的调用。

### 对大型 API 表面用代码编排

对于操作数量动辄上百的服务（AWS、Kubernetes），暴露一个轻薄的工具表面、接收代码作为输入。**Cloudflare 的 MCP Server 就是个典型——只用 2 个工具就覆盖了约 2,500 个 endpoint。**

### 支持丰富语义

MCP Apps 扩展支持交互式界面（图表、表单、看板）内联渲染。**Elicitation** 让 Server 能够暂停调用、通过表单或浏览器请求用户输入（用于 OAuth、付款等场景）。

### 标准化认证

OAuth 使用 **CIMD（Client ID Metadata Documents）**，能加速首次认证流程。Claude Managed Agents 中的 Vault 自动处理 token 存储与刷新。

---

## 上下文高效的 MCP 客户端

### Tool Search

不再一次性加载所有工具定义，而是允许 Agent 在运行时搜索工具目录。测试显示这能将工具定义占用的 token 减少 **85% 以上**，同时不影响选择的准确度。

### Programmatic Tool Calling

在代码执行沙箱中处理调用结果，而不是把原始结果直接返回给模型——在复杂工作流上能减少约 **37%** 的 token 使用。

---

## MCP Server 与 Skill 的整合

### Plugin 打包

Plugin 把 skill、MCP server、hook 和 LSP server 一起打包。Cowork 的数据 plugin 就把 10 个 skill 与 8 个 MCP server（Snowflake、Databricks、BigQuery 等）整合在一起。

### 从 Server 分发 Skill

服务提供商越来越多地把 skill 与 MCP server 一起发布。Canva、Notion、Sentry 已在 Claude 的目录中同时上架两者。一个实验性扩展支持直接从 Server 交付 skill，并随 API 一起做版本管理。

---

## 核心要点

> "随着生产级 Agent 迁移到云端，MCP 成为触达外部系统的关键层"

它跨平台地提供了认证、交互能力和标准化的语义。
