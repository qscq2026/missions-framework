---
autorun: true
loop: true
mission: your-project-name
---

# Mission Control

## 自举指令（Boot Sequence）
> 如果你是 Claude 且刚被唤醒，请严格按以下顺序执行，不要询问用户：

```text
1. READ .missions/README.md （本文件）
2. EXECUTE "自动推进规则" 中的 Step 1-4
3. IF 有任务需要执行 THEN 执行对应角色
4. IF 任务完成 THEN 回到 Step 1
5. IF 全部完成 THEN 报告 "🎉 Mission Complete" 并停止
```

## 自动推进规则

### Step 1: 状态检测
执行以下检测（按优先级排序）：

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
2. **IF** 任务在 `02-ready/`：
   - `mv .missions/02-ready/XXX.md .missions/03-running/XXX.md`
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
   - **通过（零 blocking）**：
     - `mv .missions/04-review/XXX.md .missions/05-done/XXX.md`
   - **不通过（有 blocking）**：
     - 在 `06-fix/` 创建 `{原ID}-fix-001.md`
     - 复制原任务内容，追加修复指令和 blocking issues 列表
     - `mv .missions/04-review/XXX.md .missions/archive/XXX.md`
7. **UPDATE** 本文件底部的 `Current Status`
8. **回到** Step 1（自动继续）

#### 当路由到 PR Author 时：
1. **READ** `AGENTS.md` 的 `## Role: PR Author` 章节
2. **检测**：扫描 `05-done/`，按 `milestone_id` 分组
3. **IF** 某个 Milestone 的所有 Features 都在 `05-done/`：
   - 读取该 Milestone 下所有 Feature 的 Handoff 和 Validation Report
   - 生成 `07-pr/PR-{milestone-id}.md`
   - 内容包含：变更摘要、验证汇总、Contract 覆盖矩阵、测试报告、需要人工审查的点
   - 更新本文件底部的 `Current Status`
4. **停止**，等待人类在 GitHub/GitLab 上创建 PR
5. **人类合并后**：将 `07-pr/PR-xxx.md` `mv` 到 `08-merged/`

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

## 快速看板（自动更新）

| Ready | Running | Review | Fix | Done | PR |
|-------|---------|--------|-----|------|-----|
| <!-- ls 02-ready --> | <!-- ls 03-running --> | <!-- ls 04-review --> | <!-- ls 06-fix --> | <!-- ls 05-done --> | <!-- ls 07-pr --> |

## 人工干预命令
```bash
# 任务卡死，强制回退到 ready
mv .missions/03-running/stuck.md .missions/02-ready/

# 跳过验证，强制完成
mv .missions/04-review/XXX.md .missions/05-done/

# 放弃任务
mv .missions/03-running/XXX.md .missions/archive/
```
