---
status: locked
locked_at: 2026-06-28T10:00:00
version: 1.0
---

# Validation Contract

> 🔒 **LOCKED** — 此文件已锁定，不可修改。
> 任何变更必须通过 Fix Feature 流程（在 `06-fix/` 中创建修复任务）。

## 全局约束
- [ ] 所有 API 返回 JSON，符合 HTTP 语义
- [ ] 所有数据库操作在事务中
- [ ] 测试覆盖率 ≥ 80%

## Assertions

### VAL-001: User Registration
- **Behavior**: 新用户提交 `email` + `password`，系统创建账户，返回 201。
- **Tool**: pytest
- **Evidence**: 测试日志 `assert response.status_code == 201` + 数据库确认

### VAL-002: User Login
- **Behavior**: 已注册用户提交正确凭证，返回 JWT `access_token`。
- **Tool**: pytest
- **Evidence**: 测试日志 `assert "access_token" in response.json()`

## Coverage Map
| Feature | Assertions |
|---------|-----------|
| F-001 | VAL-001, VAL-002 |
| F-002 | VAL-003, VAL-004 |
