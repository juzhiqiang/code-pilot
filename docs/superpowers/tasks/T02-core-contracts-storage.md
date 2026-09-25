# T02 Core Contracts, Storage, and Scheduling

**目标：** 建立 provider 无关的会话合同、SQLite 事件存储、命令幂等、项目路径校验和并发调度。

**依赖：** T01 的能力矩阵。  
**产出：** 可独立运行的 TypeScript workspace、共享类型包、Agent 核心单元测试。

## 文件边界

- 创建 `package.json`、`pnpm-workspace.yaml`、`tsconfig.base.json`、`vitest.config.ts`。
- 创建 `packages/contracts/src/sessions.ts`、`events.ts`、`commands.ts`。
- 创建 `apps/agent/src/storage/database.ts`、`events.ts`、`logs.ts`。
- 创建 `apps/agent/src/sessions/manager.ts`、`commands.ts`、`scheduler.ts`。
- 创建 `apps/agent/src/projects/registry.ts`、`isolation.ts`。
- 测试放在 `apps/agent/src/**/*.test.ts` 和 `tests/integration/core.test.ts`。

## 步骤

- [ ] 固定 Node/TypeScript/SQLite/Vitest 依赖并添加 `typecheck`、`test`、`build` 脚本；执行空测试确认 workspace 可运行。
- [ ] 定义 `SessionLifecycle`、`TurnStatus`、`ConnectionStatus` 和 `SessionCapabilities`；能力必须逐项建模，前端不能从 provider 名称推断能力。
- [ ] 定义统一事件 `{sessionId, seq, occurredAt, receivedAt, type, source, payload}`；使用 `sessionId + seq` 唯一约束并支持按游标分页。
- [ ] 定义写命令 `{commandId, sessionId, kind, payload, createdAt}` 和结果状态 `accepted/queued/started/completed/failed/pending_confirmation`。
- [ ] 编写状态测试：手机离线不改变任务状态；没有权威事件时为 `unknown`；一轮完成后会话保持可继续状态。
- [ ] 建立 SQLite 表：sessions、events、commands、projects、devices、audit_log；所有状态迁移通过事务完成。
- [ ] 编写命令幂等测试：相同 `commandId` 只执行一次；模拟执行后确认落盘前崩溃时返回 `pending_confirmation`，禁止自动重复发送。
- [ ] 实现项目白名单和真实路径解析；拒绝越界路径、junction/symlink 越界、未登记项目和不存在目录。
- [ ] 实现五个并发槽、同目录写锁和队列；Git 项目创建 worktree，非 Git 项目等待前一写任务结束。
- [ ] 实现每会话 20 MiB、总计 1 GiB、最长 7 天的日志轮转；历史被清理时返回 gap 而不是静默拼接。

## 阶段出口

核心包不依赖真实 Claude/Codex 账户即可通过单元测试，能表达“可继续、等待授权、未知、缺口、待确认”等状态。

## 验证命令

```powershell
npm run typecheck
npm run test --workspace apps/agent
```

预期：TypeScript 无错误，核心测试全部通过。
