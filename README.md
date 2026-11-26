# Codex Vibe Coding & Anthropic Agent 架构精译（双册全集）

> **在线阅读（GitHub Pages）**：https://huayintianlai.github.io/codex-agent-handbook/

一份面向独立开发者与 AI 工程实践者的双册离线/在线交互式技术电子书，将 **Codex 实战方法论** 与 **Anthropic 工程专栏 9 篇 Agent 架构核心长文全量中文精译** 融为一体，支持三套阅读主题（纸白 / 暖黄 / 深色）、三档响应式字号、`⌘K` 全文检索、代码块一键复制与官方原图点击全屏放大。

---

## 本站内容总览

### 第一册 · Codex Vibe Coding 实战（12 章 + 4 个交互实验室 + 操作卡库）
- **第一篇 · 搭台子与防打扰**：百行 `AGENTS.md` 地图、验收口径前置、`/plan` 与 `/goal` 放手长跑
- **第二篇 · 多线推进与复利**：上下文腐烂交互模拟器、Worktree 与多线并行、Skill 与自愈护栏沉淀
- **第三篇 · 锁死黑盒质量**：应用可读性与四项交付单、并发幂等交互模拟器、独立只读安全审查
- **第四篇 · 对抗熵增与复工**：6 个月代码熵增与每周 GC 推演实验室、老项目微创接管、全生命周期总纲

### 第二册 · Anthropic 工程专栏 Agent 架构精译全集（完整 9 篇 + 57 张官方原图）
全系列严格对照 Anthropic Engineering Blog 官方原文逐段忠实精译、统一 18 条核心工程术语口径，**像素级克隆并中文化了官方原文全部 57 张原始架构图、实验曲线与实拍截图**，并在每篇文末附有默认收起的「译者附注（非原文）」实战映射卡：

| 篇目 | 中文标题 | 官方原文（Anthropic Engineering） | 发布日期 | 官方原图 |
| :---: | :--- | :--- | :---: | :---: |
| **01** | 构建高效的智能体 | [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | 2024-12-19 | 8 张（中文化） |
| **02** | 与智能体协作编写高效工具 | [Writing effective tools for agents — with agents](https://www.anthropic.com/research/writing-tools-for-agents) | 2025-09-11 | 8 张（图表汉化 + 终端代码原图） |
| **03** | AI 智能体的高效上下文工程 | [Effective context engineering for AI agents](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents) | 2025-09-29 | 2 张（中文化） |
| **04** | 用 Agent Skills 为智能体赋能真实世界任务 | [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/research/equipping-agents-for-the-real-world-with-agent-skills) | 2025-10-16 | 6 张（中文化） |
| **05** | 结合代码执行与 MCP：构建更高效的智能体 | [Code execution with MCP: Building more efficient agents](https://www.anthropic.com/research/code-execution-with-mcp) | 2025-11-04 | 1 张（中文化） |
| **06** | 扩展托管智能体：将“大脑”与“手脚”解耦 | [Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/research/managed-agents) | 2026-04-08 | 4 张（中文化） |
| **07** | 揭秘 AI 智能体评估体系 | [Demystifying evals for AI agents](https://www.anthropic.com/research/demystifying-evals-for-ai-agents) | 2026-01-09 | 5 张（中文化） |
| **08** | 长程应用开发的执行框架设计 | [Harness design for long-running application development](https://www.anthropic.com/research/harness-design-long-running-apps) | 2026-03-24 | 8 张（官方实拍截图） |
| **09** | 智能体记忆的核心组件 | [The components of agent memory](https://www.anthropic.com/research/components-of-agent-memory) | 2026-11-06 | 15 张（中文化） |

---

## 仓库目录结构

- `index.html`：单文件完全自包含交互式网页（约 3.44 MB，内嵌全部 57 张高清 WebP 原图，下载即可离线双击阅读）
- `markdown/`：第二册导读术语表及全部 9 篇纯净 Markdown 源文件（可直接批量导入 NotebookLM、Obsidian 或任意知识库）

---

## 来源与版权声明

- 第一册内容整理参考自 `codex.99uwen.com`。
- 第二册 9 篇文章英文原文与原始图表版权均归 **Anthropic** 及对应原作者所有；本仓库为非官方中文学习精译与排版整理，每篇顶部均保留了完整署名与官方英文原文链接。
- 本仓库仅用于个人学习交流与技术研究，不用于商业用途；如原作者或版权方有任何异议，请提 Issue 告知，将第一时间处理或下架。
