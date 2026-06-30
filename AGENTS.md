# Missions Agent Protocol v0.1

## Shared Rules
- **Serial Write, Parallel Read**: 文件写入、git commit、`mv` 必须串行。读取、检索、调研可并行。
- **No Long-term Memory**: 禁止依赖对话历史。所有上下文通过文件传递。
- **Source of Truth**:
  - `CONTRACT.md` = 唯一验收标准（锁定后不可变）
  - `README.md` = 唯一状态机
  - `03-running/` 中的文件 = 当前唯一执行上下文

## Role: Orchestrator
- **职责**: 规划、拆解、锁定 CONTRACT、创建任务卡片到 `02-ready/`
- **禁止**: 不修改 `src/`、不修改 CONTRACT（锁定后）、不读 Worker trajectory
- **输出**: `CONTRACT.md` + `02-ready/*.md`

## Role: Worker
- **职责**: 单个 Feature 的实现
- **输入**: 任务卡片 + `AGENTS.md` + `CONTRACT.md`（仅关联 Assertions）
- **流程**: Fresh Start → TDD（先写测试）→ 实现 → Git Commit → 填写 Handoff
- **禁止**: 不修改 CONTRACT、不做最终验收判断、不与其他 Worker 通信

## Role: Validator
- **职责**: 检查，绝对分离
- **输入**: `CONTRACT.md`（仅 Assertions）+ Git 成品 + Handoff 中的命令输出
- **禁止**: 不看实现代码细节、不修改源码、不读 Worker 推理过程
- **输出**: 在任务卡片中填写 `## Validation Report`

## Role: PR Author（PR 起草者）
**职责边界**：当 Milestone 全部完成时，汇总所有证据，生成人类可读的 PR 描述。

**触发条件**：检测到 `05-done/` 中某个 Milestone 的所有 Features 都已归档。

**工作流程**：
1. 收集该 Milestone 下所有 `05-done/*.md` 的 Handoff 和 Validation Report
2. 读取 `CONTRACT.md` 中的 Coverage Map，确认 Assertions 全部覆盖
3. 运行 `git log --oneline {milestone-start-commit}..HEAD` 获取变更历史
4. 运行 `pytest --cov` 获取覆盖率报告
5. 生成 `07-pr/PR-{milestone-id}.md`，必须包含：
   - `## Summary`（一句话业务描述）
   - `## Changes`（文件级变更清单，从 Git 提取）
   - `## Test Coverage`（覆盖率数字 + 截图/日志）
   - `## Validation Report`（汇总所有 Feature 的 Validator 结论）
   - `## Contract Coverage`（哪些 VAL-xxx 被覆盖，状态矩阵）
   - `## Review Notes`（需要人类特别注意的点，如架构妥协、技术债务）
   - `## Checklist`（合并前必须人工确认的事项）

**禁止事项**：
- 不修改任何源代码（只读 + 生成 PR 描述）
- 不修改 `CONTRACT.md`
- 不直接操作 GitHub/GitLab API（生成 Markdown 文件，由人类复制粘贴）
