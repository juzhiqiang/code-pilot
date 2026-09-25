# Mobile Agent Console Tasks

这些文件把 [实施计划](../plans/2026-09-25-mobile-agent-console.md) 拆成可执行任务。每个任务完成后运行自己的验证命令，再进入下一个任务。

## 执行顺序

```text
T01 接入能力验证
  ↓
T02 工作区、合同、存储和调度
  ↓
T03 Codex / Claude / PTY 适配器
  ↓
T04 Gateway、认证和实时事件
  ↓
T05 手机 PWA
  ↓
T06 Cloudflare Tunnel、Windows 常驻和外网验收
```

T01 是风险门：如果某个 CLI 的原生控制能力不可用，必须在 `docs/compatibility.md` 中记录降级路径，再继续 T02/T03。T06 不使用 Tailscale，不购买 VPS；它使用用户域名、Cloudflare DNS、Cloudflare Tunnel 和电脑上的 `cloudflared.exe`。

## 文件索引

| 文件 | 产出 |
| --- | --- |
| [T01](./T01-provider-capability.md) | CLI、App Server、Headless、PTY 能力矩阵和脱敏样本 |
| [T02](./T02-core-contracts-storage.md) | TypeScript 工作区、共享合同、SQLite、命令幂等和调度 |
| [T03](./T03-provider-adapters.md) | Codex、Claude、PTY、已有会话发现适配器 |
| [T04](./T04-gateway-auth-events.md) | Fastify Gateway、设备配对、HTTP/WS 和事件重放 |
| [T05](./T05-mobile-pwa.md) | 手机优先 PWA、任务列表、详情、新建和断线恢复 |
| [T06](./T06-cloudflare-deployment.md) | Cloudflare Tunnel、Windows 自动启动、蜂窝网络验收 |

## 完成规则

- 任务中的复选框按实际执行更新，不把设计完成当作实现完成。
- 任何真实 CLI 冒烟测试使用隔离目录和无敏感内容任务。
- 不把 CLI 凭据、Cloudflare Tunnel credentials JSON、日志中的密钥或真实聊天内容提交到 Git。
- 每个任务完成后执行文件末尾的验证命令并保存输出摘要。
