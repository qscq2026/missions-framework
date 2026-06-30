# Missions 多 Agent 框架原理说明书

> 版本: v0.1 | 日期: 2026-06-28 | 作者: Claude Code

---

## 目录

1. [设计哲学](#1-设计哲学)
2. [核心问题](#2-核心问题)
3. [系统架构](#3-系统架构)
4. [状态机设计](#4-状态机设计)
5. [文件结构详解](#5-文件结构详解)
6. [角色协议](#6-角色协议)
7. [工作流详解](#7-工作流详解)
8. [与现有方案对比](#8-与现有方案对比)
9. [局限性与未来](#9-局限性与未来)

---

## 1. 设计哲学

### 1.1 核心命题

> **"瓶颈不是智能，而是人类注意力。"**

前沿模型已能并行处理 50 个任务，但人类工程师同时只能关注 3-4 个线程。Missions 的目标不是让单个 Agent 更聪明，而是让**多个 Agent 像工程团队一样协作**，把人类从"写代码"变成"管理工程团队"。

### 1.2 关键洞察

| 洞察 | 来源 | 解决方案 |
|------|------|---------|
| 自我评估偏见 | Factory 实践 | 实现者/评估者绝对分离 |
| 上下文稀释 | Factory 实践 | 短轨迹 + 结构化交接 |
| 状态碎片化 | 多 Agent 通信 | 中心化编排 + 文件系统状态机 |
| 契约漂移 | 传统 TDD | 先验锁定验证契约 |

### 1.3 设计原则

1. **文件夹即状态机**：文件移动 = 状态流转，无需数据库
2. **Markdown 即指令**：每个文件包含自描述的执行规则
3. **零脚本依赖**：纯文件系统驱动，任何环境可用
4. **三角色分离**：规划、实现、验证由不同"人格"执行
5. **先验契约**：代码之前锁定验收标准，不可事后修改

---

## 2. 核心问题

### 2.1 单 Agent 长任务的失败模式

```
┌─────────────────────────────────────────────────────────────┐
│  单 Agent 长任务（如"实现一个 Slack 克隆"）                      │
├─────────────────────────────────────────────────────────────┤
│  Step 1: 写代码 100 行 ✓                                      │
│  Step 2: 写代码 200 行 ✓                                      │
│  Step 3: 发现 Step 1 有 bug，但上下文已稀释，信号比 38%          │
│  Step 4: 试图修复，但自我评估偏见导致"合理化"而非"找问题"        │
│  Step 5: 5000 行后，代码能跑但架构腐烂，测试是"对实现的追认"    │
│  Step 6: 人类审查时发现根本性问题，需重写                       │
└─────────────────────────────────────────────────────────────┘
```

**根本原因**：
- **Context Dilution**：任务边界变宽，每一步有用上下文占比下降
- **Self-Evaluation Bias**：实现者会从自己之前的推理中寻求连贯性，无法客观评估
- **Append-only Trajectory**：Agent 的推理轨迹是只增不减的，错误会累积

### 2.2 传统多 Agent 框架的问题

| 问题 | 表现 | Missions 的解决 |
|------|------|----------------|
| 术语膨胀 | "delegation"、"orchestration"、"workflow" 混用 | 归约为 5 种通信模式，只用 4 种 |
| 直接通信 | Agent 点对点通信，状态在多个会话漂移 | 禁止 direct communication，中心化编排 |
| 缺乏契约 | 测试是"实现后补的" | 先验锁定 CONTRACT.md |
| 角色模糊 | "Lead Agent" 同时是规划者和执行者 | 严格三角色分离 |

---

## 3. 系统架构

### 3.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         人类项目经理                                 │
│                    （只在 milestone 边界介入）                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        .missions/README.md                            │
│                    （编排引擎：自举指令 + 自动路由）                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
    ┌─────────────────┐ ┌──────────────┐ ┌─────────────────┐
    │   Orchestrator   │ │    Worker     │ │    Validator    │
    │   （规划者）      │ │  （实现者）    │ │   （验证者）     │
    │                  │ │              │ │                 │
    │ • 拆解目标        │ │ • TDD 实现   │ │ • 独立验证      │
    │ • 锁定契约        │ │ • Fresh Start │ │ • 不看实现代码  │
    │ • 创建任务卡片    │ │ • 结构化交接  │ │ • 只看断言+结果 │
    └────────┬──────────┘ └──────┬───────┘ └────────┬────────┘
             │                   │                  │
             └───────────────────┼──────────────────┘
                                 ▼
                    ┌─────────────────────┐
                    │   文件系统状态机      │
                    │  （文件夹 = 状态）    │
                    └─────────────────────┘
```

### 3.2 数据流

```
用户目标
   │
   ▼
Orchestrator ──► CONTRACT.md（锁定）
   │                │
   ▼                ▼
02-ready/*.md ◄── 契约断言
   │
   ▼ mv
03-running/*.md ──► Worker 实现
   │                    │
   ▼ mv                 ▼
04-review/*.md ◄─── 结构化 Handoff
   │
   ▼
Validator ──► 通过 ──► 05-done/*.md
   │              │
   └─► 不通过 ──► 06-fix/*.md ──► 回到 Worker
   │
   ▼
PR Author ──► 07-pr/PR-M1.md
   │
   ▼
人类审查员 ──► GitHub PR ──► 08-merged/
```

---

## 4. 状态机设计

### 4.1 文件夹状态机

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 00-orchestrate│     │ 01-contract │     │ 02-ready    │
│  (规划草稿)   │     │  (契约锁定)  │     │  (待领取)   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                              │
                                              ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 05-done     │◄────│ 04-review   │◄────│ 03-running  │
│  (已完成)   │     │  (待验证)   │     │  (进行中)   │
└──────┬──────┘     └─────────────┘     └─────────────┘
       │
       ▼ 当 Milestone 全部完成
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  PR Author  │────►│  07-pr/     │────►│ 08-merged/  │
│  (生成 PR)  │     │  (待提交)   │     │  (已合并)   │
└─────────────┘     └─────────────┘     └─────────────┘
       │
       ▼ 不通过
┌─────────────┐
│ 06-fix/     │
│  (待修复)   │
└─────────────┘
```

### 4.2 状态流转规则

| 当前状态 | 触发条件 | 下一状态 | 操作 |
|---------|---------|---------|------|
| 02-ready | Worker 领取 | 03-running | `mv` 文件 |
| 03-running | Worker 完成 | 04-review | `mv` 文件 + 填写 Handoff |
| 04-review | Validator 通过 | 05-done | `mv` 文件 |
| 04-review | Validator 不通过 | 06-fix | 创建 fix 卡片 + 归档原任务 |
| 06-fix | Worker 修复完成 | 04-review | `mv` 文件 |
| 05-done | Milestone 全部完成 | 07-pr | PR Author 生成 PR 描述 |
| 07-pr | 人类合并 PR | 08-merged | `mv` 文件 |

### 4.3 串行约束

```
┌─────────────────────────────────────────────────────────────┐
│  串行写（Serial Write）                                      │
│  ─────────────────────                                       │
│  • 文件写入（代码修改）                                       │
│  • Git commit                                               │
│  • 状态移动（mv 文件）                                        │
│  • Handoff 填写                                             │
│                                                             │
│  并行读（Parallel Read）                                     │
│  ─────────────────────                                       │
│  • Codebase 检索                                            │
│  • API 文档调研                                             │
│  • 设计阶段研究                                             │
│  • 共享只读文件（AGENTS.md, CONTRACT.md）                    │
└─────────────────────────────────────────────────────────────┘
```

**为什么串行写？**

数学上：串行执行每步错误率 0.1%，100 步累计成功率 90%。并行让每步错误率涨到 1%，100 步累计成功率暴跌到 36%。**长周期任务的正确性是复利**。

---

## 5. 文件结构详解

### 5.1 核心文件

```
.missions/
├── README.md              # 编排引擎
│   ├── 自举指令（Boot Sequence）
│   ├── 自动推进规则（Step 1-4）
│   ├── 角色执行模板
│   ├── Current Status（YAML 状态看板）
│   └── 人工干预命令
│
├── AGENTS.md              # 三角色宪法
│   ├── Shared Rules（全局约束）
│   ├── Role: Orchestrator
│   ├── Role: Worker
│   ├── Role: Validator
│   └── Role: PR Author
│
└── CONTRACT.md            # 锁定契约
    ├── 全局约束
    ├── Assertions（行为断言）
    │   ├── ID
    │   ├── Behavior
    │   ├── Tool
    │   └── Evidence
    └── Coverage Map（Feature ↔ Assertion 映射）
```

### 5.2 状态文件夹

```
.missions/
├── 00-orchestrate/        # Orchestrator 规划草稿
│   └── 临时文件，规划完成后删除或归档
│
├── 01-contract/           # 契约锁定后归档
│   └── CONTRACT.md 的副本（不可变）
│
├── 02-ready/              # Worker 待领取
│   └── *.md（任务卡片，自包含执行指令）
│
├── 03-running/            # 进行中（串行，只允许一个文件）
│   └── *.md（当前执行中的任务）
│
├── 04-review/             # 待 Validator 验证
│   └── *.md（含 Handoff + Validation Report）
│
├── 05-done/               # 已完成
│   └── *.md（完整执行记录）
│
├── 06-fix/                # 待修复（blocking issues）
│   └── *-fix-*.md（修复指令，引用父任务验证报告）
│
├── 07-pr/                 # 待提交 PR
│   └── PR-*.md（人类可读的 PR 描述）
│
├── 08-merged/             # 已合并 PR
│   └── PR-*.md（归档）
│
└── archive/               # 历史归档
    └── 废弃/替换的任务卡片
```

### 5.3 任务卡片结构

```markdown
---
id: F-001
type: implementation | fix
agent: worker
milestone: M1
contract_assertions: [VAL-001, VAL-002]
parent: null | F-001
created: 2026-06-28T14:00:00
---

# {ID}: {标题}

## 上下文
- 前置依赖
- 需要读取的文件

## 执行要求
1. 具体步骤
2. TDD 要求
3. 覆盖率要求

## Handoff（Worker 填写）
- [ ] 检查项
- Git Commit 信息
- 发现的问题与妥协

## Validation Report（Validator 填写）
- 检查结果
- Issues（blocking / non-blocking / suggestion）
- Verdict

## 状态历史
- 时间线记录
```

---

## 6. 角色协议

### 6.1 Orchestrator（编排者）

```
输入: 用户目标
输出: CONTRACT.md + 02-ready/*.md

职责:
  ✓ 提出澄清问题（最多 3 个）
  ✓ 产出 Validation Contract（行为断言）
  ✓ 拆解为 Milestones → Features
  ✓ 创建任务卡片

禁止:
  ✗ 修改 src/ 代码
  ✗ 修改 CONTRACT（锁定后）
  ✗ 读 Worker trajectory
```

### 6.2 Worker（实现者）

```
输入: 任务卡片 + AGENTS.md + CONTRACT.md（仅关联 Assertions）
输出: Git Commit + Handoff

职责:
  ✓ Fresh Start（不依赖对话历史）
  ✓ TDD：先写测试，再写实现
  ✓ 最小侵入（只改必需文件）
  ✓ 结构化 Handoff

禁止:
  ✗ 修改 CONTRACT
  ✗ 做最终验收判断
  ✗ 与其他 Worker 通信
```

### 6.3 Validator（验证者）

```
输入: CONTRACT.md（仅 Assertions）+ Git 成品 + Handoff 命令输出
输出: Validation Report

职责:
  ✓ 运行测试 / Lint / Type-check
  ✓ Code Review（fresh context）
  ✓ E2E 测试（playwright / computer-use）

禁止:
  ✗ 看实现代码细节
  ✗ 修改源码
  ✗ 读 Worker 推理过程
```

### 6.4 PR Author（PR 起草者）

```
输入: 05-done/ 中同 Milestone 的所有 Feature
输出: 07-pr/PR-*.md

职责:
  ✓ 汇总变更（git diff --stat）
  ✓ 汇总测试覆盖率
  ✓ 汇总 Validation Report
  ✓ 生成 Contract Coverage 矩阵
  ✓ 标注 Review Notes（需要人类注意的点）

禁止:
  ✗ 修改源码
  ✗ 修改 CONTRACT
  ✗ 直接操作 GitHub API
```

---

## 7. 工作流详解

### 7.1 完整生命周期

```
Phase 1: Planning（规划）
─────────────────────────
人类: "目标是构建一个支持 JWT 登录的 FastAPI 系统"
   │
   ▼
Orchestrator:
   ├─ 问: "PostgreSQL 还是 SQLite?"
   ├─ 问: "bcrypt 还是 argon2?"
   └─ 问: "需要 refresh token 吗?"
   │
   ▼
人类: "PostgreSQL, bcrypt, 需要"
   │
   ▼
Orchestrator:
   ├─ 生成 CONTRACT.md（锁定）
   ├─ 生成 02-ready/F-001.md
   └─ 生成 02-ready/F-002.md
   │
   ▼
Phase 2: Implementation（实现）
────────────────────────────
Worker:
   ├─ mv F-001.md 03-running/
   ├─ Fresh Start
   ├─ 先写 tests/test_auth.py
   ├─ 再写 src/routers/auth.py
   ├─ pytest → pass
   ├─ git commit
   ├─ 填写 Handoff
   └─ mv F-001.md 04-review/
   │
   ▼
Phase 3: Validation（验证）
──────────────────────────
Validator:
   ├─ 读 CONTRACT.md VAL-001/002
   ├─ 运行 pytest（不看实现代码）
   ├─ 运行 mypy / ruff
   ├─ 发现: blocking issue（缺少 refresh_token）
   ├─ 填写 Validation Report
   ├─ 创建 06-fix/F-001-fix-001.md
   └─ mv F-001.md archive/
   │
   ▼
Phase 4: Fix（修复）
───────────────────
Worker:
   ├─ mv F-001-fix-001.md 03-running/
   ├─ 读父任务 Validation Report
   ├─ 修复 refresh_token
   ├─ 更新测试
   ├─ git commit
   ├─ 填写 Handoff
   └─ mv F-001-fix-001.md 04-review/
   │
   ▼
Phase 5: Re-validation（重新验证）
────────────────────────────────
Validator:
   ├─ 重新运行测试
   ├─ 通过（零 blocking）
   └─ mv F-001-fix-001.md 05-done/
   │
   ▼
Phase 6: PR（提交）
───────────────────
PR Author:
   ├─ 检测 M1 所有 Features 在 05-done/
   ├─ 汇总 git diff --stat
   ├─ 汇总 pytest --cov
   ├─ 汇总 Validation Reports
   ├─ 生成 Contract Coverage 矩阵
   ├─ 标注 Review Notes
   └─ 生成 07-pr/PR-M1.md
   │
   ▼
人类:
   ├─ 复制粘贴到 GitHub
   ├─ 审查 Review Notes
   ├─ 合并 PR
   └─ mv PR-M1.md 08-merged/
   │
   ▼
🎉 Mission Complete
```

### 7.2 关键设计决策

#### 决策 1: 为什么禁止 Direct Communication？

```
Direct Communication（点对点）:
  Agent A ──► Agent B
  Agent B ──► Agent C
  Agent C ──► Agent A

  问题: 状态在多个会话漂移，没有 single source of truth
  结果: 长周期任务中，谁也不知道当前真实状态

Missions（中心化编排）:
  Orchestrator
       │
       ├──► Worker
       │     │
       │     └──► Git + Handoff
       │
       └──► Validator
             │
             └──► Validation Report

  优势: 所有状态在文件系统中，任何时刻可审计
```

#### 决策 2: 为什么先写 CONTRACT 再写代码？

```
传统流程:
  写代码 ──► 写测试 ──► "测试确认了我的实现"

  问题: 测试被实现塑造，成为"对实现的追认"
  结果: 测试通过了，但可能没覆盖边界条件

Missions 流程:
  锁定 CONTRACT ──► 写测试（覆盖 CONTRACT）──► 写实现（满足测试）

  优势: 人为切断"实现细节回流到验收标准"的路径
  结果: 测试是真正独立的验证
```

#### 决策 3: 为什么用文件系统而不是数据库？

```
数据库状态机:
  ┌─────────┐     ┌─────────┐
  │  MySQL  │     │  Redis  │
  │  table  │     │  queue  │
  └────┬────┘     └────┬────┘
       │               │
       └───────┬───────┘
               ▼
         ┌──────────┐
         │  Python  │
         │  script  │
         └────┬─────┘
              │
              ▼
         需要安装、配置、维护

文件系统状态机:
  ┌─────────┐
  │  ls     │
  │  mv     │
  │  cat    │
  └────┬────┘
       │
       ▼
  零依赖、可版本控制、可审计、可迁移
```

---

## 8. 与现有方案对比

### 8.1 与 Claude Code /goal 对比

| 维度 | /goal | Missions |
|------|-------|---------|
| 状态存储 | Claude 上下文（不可见） | 文件系统（纯文本） |
| 角色分离 | ❌ 无 | ✅ 三角色强制分离 |
| 验证契约 | ❌ 无 | ✅ 先验锁定 |
| Fresh Context | ❌ 依赖历史 | ✅ 强制 Fresh Start |
| 串行约束 | ❌ 无 | ✅ 文件系统级串行 |
| 可审计性 | ❌ 差 | ✅ 完整 Handoff + Report |
| 可迁移性 | ❌ 绑定会话 | ✅ 任何 Claude 实例可续跑 |
| 自动化程度 | 半自动 | 全自动（除 PR 合并） |

### 8.2 与 AutoGPT / LangChain Agent 对比

| 维度 | AutoGPT | LangChain | Missions |
|------|---------|-----------|---------|
| 目标 | 通用任务自动化 | LLM 应用构建 | 软件工程交付 |
| 状态管理 | 内存/向量数据库 | 链式调用 | 文件系统状态机 |
| 验证机制 | 自我评估 | 回调函数 | 独立 Validator |
| 契约机制 | ❌ 无 | ❌ 无 | ✅ 先验 CONTRACT |
| 人类介入 | 频繁（需要监督） | 中等 | 最少（milestone 边界） |
| 可解释性 | 低 | 中 | 高（完整文件记录） |

### 8.3 与 Factory 原版 Missions 对比

| 维度 | Factory Missions | 本 Markdown 原型 |
|------|-----------------|-----------------|
| 实现语言 | Python + 内部系统 | 纯 Markdown + 文件系统 |
| 部署方式 | 企业级 SaaS | 任何 Claude Code 实例 |
| 状态机 | 内部数据库 | 文件夹 |
| 通信模式 | 5 种（用 4 种） | 同上 |
| 核心设计 | 完全一致 | 完全一致 |
| 适用场景 | 企业级长周期任务 | 个人/小团队项目 |

---

## 9. 局限性与未来

### 9.1 当前局限性

1. **PR 合并仍需人工**：Agent 无法直接操作 GitHub API（安全考虑）
2. **冲突解决**：并发修改同一文件时，依赖 Git 合并，Agent 无法自动解决复杂冲突
3. **外部依赖**：数据库迁移、第三方 API 密钥等仍需人工配置
4. **创造性设计**：架构层面的重大决策仍需人类（Orchestrator 只能做范围限定内的规划）

### 9.2 未来方向

```
v0.2: 增加 Milestone 依赖图（DAG）
  └── 02-ready/ 支持任务依赖，自动拓扑排序

v0.3: 增加 Token 预算管理
  └── README.md 中增加预算看板，超限自动暂停

v0.4: 增加多模型支持
  └── Worker 用 Claude，Validator 用 GPT-4，Orchestrator 用 Gemini

v0.5: 增加 CI/CD 集成
  └── 07-pr/ 直接触发 GitHub Actions，PR 合并后自动归档

v1.0: 自举式改进
  └── 用 Missions 框架自身来改进 Missions 框架
```

---

## 附录 A: 快速参考卡

### 启动指令
```
请读取 .missions/README.md，执行自举指令，按自动推进规则运行。
当前目标是：【你的目标描述】
```

### 人工干预指令
```
# 任务卡死
mv .missions/03-running/stuck.md .missions/02-ready/

# 跳过验证
mv .missions/04-review/XXX.md .missions/05-done/

# 放弃任务
mv .missions/03-running/XXX.md .missions/archive/
```

### 状态检测
```bash
ls .missions/02-ready/   # 待领取
ls .missions/03-running/ # 进行中
ls .missions/04-review/  # 待验证
ls .missions/06-fix/     # 待修复
ls .missions/07-pr/      # 待提交 PR
```

---

> **"文件夹即状态机，Markdown 即指令，文件移动即流转。"**
>
> — Missions 框架设计哲学
