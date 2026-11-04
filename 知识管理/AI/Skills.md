---
title: AI Skills 收藏
tags:
  - ai
  - skills
  - 工具
---

# AI Skills 收藏

## 总览

| 名称 | 定位 | 仓库 |
| --- | --- | --- |
| Buildomator（原 gsd） | Claude Code 插件：计划 / 执行 / 验证的开发流程 | [jnuyens/gsd-plugin](https://github.com/jnuyens/gsd-plugin) |
| OpenSpec | 面向 AI 编码助手的规格驱动开发（SDD） | [Fission-AI/OpenSpec](https://github.com/Fission-AI/openspec) |
| mattpocock/skills（含 grill-me） | 小而可组合的工程实践 skills 集合 | [mattpocock/skills](https://github.com/mattpocock/skills) |

## 一、Buildomator（原 gsd-plugin）

> [!note] 仓库已改名
> `jnuyens/gsd-plugin` 现重定向到 `buildomator/buildomator`，命令前缀由 `/gsd:` 改为 `/bm:`。旧前缀暂时仍可解析，迁移只需换前缀。

- **定位**：为 Claude Code 提供 plan → execute → verify 的工作流，带 MCP 状态存储、`/compact` 后自动续跑、约 92% 的每轮 token 开销削减。
- **安装**（在 Claude Code 会话内执行，不是宿主 shell）：
  ```
  /plugin marketplace add buildomator/marketplace
  /plugin install bm@buildomator
  /reload-plugins
  ```
- **常用命令**：
  - `/bm:new-project` — 新建项目
  - `/bm:plan-phase` — 规划阶段
  - `/bm:execute-phase` — 执行阶段
  - `/bm:verify-work` — 验证
  - `/bm:version` — 查看版本 / 检查更新

## 二、OpenSpec

- **定位**：轻量的 spec 层，写代码前先对齐需求；每个变更一个目录（proposal / specs / design / tasks），支持 30+ AI 工具。
- **安装**：
  ```powershell
  # Windows
  irm https://openspec.dev/cli/install.ps1 | iex

  # 或用 npm（需 Node.js 22.12+）
  npm install -g @fission-ai/openspec@latest
  ```
  然后在项目目录初始化：
  ```bash
  cd your-project
  openspec init
  ```
- **常用命令**：
  - `/opsx:explore` — 探索式讨论，先想清楚再动手
  - `/opsx:propose <需求>` — 生成 proposal / specs / design / tasks
  - `/opsx:apply` — 按 tasks 实施
  - `/opsx:archive` — 归档变更并更新 specs
  - `openspec update` — 升级后刷新项目内的 AI 指令

## 三、mattpocock/skills（含 grill-me）

- **定位**：作者日常使用的工程 skills 集合，强调小、可改、可组合；不接管流程，保持人的控制权。
- **安装**：
  ```
  claude plugin install mattpocock-skills@claude-plugins-official
  ```
  首次使用在每个仓库跑一次 `/setup-matt-pocock-skills`（配置 issue tracker、triage 标签、文档位置）。
- **常用命令**：
  - `/ask-matt` — 路由入口，告诉你当前场景该用哪个 skill / 流程
  - `/grill-me` — 针对方案或设计反复追问，直到设计树的每个分支都有答案
  - `/grill-with-docs` — 同上，同时沉淀领域术语与 ADR
  - `/tdd`、`/diagnosing-bugs`、`/code-review`、`/improve-codebase-architecture` 等

> [!tip] 三者关系
> OpenSpec 管“写代码前对齐需求”，Buildomator 管“分阶段推进并验证”，mattpocock/skills 提供可插拔的具体实践（追问、TDD、调试、评审）。按项目习惯挑一套主流程即可，避免流程互相打架。
