# 一位非技术背景的 PM 如何用 Claude Code 在 6 周内构建并发布了一款减压 App

> 原文：[How a non-technical project manager built and shipped a stress management app with Claude Code in six weeks](https://claude.com/blog/how-a-non-technical-project-manager-built-and-shipped-a-stress-management-app-with-claude-code-in-six-weeks)
> 发布日期：2026年5月1日
> 翻译日期：2026年5月7日

## 概述

Kostiantyn Vlasenko 是一位**没有任何编程经验**的项目经理，仅用六周多一点时间就用 Claude Code 构建并发布了 **Respiro**——一款减压 iOS App。这款 App 在 "Built With Opus 4.6" Claude Code 黑客松中获奖，目前已上架 Apple App Store，拥有数百名用户。

---

## 一个没有 App 能解决的问题

Vlasenko 是基辅 Mythical Games 的项目经理，有十年管理利益相关方压力和团队动态的经验。他一直在寻找能**实时探测压力信号**的减压工具。已有 App 提供的是按计划安排的呼吸练习，但他需要一种能识别他真正感到压力、并立即介入的东西。

这个想法是在乌克兰西部山区度假时成型的：构建一款 App，能从个人设备**实时探测压力信号**，并在真正需要的时刻通过引导式正念呼吸做出回应。

---

## 像管理人一样管理 Agent

Vlasenko 当时已经在用 Claude 自动化一些工作任务，比如更新 Jira、发 Slack。后来他发现了 Claude Code CLI。他的项目管理背景比编程能力更有用。

> "我有大量管理真实人的经验。我意识到这是一回事——只是把人换成了 IDE 里的 Agent。"

他最初的 prompt 其实很直白：构建一款能减压的应用。结果让他吃惊——一个能用的 App 几乎立刻就出现在他的手机上。

但他没有止步于此。Vlasenko 在 Claude 的帮助下研究了多 Agent 系统和 Apple API，最终构建了一套精致的架构，包括：

- **TCA architect agent**
- **Swift developer agent**
- **Metal specialist**
- **Code reviewer**
- 其他专门的 subagent（**总共 15+ 个**）

当基于 React Native 的原型因为没法测试 Android 而出现问题时，Claude **几小时内就帮他用 Swift 把整个 App 重写了一遍**——展示了这种工作方式无痛切换的能力。

**时间线：从概念到 App Store 可发布版本，仅用了不到六周。**

---

## 从 Claude Code 到 App Store

发布带来了新的挑战。Vlasenko 此前从未走过 Apple Developer Program 流程，但他**把 Claude 当成一步步的导览员**，甚至利用 Claude 的视觉能力分析截图。

> "卡住的时候，我可以直接截屏问 Claude：'嘿，我应该按哪里？'"

**第三方集成包括：**

- Sentry（日志）
- Amplitude（分析）

对于复杂配置，Vlasenko 表示：

> "**99% 的情况 Claude 一次就成功了。剩下的 1% 只意味着我需要把初始 prompt 写得更清楚一点。**"

Claude 的视觉能力尤其有价值：

> "你给 Claude 发一张截图让它分析，它会告诉你它看到了什么。然后你可以用这个让 Claude 引导你走完哪怕最复杂的 UX。"

### 营销与增长

Claude 的帮助远不止技术开发：

- **分析**：集成 Amplitude SDK，搭建完整的用户漏斗、留存指标、DAU/MAU 跟踪
- **内容创作**：博客文章、TikTok 内容
- **增长策略**：Claude 建议联系心理学家和正念从业者——这是 Vlasenko 没想到的策略，但效果显著

---

## "Founder Mode" 工作模式

这次经历改变了 Vlasenko 在 Mythical Games 的角色。**他现在作为日常工作的一部分直接 commit 代码、发布功能**，并在公司内部成为他自己的 Claude 辅助开发工作流的倡导者。

工程团队的反馈很积极：

> "他们大多数人回来都说：'嘿，这种工作流比我现在的方式好多了！'"

Mythical Games 现在有一个**完全通过 Claude Code 交付工作的小型内部团队**。

不过，采纳并非毫无阻力。Vlasenko 观察到：

> "**对一些工程师来说，要切换到不再完全控制每一行代码的心态很难。**对我来说很容易，因为我没有深厚的编程背景，这种工作方式对我来说很自然。"

---

## 当前与未来方向

Vlasenko 正在继续构建 Respiro 的下一个版本，按 Claude 辅助制定的 roadmap 工作，包括：

- 语音引导练习
- 动作练习

他在每天的正常工作之后，还会有 7–8 小时的编码 session：

> "我会说，**Claude Code 是我的新瘾**。"

---

## Claude Code Day Zero 问卷™

**谁或什么激励了你？**

LeBron James——他的纪律性、一致性和长期主义；他靠执行而不是仅靠天赋打造伟大。还有 Warren Buffett，代表着相似的核心特质：耐心、一致性、跨数十年的清晰思考。

**如果可以回到过去，你会做哪些不同的事？为什么？**

更早意识到**"构建一款产品其实是简单的部分"**：

> "传达它的价值要难得多！今天，构建一个高质量的东西出乎意料地容易，尤其是有 Claude 这样的工具。但让人理解为什么这件事重要、为什么他们要在乎——这才是真正的挑战。"

**如果有人为你的创业旅程写一本书，书名会是什么？**

"Autonomous by Design"

**你的工作空间是什么样子？**

- 约 70% 的时间用 MacBook 接外接显示器、键盘和鼠标
- 约 20% 的时间在外面用笔记本
- 约 10% 的时间在手机上记下灵感
- 猫经常在桌边陪着，防止他过度连续工作

**你现在浏览器里开了多少个 tab？**

大约 5 个，刻意保持精简。通常会有一个后台播放的 YouTube 视频——真实犯罪类或创业者播客。同时积极维护一个 Threads 账号，分享构建进度、实验、Claude 工作流，帮助其他人建立自己的工作系统。
