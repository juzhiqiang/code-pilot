# T03 Provider Adapters

**目标：** 将 Codex App Server、Claude Headless/Agent SDK、受管 Windows PTY 和已有会话发现接入 T02 的统一合同。

**依赖：** T01、T02。  
**产出：** `apps/agent/src/adapters/*`、本机 IPC、适配器集成测试。

## 步骤

- [ ] 用 T01 的脱敏 schema 和 stream-json 样本写契约测试：碎片 JSON、通知、工具调用、授权、结束、错误、未知字段和版本不匹配。
- [ ] 实现 `adapters/codex.ts`：通过本机 stdio/Unix socket 优先连接；按当前 CLI 生成的 schema 解析 JSON-RPC；请求和通知使用独立 id/seq；只在能力声明存在时开放 queue/approve/interrupt。
- [ ] 实现 Codex 重连：记录最后事件序号，连接断开时重建订阅并返回 gap/快照；收到 `-32001` 采用指数退避和 jitter，不忙等。
- [ ] 实现 `adapters/claude.ts`：使用 `claude -p --output-format stream-json` 或已验证的 Agent SDK；把 session id、turn、工具事件、结果和退出码映射到统一事件。
- [ ] 实现 Claude `resume` 和 `interrupt`；区分 SIGTERM 导致的未完成轮次与 SDK interrupt 的语义，不能把二者都标记为 completed。
- [ ] 实现 `adapters/pty.ts`：只启动由本 Agent 创建且路径来自 provider 配置的进程；传递 stdin、resize、Ctrl+C 和退出事件，不用 ANSI 文本猜测百分比。
- [ ] 实现 `adapters/discovery.ts`：导入 Codex/Claude 已有会话并标记 `source` 与 capabilities；活跃会话不自动 resume，不确定时显示 unavailable/unknown。
- [ ] 实现 `control/ipc.ts`：Gateway 只能调用已定义的 RPC，不允许传任意 shell 路径、参数数组或工作目录。
- [ ] 用两个隔离真实会话验证创建、追加、中断和结束；模拟五个高频会话确认事件没有串线。
- [ ] 重启 Agent 后验证原生会话重新发现；PTY 会话如果无法确认存活，必须显示 unknown/unavailable，禁止自动重跑。

## 阶段出口

适配器可由 Gateway 通过统一接口创建和控制会话；不支持能力返回明确错误；真实冒烟测试不修改用户已有项目。

## 验证命令

```powershell
npm run test:integration -- tests/integration/adapters.test.ts
```

预期：脱敏契约测试通过；真实 CLI 测试单独执行并记录 provider/版本/会话 id。
