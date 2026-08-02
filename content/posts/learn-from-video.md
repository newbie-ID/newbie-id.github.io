---
title: "【Skill】把视频变成学习材料"
date: 2026-07-28T20:30:00+08:00
draft: false
tags:
  - Claude Code
  - Skill
  - AI
  - whisper.cpp
  - 视频学习
categories:
  - 工具
summary: "learn-from-video-skill 是一个 Agent Skill：丢给它一个视频链接或本地音视频文件，自动转录成带时间戳的文稿，再用 AI 把它变成可学、可记、可测的学习成品。"
---

> 开源地址：https://github.com/newbie-ID/learn-from-video-skill （MIT License）
>
> 完整工作流图（动画演示）：[learn-from-video-skill-preview.html](/posts/learn-from-video/learn-from-video-skill-preview.html)

## 一句话介绍

**learn-from-video-skill** 是一个 Agent Skill：丢给它一个视频链接（B站 / YouTube）或本地音视频文件（面试录音、会议录音、录屏），它会自动转录成**带时间戳的文稿**，然后基于文稿帮你做三件事——

- **学**：追问学习，AI 带你看视频，回答引用时间戳，随时跳回原片
- **记**：学霸笔记 / PPT 复习卡 / 概念图 / 一页纸摘要
- **测**：自测题 / 闪卡（兼容 Anki）/ 时间戳大纲

## 为什么做这个

不知道你有没有这样的经历：

- 收藏了一个 80 分钟的技术课程视频，看了 20 分钟就搁置了，因为没有笔记、没有重点，再捡起来等于重看；
- 面试录了音想复盘，但拖动着进度条找"那个问题我是怎么答的"找到崩溃；
- 看完视频觉得"会了"，两周后什么都不记得——没有自测，根本不知道自己没学会。

视频是信息密度很低的学习载体：**不能搜索、不能跳读、不能复习、不能检验**。

而这个问题的解法其实很清楚——**把视频先变成结构化文本，剩下的事交给 AI**。

这就是 learn-from-video-skill 的定位：**上游管道 skill**。核心能力只有一件事——`视频/音频 → 结构化文本 + 时间索引`，呈现层全部外包给成熟的下游 skill，不造轮子。

## 快速开始

### 安装

对你的 Agent（Claude Code 等）说一句话：

```
帮我安装 https://github.com/newbie-ID/learn-from-video-skill 这个 skill
如遇网络问题，尝试使用镜像站 https://gh-proxy.com/ 进行安装
```

首次使用会自动下载并配置 **whisper.cpp / yt-dlp / FFmpeg** 三件套 + 三个下游 skill（scholar-notes / ppt-animation / flowchart），按你的硬件自动选择 Whisper 模型档位（large-v3 / medium / small）。完成后重启 Agent 让下游 skill 被发现——**整个初始化是一次性的，后续使用不再需要加载**。

### 用法

还是一句话：

```
用 learn-from-video-skill 处理这个视频 https://www.bilibili.com/video/BVxxxxxxxx
帮我把这段面试录音做成复盘笔记：D:/interview.m4a
这个面试录屏讲了什么，出几道自测题：D:/screen-record.mp4
```

同一个视频可以连续触发多个输出（先做笔记、再出自测题），共享同一份缓存文稿，**不重复下载、不重复转录**。

## 完整工作流

### 阶段 0：双模式路由

每次激活，skill 的第一步永远是运行 `setup.sh --check` 读环境快照（`env.local.json`）：

- 返回 `NOT_INITIALIZED` → 走**模式一：首次初始化**（一次性）
- 返回 `READY` → 走**模式二：日常使用**

> `env.local.json` 只由 setup 生成，绝不手写——它记录的是本机绝对路径。

### 阶段 1：首次初始化（一次性）

`setup.sh` 自动完成五步：

1. **探测环境** — OS / GPU / 显存 / 内存 / 当前 Agent 的标准 skills 目录
2. **装三件套** — FFmpeg / whisper.cpp / yt-dlp（已存在则跳过；预编译二进制优先，包管理器兜底，全程镜像加速）
3. **选模型** — 按硬件自动选 `large-v3` / `medium` / `small`（中文优先），下载 ggml 模型
4. **装下游 skill** — 检测 scholar-notes / ppt-animation / flowchart，缺则按依赖清单拉取（git clone 优先，失败 fallback zip + 镜像站）
5. **写快照** — 生成 `env.local.json`，之后每次激活秒级自检

### 阶段 2：日常使用 Pipeline

`process.sh` 是日常入口，流程是确定性的机械步骤：

```
双入口判断 → 缓存命中检查 → 下载 → 提取音频 → 字幕优先判断 → 生成文稿 → 入库
```

几个关键设计：

- **双入口**：`http(s)://` 开头走 URL 模式（yt-dlp 支持的平台都行）；本地路径走本地模式——**音频文件直接转录，视频文件先由 FFmpeg 提取音频**。面试录音、录屏、会议录音都是一等公民。
- **缓存命中检查**：以视频 ID（本地文件用内容 hash）为键，命中 `library/<id>/transcript.*` 就秒出，跳过后续所有步骤。
- **字幕优先**：下载时同时尝试拉官方字幕——有字幕直接采用（**零成本、100% 准确**，跳过 Whisper）；没有才 fallback 到 whisper.cpp 转录。
- **产出**：`transcript.{srt,txt,json}` 三种格式，全部带时间戳。

### 阶段 3：学 · 记 · 测 闭环

文稿入库后，skill 会展示输出选项菜单，可多选叠加：

| 类别 | 输出 | 谁来做 |
|------|------|--------|
| **学** | 追问学习（AI 带你看视频，带时间戳定位） | 本 skill 自带 |
| **记** | 学霸笔记 / PPT 复习卡 / 概念图 / 一页纸摘要 | 下游 skill（前 3）/ 自带（摘要） |
| **测** | 自测题 / 闪卡 / 时间戳大纲 | 本 skill 自带 |

推荐组合：

- **深度学习**：追问学习 → 学霸笔记 → 自测题
- **快速消化**：一页纸摘要 → 时间戳大纲
- **做分享成品**：PPT 复习卡 / 概念图

## 实测性能

> 测试设备：RTX 4070 SUPER（12GB 显存），whisper.cpp large-v3，cublas-12.4 GPU build。

| 视频 | 时长 | CPU（估算） | GPU（实测） | 加速比 |
|---|---|---|---|---|
| 某面试录屏 | ~11.5 min | ~30–40 min | **122 s** | ~**15–20×** |
| CS336 Lecture 1（课程视频） | ~80 min | ~150–240 min | **522 s**（~8.7 min） | ~**17–28×** |

整个 Skill 的处理流程时间主要开销在 whisper 转录。有 N 卡，在 CUDA 的加持下，一部 80 分钟的课程，不到 9 分钟就变成全文稿；而如果官方字幕命中，转录成本直接为零。若是 CPU 模式，速度会慢很多， 可调用音频转文字 API 实现快速转录。

## 实测成品 Demo

三个真实视频，每个产出三种成品（**学霸笔记** · **PPT 演示** · **概念图**），全部由 learn-from-video-skill 一键生成，点击即可在线查看。

| 视频源 | 语言 | 学霸笔记 | PPT 演示 | 概念图 |
|---|:---:|---|---|---|
| **BERT 论文精读**<br>BV1PL411M7eQ · ~38 min | 中文 | [📖 笔记](/posts/learn-from-video/demos/bert/note.html) | [🎞️ 演示](/posts/learn-from-video/demos/bert/slides.html) | [🕸️ 概念图](/posts/learn-from-video/demos/bert/diagram.html) |
| **LLM 架构与超参数综述**<br>Stanford · lVynu4bo1rY · ~89 min | 英文 | [📖 笔记](/posts/learn-from-video/demos/llm-architecture/note.html) | [🎞️ 演示](/posts/learn-from-video/demos/llm-architecture/slides.html) | [🕸️ 概念图](/posts/learn-from-video/demos/llm-architecture/diagram.html) |
| **CS224N L1 · Word2Vec**<br>Stanford · BV1Nh8BzrEED_p1 | 英文 | [📖 笔记](/posts/learn-from-video/demos/word2vec/note.html) | [🎞️ 演示](/posts/learn-from-video/demos/word2vec/slides.html) | [🕸️ 概念图](/posts/learn-from-video/demos/word2vec/diagram.html) |

> 中英文视频都能处理：**官方字幕命中时直接采用**（零转录成本），笔记 / 演示 / 概念图会跟随原片语言输出；无字幕时由 whisper.cpp 自动检测语言转录。

## 技术栈与致谢

- **whisper.cpp** — 本地转录，单二进制、跨平台，CPU / cublas GPU 自动选择
- **yt-dlp** — 视频下载
- **FFmpeg** — 音频提取
- **[AI_Animation](https://github.com/Unclecheng-li/AI_Animation)**（MIT, @Unclecheng-li）— scholar-notes / ppt-animation / flowchart 三个下游渲染 skill，感谢作者[网络小白_Uncle城](https://space.bilibili.com/37858284)。

项目地址：https://github.com/newbie-ID/learn-from-video-skill ，MIT License，仅供学习用途。欢迎试用和提 Issue。
