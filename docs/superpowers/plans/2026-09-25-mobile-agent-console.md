# 手机端多会话控制台实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在手机上通过安全外网连接监控和控制 Windows 上的多个 Claude Code / Codex 会话，并创建新任务。

**Architecture:** 手机 PWA 连接本地网关，网关通过本机 IPC 连接持久会话管理进程。原生会话适配器优先，Windows PTY 补充；SQLite 存储索引和可重放事件。

**Tech Stack:** TypeScript、React、Vite、Fastify、WebSocket、SQLite、node-pty、xterm.js、Cloudflare Tunnel、cloudflared、Vitest、Playwright。

---

本计划基于本机 CLI 帮助和已成功读取的 Codex App Server、Claude Code Headless 官方文档制定，但仍不是已验证的 SDK 集成代码。先执行 Task 1 的兼容性验证，再将每个实现阶段展开为代码级步骤，避免把尚未证实的协议写死。设计依据：`docs/superpowers/specs/2026-09-25-mobile-agent-console-design.md`。

## 目录与责任边界

```text
apps/web/src/
  pages/SessionList.tsx           会话列表和筛选
  pages/SessionDetail.tsx         活动、终端、输入与授权
  pages/CreateSession.tsx         新建任务
  pages/PairDevice.tsx            手机配对
  components/Terminal.tsx         xterm 生命周期及手机键盘
  data/events.ts                 重连游标、去重和缺口
  data/client.ts                 已认证 HTTP 客户端
apps/gateway/src/
  server.ts                      HTTP 与 WebSocket 服务
  auth/devices.ts                 配对、会话过期及撤销
  auth/origin.ts                  Origin 与 CSRF 检查
  routes/sessions.ts             列表、创建、历史与控制
  routes/projects.ts             已登记项目列表
  transport/events.ts            游标重放、背压
  transport/agent.ts             仅本机的已认证 IPC 客户端
apps/agent/src/
  main.ts                        常驻管理进程入口
  control/ipc.ts                 仅本机已认证控制入口
  sessions/manager.ts            会话生命周期、恢复和写入所有者
  sessions/commands.ts           命令意图、去重和结果核实
  sessions/scheduler.ts          并发限额与队列
  projects/registry.ts           项目真实路径校验
  projects/isolation.ts          worktree 与同目录写入排队
  adapters/codex.ts              Codex 原生协议
  adapters/claude.ts             Claude 发现、结构化运行和附着
  adapters/pty.ts                ConPTY 输入、输出与进程归属
  adapters/discovery.ts          外部会话发现与能力更新
  storage/database.ts           SQLite 初始化与事务
  storage/events.ts             事件落盘与分页重放
  storage/logs.ts                日志轮转与配额
packages/contracts/src/
  sessions.ts                    能力、状态和会话数据
  events.ts                      统一事件与缺口定义
  commands.ts                    写命令和结果定义
tests/fixtures/                  脱敏协议样本和模拟 CLI
tests/integration/               适配器、重连和权限测试
tests/e2e/                      手机页面与完整用户流程
scripts/start-user-agent.ps1      当前用户启动入口
  scripts/start-cloudflared.ps1   启动 Cloudflare Tunnel
  docs/setup.md                    本机安装、配对、外网访问与卸载
  docs/cloudflare-tunnel-deployment.md  Cloudflare Tunnel 部署和排障
docs/compatibility.md            已测试版本、能力和退路
```

## Task 1：验证真实接入能力

**文件：** `docs/compatibility.md`、`tests/fixtures/codex/`、`tests/fixtures/claude/`。在实际开发阶段创建隔离临时项目，使用无敏感内容的测试会话。

- [ ] 记录 CLI/Node/Windows 版本，确认当前 CLI 的登录方式可用于选定运行模式；不得推断所有集成模式都复用同一种订阅或认证。
- [ ] 使用 `codex app-server generate-json-schema --out tests/fixtures/codex/schema` 导出本机协议，核实初始化、列举、订阅、历史、发送、授权、中断和恢复字段。
- [ ] 验证连接已有 Codex 服务与启动独立服务的差别；不得为了探测而重启用户现有 daemon。验证 `queue` 的入队与实际开始执行分别如何确认。远程 WebSocket 仅在 TLS 下启用，并按文档配置 capability token 或 signed bearer token；模拟过载 `-32001` 后采用指数退避和 jitter。
- [ ] 在测试 Claude 会话中验证 `agents --json` 的来源和字段；验证后台会话 `logs`/`attach`，不推断任意交互会话均可附着。
- [ ] 验证 Claude `-p/--print` 的 `stream-json` 增量输出、多轮输入、工具事件和授权回传；验证 `--continue`/`--resume` 及 SDK `interrupt()`。若结构化授权不可用，新建交互会话采用受管 PTY，界面保留原始授权提示；不把 `--bg` 与 `-p` 组合。
- [ ] 验证 Windows node-pty 的安装、中文目录、中文多行输入、终端 resize、Ctrl+C 和子进程退出；不终止用户真实会话。
- [ ] 为每个会话来源填写设计文档中的能力矩阵，保存脱敏样本。预期结果是逐项通过/不支持及原因，不以“能启动”代替“可控制”。

**阶段出口：** 至少两种 CLI 都能在隔离项目中创建、观察输出、继续输入和中断；已有会话明确区分原生可控、只读和可恢复。如关键能力不满足，先更新设计再扩展实现。

## Task 2：建立合同、存储和调度核心

**文件：** `packages/contracts/src/*`、`apps/agent/src/storage/*`、`apps/agent/src/sessions/*`、`apps/agent/src/projects/*`。

- [ ] 初始化工作区和依赖，固定版本；建立 typecheck、test、build 脚本。当前目录无 Git，开发阶段初始化后再记录阶段提交。
- [ ] 编写状态测试：一轮结束后会话可继续；手机 offline 不使任务 failed；缺少权威事件时保持 unknown。
- [ ] 定义 provider 无关合同，区分 readHistory/streamEvents/send/queue/approve/interrupt/resume/terminal 能力。
- [ ] 建立 SQLite 会话、事件、命令意图和设备审计表；测试 sessionId + seq 唯一与历史分页。
- [ ] 测试相同 commandId 不重复提交；模拟“外部执行后、确认落盘前”崩溃，必须返回待确认而非自动重试。
- [ ] 实现项目注册与真实路径校验；测试 junction/symlink 越界、带空格中文目录及不存在目录。
- [ ] 实现默认五个并行槽与同目录写入锁；Git 项目使用 worktree 隔离，非 Git 项目排队。测试结束任务释放槽位与锁。
- [ ] 实现日志轮转和配额；测试保留边界出现 gap，不删除提供商原始会话记录。

**验证命令：** `npm run typecheck`、`npm run test --workspace apps/agent`。期望核心测试全部通过，且不需要连接真实提供商账户。

## Task 3：实现两种提供商与 PTY 适配器

**文件：** `apps/agent/src/adapters/*`、`apps/agent/src/main.ts`、`apps/agent/src/control/ipc.ts`、`tests/integration/adapters.test.ts`。

- [ ] 用 Task 1 的脱敏样本编写契约测试，覆盖碎片 JSON、工具输出、授权、结束、错误与不支持事件；未知字段保留兼容性。
- [ ] 实现 Codex 适配器，协议按本机导出类型生成；为实验性功能检查版本和能力，不硬编码未经验证的 RPC 名称。
- [ ] 实现 Claude 原生发现与选定运行模式；将原生 session ID 与系统 session ID 显式映射。
- [ ] 实现 PTY 适配器，结构化进度不可用时提供终端流与真实连接/退出状态，不用 ANSI 文本猜测完成百分比。
- [ ] 实现仅本机认证 IPC；前端网关不能传任意可执行路径或 shell 字符串。
- [ ] 实现外部会话只读导入、能力更新、写入所有者与安全恢复；测试活跃会话禁止后台自动 resume。
- [ ] 使用两个隔离真实会话验收完整一轮、追加一轮和中断；模拟五会话高频输出，验证不串线。
- [ ] 测试管理进程重启后的原生重新发现与 PTY 不确定状态；禁止无证据将遗留任务标成 completed。

**验证命令：** `npm run test:integration`。真实 CLI 冒烟测试单独显式运行，避免单元测试消耗模型额度或修改个人项目。

## Task 4：实现认证网关与实时传输

**文件：** `apps/gateway/src/*`、`tests/integration/gateway.test.ts`。

- [ ] 配对码只在本机生成，五分钟内单次兑换并限速；实现设备撤销和安全 cookie。
- [ ] 为项目、会话列表、创建、历史、命令与事件订阅建立接口；所有任务接口复用相同认证校验。
- [ ] 写操作检查 CSRF/Origin；WebSocket 检查认证和 Origin，设备撤销后关闭相关连接。
- [ ] 事件订阅按 seq 重放；测试重复、乱序、日志清理缺口和慢消费者。发送队列超过 1 MiB 后关闭连接，让客户端重连补读。
- [ ] 对输入尺寸、项目归属、会话控制权和命令能力进行验证，拒绝不支持操作；错误包含可展示的原因，不泄露凭据。
- [ ] 测试未认证访问、跨来源请求、撤销后操作、重复写命令与恶意目录路径。

**验证命令：** `npm run test --workspace apps/gateway`、`npm run test:integration`。期望未授权的 HTTP 与 WS 均失败，合法重连可以补读。

## Task 5：实现手机界面

**文件：** `apps/web/src/*`、`tests/e2e/mobile.spec.ts`。

- [ ] 实现设备配对、连接提示和任务首页。每卡展示工具、项目、轮次状态、当前活动、最后活动时间与可控性。
- [ ] 实现新建流程：工具、登记项目、模型/权限配置、任务文本。重复点击使用相同 commandId；不开放任意 shell 启动字段。
- [ ] 实现时间线、分页/虚拟滚动、可用文件变更与任务清单；区分“步骤计数”与未经证实的百分比。
- [ ] 实现终端视图和手机快捷键，离开页面释放渲染资源但不结束服务器任务。
- [ ] 实现输入、入队回执、授权、当前轮中断与会话结束；按能力显示按钮，支持只读/待确认原因。
- [ ] 实现静态 PWA 外壳与断线提示；离线时禁用发送，任务数据不进入 Service Worker 缓存。
- [ ] 自动化覆盖窄屏布局、列表切换、任务创建、授权拒绝、断线重连和重复点击；真实设备检查中文输入法、软键盘与安全区。

**验证命令：** `npm run test:e2e`、`npm run build`。期望页面能在窄屏完成完整流程，没有被软键盘遮挡的发送按钮。

## Task 6：配置 Cloudflare Tunnel 外网访问

**文件：** `scripts/start-user-agent.ps1`、`scripts/start-cloudflared.ps1`、`docs/setup.md`、`docs/cloudflare-tunnel-deployment.md`、`docs/compatibility.md`。

- [ ] 安装 Windows 版 `cloudflared`，用 `cloudflared tunnel login`、`cloudflared tunnel create agent-console`、`cloudflared tunnel route dns agent-console agent.example.com` 创建固定 Tunnel 和域名路由。
- [ ] 编写 `config.yml`，将 `agent.example.com` 映射到 `http://127.0.0.1:8787`，并用 `cloudflared tunnel run agent-console` 验证 HTTPS 与 WebSocket。
- [ ] 网关仅监听 loopback；配置本地 Agent 与 `cloudflared` 当前用户登录启动、日志和失败重启。启动脚本不打印 CLI 凭据，Tunnel credentials JSON 不进入 Git 或前端资源。
- [ ] 配置应用配对认证；按需要增加 Cloudflare Access，验证 Access 与应用 Cookie 不冲突。
- [ ] 手机切换蜂窝网络测试 `https://agent.example.com`、HTTPS/WSS、配对、两个真实工具任务与追加输入。
- [ ] 手机断网 30 秒后恢复，核实无重复任务；重启 Web 网关，核实会话管理进程任务继续。
- [ ] 执行设计文档全部九条验收，记录电脑/CLI/手机系统版本、实际支持能力和已知限制。
- [ ] 文档说明电脑休眠、账号限流、日志配额、设备撤销、停止工具、卸载以及已有只读会话如何恢复。

**完成定义：** 九条验收有结果；不支持的原生能力明确降级；手机能够通过外网操作两种 CLI 的受管会话。不能把“已生成页面”或“本机浏览器可打开”作为交付完成。

## 顺序与工作量

按 Task 1 → 2 → 3 → 4 → 5 → 6 顺序推进。每阶段先用失败用例暴露风险，再实现并运行相应验证；Cloudflare Tunnel 只在 Task 6 的隔离环境中配置，不在本轮规划阶段启动真实远控或修改现有会话。

接入验证按 1–2 个工作日预留；可用版本按约 1–2 周规划，包含 Windows 适配、手机测试和外网部署。此为排期估计，不是承诺；Task 1 结束后根据原生接口可用性重新估算。
