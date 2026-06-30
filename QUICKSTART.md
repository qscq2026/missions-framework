# Missions 框架 — 快速入门

## 1. 安装
将 `.missions/` 文件夹复制到你的项目根目录。

```bash
cp -r missions-framework/.missions ./.missions
```

## 2. 启动
在 Claude Code 中输入一句话：

> **"请读取 `.missions/README.md`，执行自举指令，按自动推进规则运行。当前目标是：【你的目标描述】"**

## 3. 观察
Claude 会自动：
1. 变成 Orchestrator，提出 1-3 个澄清问题
2. 锁定 `CONTRACT.md`
3. 创建任务卡片到 `02-ready/`
4. 变成 Worker，领取任务 → 实现 → 验证 → 修复 → PR

## 4. 你只需在 4 种情况下说话

| 情况 | 你说什么 |
|------|---------|
| Orchestrator 问澄清问题 | 回答技术选型（如"PostgreSQL + bcrypt"） |
| 任务卡死 | "任务卡死，请回退到 ready 重新评估" |
| 中途改需求 | "放弃 X 功能，改为 Y。请重新规划" |
| PR 生成后 | 复制粘贴到 GitHub，审查后合并 |

## 5. 文件夹含义速查

| 文件夹 | 含义 | 谁操作 |
|--------|------|--------|
| `00-orchestrate/` | Orchestrator 规划草稿 | Claude |
| `01-contract/` | 锁定后的契约归档 | Claude |
| `02-ready/` | Worker 待领取 | Claude (mv) |
| `03-running/` | 进行中（串行，只一个） | Claude (mv) |
| `04-review/` | 待 Validator 验证 | Claude (mv) |
| `05-done/` | 已完成 | Claude (mv) |
| `06-fix/` | 待修复（blocking issues） | Claude (mv) |
| `07-pr/` | 待提交 PR（人类操作） | 你 |
| `08-merged/` | 已合并 PR | 你 (mv) |
| `archive/` | 历史归档 | Claude |

## 6. 核心设计原则

- **文件夹即状态机**：文件移动 = 状态流转
- **Markdown 即指令**：每个文件包含自描述的执行规则
- **零脚本**：没有 Python/JS，纯文件系统驱动
- **三角色分离**：Orchestrator 规划、Worker 实现、Validator 验证、PR Author 汇总
- **先验契约**：代码之前锁定验收标准，不可事后修改
