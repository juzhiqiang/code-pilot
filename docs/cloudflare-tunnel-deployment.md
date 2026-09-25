# 手机远程访问本地 Claude Code / Codex 控制台

本文说明手机和电脑不在同一网络时，如何通过 Cloudflare Tunnel 访问电脑上的本地监控后端。

## 1. 方案结论

本方案使用 Cloudflare Tunnel 作为**出站连接型内网穿透**：电脑主动连接 Cloudflare，手机通过 HTTPS 域名访问 Cloudflare，Cloudflare 再通过已建立的 Tunnel 转发到电脑本地服务。

不需要购买 VPS，也不需要手机安装 Tailscale。

```text
手机浏览器
    │ HTTPS / WSS
    ▼
Cloudflare DNS + HTTPS + Tunnel
    │ 已建立的加密 Tunnel
    ▼
电脑 cloudflared.exe
    │ localhost
    ▼
本地 Web Gateway 127.0.0.1:8787
    │ 本机 IPC
    ▼
Session Agent
    ├── Codex App Server / Codex CLI
    └── Claude Code / Agent SDK / PTY
```

Cloudflare Tunnel 的本质是反向隧道和反向代理，不是传统路由器端口映射，也不会把手机和电脑放进同一个局域网。

## 2. 组件职责

### 电脑本地后端

运行在 Windows 电脑上，负责：

- 手机端 API 和 WebSocket 实时连接
- 任务列表、任务创建和任务控制
- Codex、Claude Code 会话管理
- 日志保存、断线重连和事件回放
- 登录、配对码、设备撤销和写入权限

后端只监听本机地址：

```text
127.0.0.1:8787
```

Claude Code、Codex 的登录凭据和会话数据始终留在电脑上，不发送到 Cloudflare 或手机端。

### cloudflared

`cloudflared` 是 Cloudflare 官方提供的连接器程序，安装并运行在电脑上。它不是 VPS，也不是部署在 Cloudflare 上的 Node.js 后端。

它负责：

- 主动向 Cloudflare 建立加密出站连接
- 将公网域名请求转发到 `127.0.0.1:8787`
- 维持连接和断线重连
- 不要求电脑开放公网入站端口

### Cloudflare

Cloudflare 云端负责：

- 域名 DNS
- HTTPS 证书
- Tunnel 流量转发
- 可选的 Cloudflare Access 身份认证和访问策略

## 3. 完整请求链路

### 手机访问页面

```text
手机打开 https://agent.example.com
    ↓
Cloudflare 接收 HTTPS 请求
    ↓
Cloudflare 通过 Tunnel 转发
    ↓
电脑 cloudflared.exe 接收请求
    ↓
127.0.0.1:8787 本地 Web Gateway
    ↓
返回页面和静态资源
```

### 手机创建任务

```text
手机点击“创建任务”
    ↓ HTTPS
Cloudflare
    ↓ Tunnel
cloudflared.exe
    ↓ localhost
Web Gateway
    ↓ 本机 IPC
Session Agent
    ↓
启动 Codex 或 Claude Code
    ↓
保存任务状态和日志
    ↓ WebSocket / HTTPS
手机实时看到输出
```

### 手机追加输入

```text
手机发送新指令
    ↓
Gateway 验证登录、设备和会话能力
    ↓
Session Agent 判断发送方式
    ├── Codex：App Server / queue
    ├── Claude：resume / Agent SDK
    └── PTY：写入受管终端 stdin
    ↓
CLI 继续执行并产生事件
    ↓
WebSocket 推送到手机
```

### 手机断线后重连

```text
手机暂时断网
    ↓
电脑上的任务继续运行
    ↓
手机重新打开页面
    ↓
按最后事件序号补读历史
    ↓
恢复实时 WebSocket
```

手机离线不会自动停止电脑任务。电脑关机或休眠时，任务无法继续运行。

## 4. 部署前提

- 一个 Cloudflare 账号
- 一个已接入 Cloudflare DNS 的域名，例如 `example.com`
- 一台运行 Windows 的电脑
- 电脑可以访问互联网并建立出站 HTTPS 连接
- 本地监控后端运行在 `127.0.0.1:8787`
- 手机可以访问公网 HTTPS

不要求：

- VPS
- 公网 IP
- 路由器端口转发
- 手机安装 Tailscale
- 电脑向公网开放 8787 端口

## 5. 创建 Cloudflare Tunnel

### 5.1 安装 cloudflared

从 Cloudflare 官方渠道下载安装 Windows 版本的 `cloudflared`，确认命令可用：

```powershell
cloudflared --version
```

建议将 `cloudflared.exe` 放到固定目录，并让 Windows 防火墙允许它建立出站连接。

### 5.2 登录 Cloudflare

在电脑执行：

```powershell
cloudflared tunnel login
```

命令会打开浏览器，让你选择 Cloudflare 账号和域名。登录成功后，凭据会保存在当前 Windows 用户的 `.cloudflared` 目录中。

### 5.3 创建 Tunnel

```powershell
cloudflared tunnel create agent-console
```

记下输出中的 Tunnel UUID。命令会生成一个凭据 JSON 文件，通常位于：

```text
C:\Users\你的用户名\.cloudflared\<TUNNEL-UUID>.json
```

该 JSON 是敏感凭据，不要提交到 Git，不要发给手机，也不要放到网站静态目录。

### 5.4 配置 DNS 路由

下面假设使用子域名 `agent.example.com`：

```powershell
cloudflared tunnel route dns agent-console agent.example.com
```

Cloudflare 会为该主机名创建指向 Tunnel 的 DNS 记录。手机之后访问：

```text
https://agent.example.com
```

### 5.5 创建配置文件

创建：

```text
C:\Users\你的用户名\.cloudflared\config.yml
```

内容：

```yaml
tunnel: <TUNNEL-UUID>
credentials-file: C:\Users\你的用户名\.cloudflared\<TUNNEL-UUID>.json

ingress:
  - hostname: agent.example.com
    service: http://127.0.0.1:8787
  - service: http_status:404
```

注意：`service` 指向的是电脑本地后端，不是 Cloudflare Workers，也不是公网地址。

### 5.6 启动 Tunnel

先确认本地后端已经运行，再执行：

```powershell
cloudflared tunnel run agent-console
```

然后在手机浏览器打开：

```text
https://agent.example.com
```

如果暂时没有正式域名，也可以使用临时测试隧道：

```powershell
cloudflared tunnel --url http://127.0.0.1:8787
```

临时地址只适合测试，不适合正式使用，因为地址不稳定、没有固定的设备策略和正式访问控制。

## 6. WebSocket 配置

监控页面需要 WebSocket 推送实时日志。Cloudflare Tunnel 支持 WebSocket，但应用和代理都必须保持长连接。

后端连接地址应使用：

```text
wss://agent.example.com/ws
```

要求：

- 页面使用 HTTPS 时，WebSocket 使用 WSS
- Cloudflare 面板确认 WebSockets 已启用
- 后端校验登录状态和 Origin
- 后端设置连接空闲超时和最大消息大小
- 断线时由前端按事件序号重新连接和补读
- 不把 Claude/Codex 原始控制协议直接暴露给手机

## 7. 认证与安全边界

Cloudflare Tunnel 只负责把请求送到电脑，不能代替应用自身的任务权限控制。至少需要两层保护：

### Cloudflare 层

可选配置 Cloudflare Access：

- 限制允许访问的邮箱或身份提供商
- 增加一次登录验证
- 限制可访问的域名和路径
- 记录外部访问日志

### 应用层

应用仍然需要：

- 首次配对一次性代码
- 配对码五分钟过期且只能使用一次
- Secure、HttpOnly、SameSite Cookie
- 设备列表和设备撤销
- HTTP 写操作的 CSRF/Origin 检查
- WebSocket 的认证和 Origin 检查
- 会话级别的控制权限
- 命令和输入长度限制
- 不允许手机直接提交任意 shell 可执行路径
- 项目目录白名单和真实路径校验
- 日志中的常见密钥脱敏

不要直接执行以下不安全方式：

```text
手机 → 公网 IP:8787
```

也不要把 `cloudflared` 凭据 JSON、CLI 登录信息或 `.env` 文件放进前端资源目录。

## 8. Windows 常驻运行

正式使用时需要让两个进程随用户登录自动启动：

```text
本地 Web Gateway / Session Agent
cloudflared tunnel run agent-console
```

可使用 Windows 任务计划程序：

- 触发器：当前用户登录时
- 运行身份：当前 Windows 用户
- 不勾选“以最高权限运行”，除非实际需求明确要求
- 工作目录固定为项目目录
- 标准输出和错误输出写入本地日志
- 进程退出时自动重启

不要把依赖个人 Claude/Codex 登录状态的 Agent 默认配置为 SYSTEM 服务运行。

## 9. 成本说明

通常不需要额外购买 VPS：

| 项目 | 是否需要额外付费 |
| --- | --- |
| `cloudflared` 软件 | 不需要 |
| Cloudflare Tunnel 基础能力 | 通常不需要 |
| Cloudflare DNS | 通常不需要 |
| 已有域名 | 只需承担域名注册/续费费用 |
| Cloudflare Access | 基础能力可能免费，高级能力按账号方案计费 |
| Claude Code / Codex | 按各自账号、订阅或 API 用量计费 |
| VPS | 本方案不需要 |

具体费用以当前 Cloudflare 账号方案和服务条款为准。

## 10. 故障排查

### 手机打不开域名

检查：

```powershell
cloudflared tunnel list
cloudflared tunnel info agent-console
```

确认：

- DNS 主机名属于当前 Cloudflare 账号
- Tunnel 正在运行
- 本地后端确实监听 `127.0.0.1:8787`
- Windows 防火墙允许 `cloudflared` 出站
- 域名证书状态正常

### 页面能打开但没有实时输出

检查：

- 浏览器是否使用 HTTPS
- WebSocket 地址是否为 `wss://`
- Cloudflare WebSockets 是否启用
- 后端 `/ws` 路径是否存在
- 后端是否因为 Origin 或认证失败主动关闭连接

### 电脑重启后无法访问

检查：

- 本地 Agent 是否随用户登录启动
- `cloudflared` 是否随用户登录启动
- Tunnel 配置文件路径是否正确
- 凭据 JSON 是否仍存在且权限正确
- 本地端口是否被其他程序占用

### 任务还在电脑运行，但手机显示离线

这通常表示手机与 Web Gateway 的连接断开，不代表任务停止。重新打开页面后，应通过事件序号、数据库快照或提供商会话记录恢复状态。

## 11. 与其他方式的比较

| 方式 | 是否需要 VPS | 是否需要手机安装软件 | 是否需要开放入站端口 | 适合本项目 |
| --- | --- | --- | --- | --- |
| Cloudflare Tunnel | 否 | 否 | 否 | 推荐 |
| Tailscale | 否 | 是 | 否 | 当前不采用 |
| 路由器端口映射 | 否 | 否 | 是 | 不推荐 |
| VPS 反向代理 | 是 | 否 | 电脑需出站连接 | 备用方案 |
| 临时 ngrok 地址 | 否 | 否 | 否 | 仅测试 |

## 12. 最终部署形态

```text
手机
  └── 浏览器访问 https://agent.example.com

Cloudflare
  ├── DNS
  ├── HTTPS
  ├── Tunnel
  └── 可选 Access

Windows 电脑
  ├── cloudflared.exe
  ├── Web Gateway: 127.0.0.1:8787
  ├── Session Agent
  ├── Codex App Server / Codex CLI
  └── Claude Code / Agent SDK / PTY
```

一句话概括：**电脑上的 `cloudflared` 主动连接 Cloudflare，手机通过 Cloudflare 域名访问，Cloudflare 再把请求转回电脑 localhost 上的监控后端。**
