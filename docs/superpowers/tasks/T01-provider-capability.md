# T01 Provider Capability Verification

**目标：** 在不影响用户已有会话的前提下，验证 Codex App Server、Claude Headless 和 Windows PTY 的真实能力，决定后续适配器使用原生协议还是降级路径。

**依赖：** 无。  
**产出：** `docs/compatibility.md`、`tests/fixtures/codex/`、`tests/fixtures/claude/`、脱敏协议样本。

## 步骤

- [ ] 记录 `node --version`、`codex --version`、`claude --version`、Windows 版本和目标工作目录；只记录版本和路径，不读取认证文件。
- [ ] 在隔离输出目录执行 `codex app-server generate-ts --out tests/fixtures/codex/schema-ts` 和 `codex app-server generate-json-schema --out tests/fixtures/codex/schema-json`，检查生成文件能被 TypeScript/JSON 解析。
- [ ] 创建一个不含敏感内容的测试项目，启动独立 Codex App Server；验证 initialize、thread、turn、事件通知、输入、审批、中断和结束消息的实际字段。
- [ ] 单独验证已有 Codex daemon 的连接与 `codex queue --thread <THREAD> --message <TEXT>`；记录“已入队”和“已开始执行”分别由哪个事件确认，不能为了探测重启用户 daemon。
- [ ] 验证 Codex WebSocket 只在本机或隔离测试中运行；远程配置使用 TLS 和 capability token 或 signed bearer token，模拟 `-32001` 后记录指数退避和 jitter 行为。
- [ ] 在隔离 Claude 项目执行 `claude -p "输出固定短句" --output-format stream-json`，保存脱敏事件样本；验证退出码、最终结果、工具事件和错误事件。
- [ ] 使用 `--resume <session-id>` 继续 Claude 测试会话，验证追加输入；验证 SDK `interrupt()` 或等价中断路径。不要把 `--bg` 与 `-p` 组合。
- [ ] 执行 `claude agents --json`、`claude logs <id>` 和 `claude attach <id>` 的测试会话验证；明确区分可发现、可读取和可写入，不推断任意交互会话都能接管。
- [ ] 安装 `node-pty` 后验证 Windows 中文目录、中文多行输入、resize、Ctrl+C、退出码和子进程树回收；测试结束后只结束自己创建的进程。
- [ ] 将每项能力填写到 `docs/compatibility.md`：`readHistory`、`streamEvents`、`send`、`queue`、`approve`、`interrupt`、`resume`、`terminal`，并注明 CLI 版本和降级路径。

## 阶段出口

两个 CLI 至少各完成一次隔离创建、输出观察、追加输入和中断；已有会话明确标记为可控、只读、可恢复或不可用。任何未验证的能力必须标记为 unknown。

## 验证命令

```powershell
node --version
codex --version
claude --version
codex app-server generate-json-schema --out tests/fixtures/codex/schema-json
```

预期：版本命令退出码为 0，schema 目录存在且包含生成文件；真实冒烟测试的输出只包含测试项目内容。
