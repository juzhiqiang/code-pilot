# T04 Gateway, Authentication, and Events

**目标：** 提供只监听 localhost 的 Fastify HTTP/WebSocket Gateway，完成设备配对、任务 API、实时事件、权限和背压。

**依赖：** T02、T03。  
**产出：** `apps/gateway/src/*`、Gateway 集成测试。

## 步骤

- [ ] 创建 `apps/gateway/src/server.ts`，监听配置的 loopback 地址，禁止默认绑定 `0.0.0.0`；通过本机 IPC 调用 Agent。
- [ ] 实现 `auth/devices.ts`：本机生成五分钟一次性配对码，单次兑换，限速，设备 cookie，设备列表和撤销。
- [ ] 实现 `auth/origin.ts`：检查 HTTPS Origin、CSRF token、WebSocket Origin 和 cookie；拒绝跨来源写入。
- [ ] 实现项目、会话列表、历史分页、创建、追加输入、授权、中断、结束和恢复接口；每个写入请求都带 `commandId`。
- [ ] 实现 `transport/events.ts`：按 `sessionId + seq` 重放；遇到日志 gap 返回 gap；慢消费者队列超过 1 MiB 时关闭并要求客户端重连。
- [ ] 为输入文本、payload 大小、项目归属、会话控制权和 capability 做服务端校验；错误不能包含 CLI 凭据。
- [ ] 记录设备和控制操作审计，但对任务内容和日志按配额处理；敏感值使用脱敏器。
- [ ] 测试未认证请求、过期配对码、重复配对码、撤销设备、跨来源请求、未授权会话控制、重复 commandId 和恶意路径。
- [ ] 测试 WebSocket 断线、重连游标、重复事件、乱序事件、gap、慢消费者和 Agent 重启。

## 阶段出口

Gateway 可在 localhost 提供完整任务 API 和事件 WebSocket；未认证或无能力的设备无法控制会话；合法客户端可按游标恢复事件。

## 验证命令

```powershell
npm run test --workspace apps/gateway
npm run test:integration -- tests/integration/gateway.test.ts
```

预期：所有认证、权限、幂等、重连测试通过；启动日志确认监听地址为 `127.0.0.1`。
