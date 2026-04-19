---
title: "给 AI Agent 的大脑装上 git 分支管理"
date: 2026-04-11T20:45:01+08:00
draft: false
tags: ["AI"]
categories: ["AI"]
summary: "本文介绍agent-memory工具，通过将AI记忆与Git分支一一绑定，实现多任务间上下文自动隔离、错误路径不污染、记忆按需沉淀，大幅提升AI编程的上下文纯净度与协作效率。"
ShowToc: true
TocOpen: false
---

## 一句话概述

将 AI 记忆与 git 分支一一绑定，切换 git 分支时记忆自动跟随切换。每个任务拥有独立的上下文空间，排查问题时的错误路径不会污染开发分支，解决多任务并行时 AI 记忆相互干扰与上下文膨胀的问题。

## 业务背景和痛点

### 背景

Claude Code 内置了 auto memory 机制——通过项目级的 `MEMORY.md` 文件在对话间持久化关键信息（架构决策、文件路径、踩坑记录等）。每次新对话开始时，Claude 会读取 MEMORY.md 作为上下文，从而"记住"之前的工作。

但在实际研发中，我们很少只做一件事。以我自己的日常开发为例，我经常需要：

- 功能开发：在分支 A 上和 Claude 深度讨论了 Fluss 对 Paimon Deletion Vector 的支持方案，积累了大量上下文。需求优先级调整后切到分支 B 做其他功能，几周后又需要回来继续 Paimon Deletion Vector 方案，之前的上下文已经被其他任务的记忆淹没。
- 紧急排查：线上报了 OOM，需要临时切到 `debug-oom` 分支排查
- Code Review：Review 同事的 PR，切到 `review-xxx` 分支

这三件事分属不同的 git 分支，但 Claude Code 的 MEMORY.md 只有一份。

### 痛点

![](images/img_01.png)

痛点 1：记忆污染

排查 OOM 时 Claude 记录了"怀疑是缓存泄漏 → 不是"、"可能是连接池 → 也不是"等试错路径。切回功能开发分支后，这些无关信息仍然留在 MEMORY.md 中，占用 token 配额，干扰 Claude 的注意力。

痛点 2：错误路径残留

排查过程中的错误假设被永久写入 MEMORY.md。下次 Claude 读到"怀疑是缓存泄漏"时可能误以为这是一个有效结论，导致后续判断偏差。

痛点 3：上下文膨胀不可控

一个月后 MEMORY.md 积累了十几个任务的记忆，相互交叉。想清理但不敢删——不知道哪条还有用，也不知道它是什么时候、因为什么原因加进来的。

痛点 4：切换成本高

每次切换任务都需要手动告诉 Claude"忘掉之前的上下文"或手动编辑 MEMORY.md，效率低且容易遗漏。

## 使用方式

### 安装（一条命令）

```
# 全局安装 CLI
uv tool install agent-memory

# 在目标项目中安装 skills + hooks + CLAUDE.md
cd /path/to/your-project
agent-memory install
```

一条命令自动配置好 Skills、Hooks、CLAUDE.md 和数据目录，开箱即用。

### 核心机制：记忆分支 = git 分支

安装后无需手动操作，一切自动运行：

1. 你切 git 分支（`git checkout support-paimon-dv`）
2. 下次发消息给 Claude Code 时，UserPromptSubmit Hook 检测到 git 分支变化
3. Hook 自动执行：保存当前记忆 → 创建/切换到对应的记忆分支 → 加载新分支的 MEMORY.md
4. 对话结束时，Stop Hook 自动保存当前 MEMORY.md 到分支

![](images/img_02.png)

```
git checkout support-hudi
  → MEMORY.md 自动加载支持 hudi 的实现计划

git checkout debug-oom
  → MEMORY.md 自动切换为 OOM 排查的上下文

git checkout support-Blob-Type
  → MEMORY.md 恢复为 Blob Type 的内容，OOM 排查的记忆完全隔离
```

### 记忆写入保障

通过 CLAUDE.md 中的强规则指令，要求 Claude **每轮回复前先按需更新 MEMORY.md**，而非等到对话结束。这确保了即使对话意外中断，之前轮次的记忆也已持久化。

### Slash Commands

| 命令 | 功能 |
| --- | --- |
| `/memory-status` | 查看当前记忆分支状态 |
| `/memory-save` | 总结当前对话并保存 |
| `/memory-history` | 查看记忆变更历史 |
| `/memory-create <name>` | 手动创建记忆分支 |
| `/memory-switch <name>` | 手动切换记忆分支 |
| `/memory-debug <topic>` | 创建临时排查分支 |
| `/memory-done <结论>` | 结束排查，发布结论到 main 并清理临时分支 |

### 发布与拉取：跨任务的知识沉淀

![](images/img_03.png)

每个记忆分支是隔离的，但有价值的经验不应该被锁在某个分支里。agent-memory 借鉴了 git 的 publish/pull 模型，用 main 分支作为团队知识库：

- publish：将当前任务中提炼出的结论发布到 main 分支。只发布结论，不发布过程中的试错和废弃方案。
- pull：在任意分支上从 main 拉取最新的共享知识，让新任务也能受益于之前的经验。

典型场景：排查完一个 OOM 问题后，过程中的"怀疑是缓存泄漏""可能是连接池"等错误假设留在临时分支里，只把最终结论发布到 main：

```
agent-memory publish "OOM root cause: BatchReader batch size 无上限，修复为 LIMIT 1000 (PR #234)"
```

之后在任何分支上执行 `agent-memory pull`，都能拉取到这条经验。这样 main 分支逐渐积累成一份干净的项目知识沉淀，没有噪音，只有结论。

## 关键交付物

### 1. agent-memory CLI 工具

- 通过 `uv tool install` 或 `pip install` 安装
- 提供 18 个子命令覆盖完整工作流

### 2. Claude Code 集成

| 组件 | 数量 | 说明 |
| --- | --- | --- |
| Skills | 7 个 | 斜杠命令，覆盖状态查看、保存、切换、排查等场景 |
| Hooks | 2 个 | UserPromptSubmit（自动切换）+ Stop（自动保存） |
| CLAUDE.md 模板 | 1 份 | 记忆写入规则，确保 Claude 主动记录 |
| 一键安装器 | 1 个 | agent-memory install 自动配置目标项目 |

## 效果评估

### 定性改进

| 指标 | 改进前 | 改进后 |
| --- | --- | --- |
| 上下文纯净度 | 所有任务记忆混杂在同一个 MEMORY.md | 每个 git 分支独立记忆，互不干扰 |
| 错误路径处理 | 排查过程的试错永久残留 | 临时分支隔离，只 publish 结论，其余随分支删除 |
| 切换成本 | 手动编辑 MEMORY.md 或口头告知 Claude | git checkout 自动触发记忆切换，零手动操作 |
| 记忆可追溯性 | 不知道何时添加、为何添加 | history diff 精确定位每次变更 |
| 清理安全性 | 直接编辑 MEMORY.md，怕丢信息 | 有完整快照历史，支持 restore 回滚 |

### 定量估算

以日常开发场景（同时进行 2-3 个任务，每天切换 3-5 次）为基准估算：

| 指标 | 估算值 | 说明 |
| --- | --- | --- |
| MEMORY.md 有效信息占比 | 从 ~40% 提升到 ~95% | 排除无关任务记忆和错误路径后的有效占比 |
| 记忆清理频率 | 从每月 1-2 次降到基本不需要 | 临时分支用完即删，main 只保留结论 |
| 上下文 token 节省 | ~30-50% | 隔离后每个分支的 MEMORY.md 只包含本任务相关内容 |

### 实际使用示例

![](images/img_04.png)

在 Apache Fluss 项目中，使用 agent-memory 后的典型工作流：

```
09:00  git checkout support-paimon-dv
       → MEMORY.md 自动加载: "需改 8 个文件..."
       → Claude 无缝继续昨天的支持 paimon dv 实现

11:00  收到线上告警，git checkout debug-oom
       → MEMORY.md 自动切换为空（新分支），Claude 专注排查
       → 排查完成: agent-memory publish "OOM root cause: unbounded batch size"

11:30  git checkout support-paimon-dv
       → MEMORY.md 恢复 paimon dv 上下文，OOM 排查内容完全不可见
       → Claude 继续 paimon dv 实现，零切换成本

14:00  git checkout review-236
       → MEMORY.md 自动切换，Claude 进入 PR Review 上下文
       → Review 结论自动逐轮写入 MEMORY.md，author 修复 comment 之后可继续切回来 review
```

整个过程中没有一次手动编辑 MEMORY.md，记忆隔离完全自动化。
