---
title: "【工具】skill-port：跨机器/跨 harness 搬运 Agent Skill"
date: 2026-08-13T17:50:00+08:00
draft: false
tags:
  - Claude Code
  - Skill
  - Agent
  - AI
  - 工具
categories:
  - 工具
summary: "skill-port 是一个给 AI Agent 用的 Skill 打包/还原工具：把任意 harness（Claude Code / Goose / Codex / Cursor 等）里已装的 skill 打成可移植的 .skill/ 目录包，再在任意机器、任意 harness 还原成相同体验。"
---

> 开源地址：https://github.com/newbie-ID/skill-port （MIT License）
>
> 完整工作流图（手写笔记风格）：[skill-port-flow.html](/posts/skill-port/skill-port-flow.html)

## 一句话介绍

**skill-port** 是一个给 AI Agent 用的 **Skill 打包 / 还原工具**。它把任意 harness（Claude Code / Goose / Codex / Cursor / Gemini CLI / Pi 等）里已装的 skill，**抽出来打成自包含的 `.skill/` 目录包**，再在**任意机器、任意 harness** 还原成相同使用体验。

## 为什么做这个

2025–2026 年，各家 LLM harness 都在出 Agent 工具，skill 格式已经收敛成开放标准（`SKILL.md` + `.agents/skills/`）。但**跨机器搬运、跨 harness 迁移、备份已装 skill** 仍是手动活：

- 目录位置各家不一：`~/.claude/skills/`、`~/.agents/skills/`、`.cursor/skills/`……
- 依赖（二进制 / 下游 skill）要重装
- 运行态缓存（下载的工具链、模型）不能搬，一搬就是几个 G

skill-port 解决的正是这个：**不做格式翻译（已经不需要了），专注打包 / 跨机还原 / 依赖重装**。

## 两阶段：pack 和 restore

核心是两个命令：

```bash
# 打包：把已装 skill 打成 .skill/ 目录包
python scripts/pack.py learn-from-video-skill --out ~/bundles

# 还原：把包还原到目标 harness
python scripts/restore.py ~/bundles/learn-from-video-skill.skill --target-harness goose
```

- **pack**：扫描已装 skill → 解析 `SKILL.md` → 复制 payload（排除运行态）→ 识别依赖与工具面假设 → 产出 `<name>.skill/`（= `manifest.json` + `payload/`）。
- **restore**：读包 → **工具面硬拦**（缺必需能力即中止、交人 review）→ 同名冲突询问 → 按目录映射复制 payload → 核查并提示重装依赖。

完整的两阶段流程见文章开头的 [skill-port-flow.html](/posts/skill-port/skill-port-flow.html)。

## 三个关键设计

1. **两张表就是全部适配**。`HARNESSES`（每个 harness 的 skill 发现目录）+ `CAPABILITIES`（每个 harness 的工具面能力）。加一个新 harness = 往这两张表加一行，核心代码不动。所谓的 wrapper「翻译」退化成一张目录位置映射表。

2. **依赖只声明、不内联**。二进制 / 下游 skill / MCP 只记进 `manifest.json`，目标机由 `setup.sh` 重装；运行态产物（`bin/` `models/` `library/`）整个排除——这也是为什么一个源 skill 5.5 GB，打包后只有 **71.8 KB**。

3. **硬拦，而不是静默装上**。工具面不匹配（比如 skill 要 sub-agents 但目标 harness 没有）会立即中止并提示交人 review；同名冲突会询问；默认不会自动跑 `setup.sh`（可能重、可能交互）。宁可停下来让人决定。

## 诚实说明

依赖和能力的自动识别是**启发式**，不一定全，pack 后建议人工 review 一下 `manifest.json`；工具面能力表是保守估计，随各 harness 演进需要维护。这是一个小而专的工具，不追求完美识别，只把「搬运 skill」这件重复的体力活自动化掉。

## 相关

- 项目仓库：https://github.com/newbie-ID/skill-port
- 工作流图：[skill-port-flow.html](/posts/skill-port/skill-port-flow.html)
