# OpenClaw 源码研究笔记（2026-03-23）

本文档汇总了 2026-03-23 对 OpenClaw 各核心模块的深度源码研究，包含以下子文档：

## 子文档列表

1. [[01-Memory模块源码解析]] - 向量检索 + 全文搜索的记忆系统
2. [[02-Session-Routing系统源码解析]] - SessionKey 生成与会话管理机制
3. [[03-Gateway模块源码解析]] - 系统核心入口，HTTP/WebSocket 双协议服务
4. [[04-Agent引擎深度解析]] - 基于 pi-mono 的 Agent 架构
5. [[05-Auto-Reply系统源码解析]] - 消息进来到 AI 回复出去的完整链路
6. [[06-ACP模块源码解析]] - Agent Communication Protocol
7. [[07-Hooks系统源码解析]] - Internal Hooks + Plugin Hooks
8. [[08-Cron与Heartbeat机制]] - 定时任务 & 心跳机制
9. [[09-Plugins与Channel-SDK]] - 插件与渠道扩展
10. [[10-ACP-Hooks-Cron-Plugins综合解析]] - 综合解析
11. [[11-pi-mono源码深度解析]] - Agent 核心底座

---

## 📊 架构图（Mermaid）

所有模块的可视化架构图见 [[diagrams/README|架构图目录]]：

| 图表 | 内容 |
|------|------|
| [[diagrams/Memory架构图]] | 整体架构、搜索流水线、MMR算法 |
| [[diagrams/Gateway架构图]] | 启动流程、WebSocket协议、健康监控 |
| [[diagrams/AutoReply架构图]] | 数据流、消息过滤、并发控制 |
| [[diagrams/Agent引擎架构图]] | 三层架构、Agent Loop、OAuth复用 |
| [[diagrams/Session-Routing架构图]] | SessionKey、Binding匹配 |
| [[diagrams/ACP-Hooks-Cron架构图]] | 通信流程、拦截能力 |
| [[diagrams/Plugins-Channel架构图]] | 注册能力、扩展层次 |

---

> 每个子文档包含完整的架构分析、源码解读和设计亮点总结。

---
*从学城同步：https://km.sankuai.com/collabpage/2752531767*
*同步时间：2026-03-26*
