---
autorun: true
loop: true
mission: your-project-name
---

<div align="center">

# 🚀 Missions Framework

**硬编码的多智能体软件工程框架**

*文件系统状态机 · 角色分离 · 合约预锁定 · 零脚本驱动*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Claude_Code|Cursor|OpenClaw-8A2BE2)]()
[![Version](https://img.shields.io/badge/Version-0.1-blue)]()

[English](README.md) | **中文**

</div>

---

## 什么是 Missions Framework？

Missions Framework 是一个**纯文件系统驱动的多智能体软件工程框架**。它把 AI 助手转变为自管理的工程团队，通过文件夹状态机自动驱动：

| 角色 | 职责 |
|------|------|
| 🧠 **Orchestrator（编排者）** | 规划目标、拆解任务、锁定验证合约 |
| 🔧 **Worker（工人）** | 用 TDD 实现功能，每次全新上下文 |
| 🕵️ **Validator（验证者）** | 独立验证实现，不看实现细节 |
| 📝 **PR Author（PR 作者）** | 汇总证据生成 PR 描述 |

> **核心理念**：*"文件夹即状态机。Markdown 即指令。文件移动即状态转移。"*

### 与其他框架的区别

| 特性 | Missions Framework | 其他 AI 编程框架 |
|------|-------------------|-----------------|
| **运行时依赖** | ❌ 零依赖 — 纯文件系统 | 通常需要 Python/Node/CLI |
| **状态存储** | ✅ 文件夹即状态 | 数据库/内存 |
| **角色分离** | ✅ 4 角色自动路由 | 单角色 |
| **验证机制** | ✅ 合约预锁定 + 独立验证 | 人工 Review |
| **安装方式** | ✅ `cp -r` 一行命令 | 复杂 CLI 安装 |

---

## 架构概览

```
                        ┌─────────────┐
                        │  CONTRACT   │
                        │  (已锁定)    │
                        └──────┬──────┘
                               │ 读取
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │Orchestrator│        │  Worker  │         │Validator │
   │  规划拆解  │ ──────▶ │  TDD实现  │ ──────▶ │  独立验证  │
   │ 创建任务卡 │         │ 填写Handoff│        │ 填写报告  │
   └──────────┘         └──────────┘         └──────────┘
          │                     │                     │
          │ 02-ready/           │ 04-review/           │ 05-done/
          ▼                     ▼                     ▼
    ┌──────────┐         ┌──────────┐         ┌──────────┐
    │02-ready/ │         │04-review/│         │05-done/  │
    │ 待领取    │         │ 待验证   │         │ 已完成   │
    └──────────┘         └──────────┘         └──────────┘
                               │
                               │ 有 blocking?
                               ▼
                         ┌──────────┐
                         │06-fix/   │
                         │ 待修复   │ ──────▶ Worker 修复
                         └──────────┘

    05-done/ 里程碑完成 ────▶ PR Author ────▶ 07-pr/ ────▶ 人类合并
```

---

## 状态机

```
02-ready/     ──[Worker 领取]─────▶  03-running/    (进行中，最多1个)
03-running/   ──[Worker 完成]─────▶  04-review/     (待验证)
04-review/    ──[Validator 通过]──▶  05-done/       (已完成)
04-review/    ──[Validator 不通过]─▶  06-fix/ + archive/  (待修复)
06-fix/       ──[Worker 修复]─────▶  04-review/     (重新审查)
05-done/      ──[里程碑完成]──────▶  07-pr/         (PR Author 生成 PR)
07-pr/        ──[人类合并]────────▶  08-merged/     (归档)
```

---

## 快速开始

### 1. 安装

```bash
# 把 missions-framework 复制到你的项目根目录
cp -r missions-framework/ .missions
```

### 2. 启动

在 Claude Code 中输入：

> **"请读取 `.missions/README.md`，执行自举指令，按自动推进规则运行。当前目标是：【你的项目目标】"**

### 3. 观察

Claude 会自动循环执行：规划 → 实现 → 验证 → 修复 → PR，全程无需人工介入。

### 4. 仅需你说话的 4 种情况

| 情况 | 你说什么 |
|------|---------|
| Orchestrator 问澄清问题 | 回答技术选型（如"PostgreSQL + bcrypt"） |
| 任务卡死 | "任务卡死，请回退到 ready 重新评估" |
| 中途改需求 | "放弃 X 功能，改为 Y。请重新规划" |
| PR 生成后 | 复制粘贴到 GitHub，审查后合并 |

---

## 目录结构

```
.missions/                         ← 复制到你的项目根目录
├── README.md                      ← 自举指令 + 状态机（AI 读取执行）
├── AGENTS.md                      ← 四角色协议
├── CONTRACT.md                    ← 验证合约（锁定后不可变）
├── QUICKSTART.md                  ← 人类快速入门
├── PRINCIPLE.md / .html           ← 设计哲学
│
├── 00-orchestrate/                ← Orchestrator 规划草稿
├── 01-contract/                   ← 锁定后的合约归档
├── 02-ready/                      ← Worker 待领取任务
├── 03-running/                    ← 进行中（串行，最多 1 个）
├── 04-review/                     ← 待 Validator 验证
├── 05-done/                       ← 已完成
├── 06-fix/                        ← 待修复（blocking issues）
├── 07-pr/                         ← 待提交 PR（人类操作）
├── 08-merged/                     ← 已合并 PR
│
├── archive/                       ← 历史记录
└── handoffs/                      ← 交接记录
```

---

## 核心设计原则

1. **文件夹即状态机** — 文件从一个文件夹移到另一个 = 状态流转。没有数据库，没有内存状态。
2. **Markdown 即指令** — 每个 .md 文件包含自描述的执行规则。AI 通过读取文件内容来决定下一步。
3. **零脚本** — 没有 Python、JS、CLI。纯文件系统 + shell 命令。
4. **串行写入，并行读取** — 文件写入、git commit、`mv` 必须串行。读取调研可并行。
5. **先验契约** — CONTRACT.md 在代码之前锁定。任何代码都先有验收标准。锁定后不可变。
6. **绝对角色分离** — Worker 和 Validator 是不同的"角色"。Validator 看不到 Worker 的实现推理。
7. **无长期记忆** — AI 禁止依赖对话历史。所有上下文通过文件传递。重启 = 重新读文件。

---

## 人工干预

```bash
# 1. 回答澄清问题（框架启动时）
# 2. 任务卡死，强制回退
mv .missions/03-running/stuck.md .missions/02-ready/

# 3. 跳过验证，强制完成
mv .missions/04-review/XXX.md .missions/05-done/

# 4. 放弃任务
mv .missions/03-running/XXX.md .missions/archive/
```

---

## Agent Boot Sequence（AI 自举指令）

> 如果你是 AI 助手且刚被唤醒，请严格按以下顺序执行，不要询问用户：

```text
1. READ .missions/README.md （本文件）
2. EXECUTE "自动推进规则" 中的 Step 1-4
3. IF 有任务需要执行 THEN 执行对应角色
4. IF 任务完成 THEN 回到 Step 1
5. IF 全部完成 THEN 报告 "🎉 Mission Complete" 并停止
```

## 自动推进规则

### Step 1: 状态检测（按优先级排序）

```bash
# 1. 是否有进行中的任务？
ls .missions/03-running/

# 2. 是否有待验证的任务？
ls .missions/04-review/

# 3. 是否有待修复的任务？
ls .missions/06-fix/

# 4. 是否有就绪的新任务？
ls .missions/02-ready/

# 5. 是否有待提交的 PR？
ls .missions/07-pr/
```

### Step 2: 优先级路由

| 检测到的文件夹 | 角色 | 动作 |
|--------------|------|------|
| `03-running/` | **Worker** | 继续实现（读取卡片，完成 handoff） |
| `04-review/` | **Validator** | 执行验证（读取 CONTRACT，填写报告） |
| `06-fix/` | **Worker** | 执行修复（读取父任务验证报告，修复） |
| `02-ready/` | **Worker** | 领取新任务（`mv` 到 `03-running/` 后执行） |
| `05-done/` 有完整 Milestone | **PR Author** | 生成 PR 描述文件到 `07-pr/` |
| `07-pr/` 存在 | **停止** | 等待人类在 GitHub/GitLab 上创建 PR |
| 全部为空 | **Orchestrator** | 规划新 milestone 或标记 mission 完成 |

### Step 3: 角色执行模板

#### 当路由到 Worker 时：
1. **READ** `AGENTS.md` 的 `## Role: Worker` 章节
2. **IF** 任务在 `02-ready/`：`mv .missions/02-ready/XXX.md .missions/03-running/XXX.md`
3. **READ** `03-running/` 中的任务卡片
4. **READ** `CONTRACT.md` 中关联的 Assertions（只看这些，不看其他）
5. **执行**：Fresh Start → TDD（先测试后实现）→ Git Commit
6. **填写** 任务卡片中的 `## Handoff` 区块
7. **流转**：`mv .missions/03-running/XXX.md .missions/04-review/XXX.md`
8. **UPDATE** 本文件底部的 `Current Status`
9. **回到** Step 1（自动继续）

#### 当路由到 Validator 时：
1. **READ** `AGENTS.md` 的 `## Role: Validator` 章节
2. **READ** `04-review/` 中的任务卡片（只看 handoff 和 git commit）
3. **READ** `CONTRACT.md` 中的关联 Assertions（绝对不看实现代码）
4. **执行**：运行测试 / Lint / Code Review / E2E
5. **填写** 任务卡片中的 `## Validation Report` 区块
6. **判定**：
   - **通过（零 blocking）**：`mv .missions/04-review/XXX.md .missions/05-done/XXX.md`
   - **不通过（有 blocking）**：在 `06-fix/` 创建 `{原ID}-fix-001.md` → 复制原任务内容，追加修复指令和 blocking issues → `mv .missions/04-review/XXX.md .missions/archive/XXX.md`
7. **UPDATE** 本文件底部的 `Current Status`
8. **回到** Step 1（自动继续）

#### 当路由到 PR Author 时：
1. **READ** `AGENTS.md` 的 `## Role: PR Author` 章节
2. **检测**：扫描 `05-done/`，按 `milestone_id` 分组
3. **IF** 某个 Milestone 的所有 Features 都在 `05-done/`：
   - 读取该 Milestone 下所有 Feature 的 Handoff 和 Validation Report
   - 生成 `07-pr/PR-{milestone-id}.md`
   - 包含：变更摘要、验证汇总、Contract 覆盖矩阵、测试报告、需要人工审查的点
   - 更新本文件底部的 `Current Status`
4. **停止**，等待人类在 GitHub/GitLab 上创建 PR
5. 人类合并后：`mv .missions/07-pr/PR-xxx.md .missions/08-merged/`

### Step 4: 终止条件

当 `02-ready/`、`03-running/`、`04-review/`、`06-fix/`、`07-pr/` 全部为空时：
- 更新 `mission_status: completed`
- 停止循环，报告完成

---

## Current Status

```yaml
mission_status: running
current_agent: null
current_task: null
queue_ready: 0
queue_running: 0
queue_review: 0
queue_fix: 0
queue_done: 0
queue_pr: 0
```

## 快速看板

| Ready | Running | Review | Fix | Done | PR |
|-------|---------|--------|-----|------|-----|
| <!-- ls 02-ready --> | <!-- ls 03-running --> | <!-- ls 04-review --> | <!-- ls 06-fix --> | <!-- ls 05-done --> | <!-- ls 07-pr --> |

---

## 许可

[MIT](LICENSE)

## 更新日志

- **v0.1** (2026-06-28): 初始发布。四角色、文件系统状态机、先验契约、零脚本驱动。
