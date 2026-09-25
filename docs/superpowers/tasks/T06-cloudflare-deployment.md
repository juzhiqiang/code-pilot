# T06 Cloudflare Tunnel Deployment and Acceptance

**目标：** 让手机无需安装 Tailscale、无需 VPS，通过用户域名和 Cloudflare Tunnel 访问电脑上的本地 Gateway。

**依赖：** T04、T05；需要用户已有 Cloudflare 账号和已接入 Cloudflare DNS 的域名。  
**产出：** `scripts/start-user-agent.ps1`、`scripts/start-cloudflared.ps1`、`docs/setup.md`、`docs/compatibility.md` 的部署记录。

## 固定网络设计

```text
手机浏览器
  → https://agent.example.com
  → Cloudflare DNS / HTTPS / Tunnel
  → 电脑 cloudflared.exe
  → http://127.0.0.1:8787
  → Gateway / Agent / Claude / Codex
```

电脑主动建立出站 Tunnel；本地 Gateway 不监听公网；不使用 Tailscale，不购买 VPS，不做路由器端口转发。

## 步骤

- [ ] 安装 Windows `cloudflared` 并执行 `cloudflared --version`；记录版本，不把安装目录写死在代码中。
- [ ] 执行 `cloudflared tunnel login`，在浏览器选择用户域名；确认凭据只写入当前用户 `.cloudflared` 目录。
- [ ] 执行 `cloudflared tunnel create agent-console`，保存 Tunnel UUID；禁止提交 `<TUNNEL-UUID>.json`。
- [ ] 执行 `cloudflared tunnel route dns agent-console agent.example.com`，确认 Cloudflare DNS 创建 Tunnel 路由。
- [ ] 创建 `C:\Users\<user>\.cloudflared\config.yml`：`agent.example.com` 指向 `http://127.0.0.1:8787`，末尾 ingress 使用 `http_status:404`。
- [ ] 编写 `scripts/start-cloudflared.ps1`：执行 `cloudflared tunnel run agent-console`，不回显凭据和环境变量，失败退出码可被任务计划程序捕获。
- [ ] 确认本地 Gateway 的 WebSocket 路径可通过 `wss://agent.example.com/ws` 访问；Cloudflare 面板启用 WebSockets，应用检查认证和 Origin。
- [ ] 可选配置 Cloudflare Access；验证 Access 身份认证与应用的一次性配对 Cookie 同时存在时不会覆盖或绕过应用权限。
- [ ] 配置 Windows 任务计划程序：当前用户登录时依次启动 Agent/Gateway 和 `cloudflared`，失败自动重启；不以 SYSTEM 身份运行依赖个人 CLI 登录的 Agent。
- [ ] 用手机蜂窝网络打开 `https://agent.example.com`，完成配对、创建 Codex/Claude 测试任务、查看 WSS 输出和追加输入。
- [ ] 做断网 30 秒、刷新页面、重启 Gateway、重启 cloudflared 测试；确认任务不因手机断线停止，不因重连重复提交 commandId。
- [ ] 执行设计文档九条验收，记录域名、Tunnel、电脑、CLI、手机浏览器版本和已知限制；清理临时测试 Tunnel 和测试任务。

## 阶段出口

手机在不安装额外网络客户端的情况下通过用户域名访问控制台；HTTPS/WSS、配对、任务创建、实时日志、追加输入、断线恢复和设备撤销均有实测记录。

## 验证命令

```powershell
cloudflared tunnel list
cloudflared tunnel info agent-console
Test-NetConnection agent.example.com -Port 443
```

预期：Tunnel 状态为可用，域名 443 可访问；应用日志确认 Gateway 仍只监听 `127.0.0.1:8787`。
