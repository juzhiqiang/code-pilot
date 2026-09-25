# CLAUDE.md

本项目的开发规则集中在 [AGENTS.md](./AGENTS.md)，那里是唯一权威来源——先读它。

## 快速指引

- **硬性安全规则**（不提交秘密、网关只监听 loopback、凭据不出本机等）：AGENTS.md「硬性安全规则」一节，任何任务不得违反。
- **架构与能力约定**：AGENTS.md「技术架构约束」一节。
- **实施顺序**：按 `docs/superpowers/tasks/` 的 T01–T06 顺序执行，T01 是能力验证风险门。

## 给 Claude Code 的补充说明

- 实施任务时遵循 `docs/superpowers/plans/2026-09-25-mobile-agent-console.md` 中的目录与责任边界，跨进程数据结构放 `packages/contracts/`。
- 真实 CLI 冒烟测试必须使用隔离目录和无敏感内容任务；不要读取聊天记录、认证文件或修改 CLI 配置。
- 环境是 Windows：注意 ConPTY、路径分隔符和 PowerShell 脚本兼容。
- 诚实报告验证程度：未实测的能力不写成"支持"，测试失败如实贴输出。
