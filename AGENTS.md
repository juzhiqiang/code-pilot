# AGENTS.md — code-pilot 开发规则

本文件是本项目 AI 协作开发的主要规则文件。`CLAUDE.md` 是面向 Claude Code 的入口，指向本文件；两者不重复维护同一规则，修改规则只改这里。

## 项目是什么

手机远程控制本机 Claude Code / Codex CLI 的多会话控制台：本机网关 + Cloudflare Tunnel + 手机 PWA。设计基线见 `docs/superpowers/specs/2026-09-25-mobile-agent-console-design.md`，实施计划见 `docs/superpowers/plans/2026-09-25-mobile-agent-console.md`，任务分解见 `docs/superpowers/tasks/`。

## 硬性安全规则（任何任务不得违反）

1. **不提交秘密**：CLI 凭据、Cloudflare Tunnel credentials JSON、`.env`、日志中的密钥、真实聊天内容一律不入 Git。
2. **网关只监听 `127.0.0.1`**：外网访问仅经 Cloudflare Tunnel，永不直接暴露本地端口到公网。
3. **CLI 凭据不出本机**：不读取、不上传、不转发 CLI 认证文件；登录命令的输入不采集。
4. **用户输入不拼接 shell**：通过协议字段或 stdin 传递；provider、CLI 路径、参数模板由服务端控制。
5. **不越权杀进程**：停止操作仅针对本工具明确归属的进程树；外部会话用其原生中断协议；PID 不明时禁止批量杀。
6. **不默认绕过权限**：保留 CLI 的审批和沙箱配置，不为远控默认开启 bypass 选项；不以 SYSTEM 身份运行依赖个人 CLI 登录的 Agent。
7. **路径校验**：服务端验证项目真实路径在登记目录内，含 Windows junction/symlink 解析；手机端的选择不可信。

## 技术架构约束

- **进程边界**：`apps/web`（PWA 前端）、`apps/gateway`（Fastify HTTP/WS 网关）、`apps/agent`（常驻会话管理进程）三者独立；网关或手机断线不得终止 agent 的任务。
- **IPC 仅限本机**：gateway ↔ agent 的 IPC 必须仅本机回环且带认证，不暴露给手机。
- **共享合同**：跨进程数据结构定义在 `packages/contracts/`，两侧只 import 合同，不复制结构。
- **能力驱动 UI**：适配器各自返回 capabilities（readHistory / streamEvents / terminal / send / queue / interrupt / approve / resume…），前端按返回值渲染，**不猜测、不假定能力存在**。不满足时显示只读或降级说明，不伪装成可控。
- **事件与命令语义**：事件按 `sessionId + seq` 去重；写入命令按 `commandId` 幂等；重连缺口显示 gap，禁止伪造连续日志、禁止断线重连后重发未确认命令。

## 诚实性规则（对 AI 的核心行为要求）

1. **不夸大验证程度**：没实测过的协议行为不写成"支持"。设计文档中"尚未测试"的项在验证前保持"未验证"状态。
2. **先探测再承诺**：任何 CLI 原生能力（attach、订阅、授权回传等）必须先在隔离测试中验证，通过后才在 UI 开放对应操作。
3. **不把设计当实现**：复选框只在实际执行并验证后勾选；"代码写完"不等于"功能可用"。
4. **如实报告失败**：测试失败就贴输出，跳过就说明跳过，不掩盖、不重述为成功。
5. **不自动重跑**：进程崩溃或结果未确认时标注"结果待确认"，先查询上游，不自动重发指令。

## 开发流程约定

- **按任务顺序实施**：T01（能力验证）→ T02 → T03 → T04 → T05 → T06。T01 是风险门：能力不可用时先在 `docs/compatibility.md` 记录降级路径再继续。
- **任务文件即真相**：每完成一个任务文件的复选框，按其实际验证命令执行并记录结果；任务间的进度记录以这些文件为准。
- **测试隔离**：真实 CLI 冒烟测试只用隔离目录和无敏感内容任务；单元测试不得消耗模型额度或修改用户项目。Codex WebSocket 只在本机或隔离测试中运行。
- **代码风格**：TypeScript strict；新模块遵循现有目录结构（见实施计划的目录边界）；注释密度与现有代码一致。
- **提交**：提交信息说明动机而非罗列文件；不绕过 hooks；不提交未运行的代码。

## 文档规则

- 设计决策的变更先改 `docs/superpowers/specs/` 的设计文档，再改代码。
- `docs/superpowers/` 下的文档是 AI 辅助生成的设计产物（README 已注明），内容与代码冲突时以代码为准，并回补文档。
- 适配器实测结果（版本、能力矩阵、退路）记录在 `docs/compatibility.md`。

## 已知的技术前提（不要遗忘）

- 环境为 **Windows**，注意 ConPTY、路径分隔符、PowerShell 脚本、中文目录名兼容。
- Codex App Server 的 WebSocket 传输官方标注 experimental，过载返回 `-32001`，需指数退避 + jitter。
- Claude Code `-p` 与 `--bg` 不能组合；`--resume` 按会话继续；SDK `interrupt()` 与杀进程语义不同。
- 初始并发上限 5 个执行会话；日志配额 20 MiB/会话、1 GiB 总量、7 天，先到者生效。
