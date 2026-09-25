# T05 Mobile PWA

**目标：** 构建手机优先的 PWA，支持设备配对、多任务监控、任务详情、新建任务、输入控制和断线恢复。

**依赖：** T04。  
**产出：** `apps/web/src/*`、移动端 E2E 测试、PWA 静态资源。

## 步骤

- [ ] 初始化 Vite React TypeScript 应用和移动端 CSS；配置 PWA 外壳只缓存静态资源，不缓存认证接口、日志和输入草稿。
- [ ] 实现 `PairDevice.tsx`：输入一次性配对码，显示 HTTPS/WSS 连接状态，支持设备撤销反馈。
- [ ] 实现 `SessionList.tsx`：按 running、waiting、idle、failed、ended 筛选；卡片展示 provider、项目、轮次状态、当前活动、最后活动时间和可控性。
- [ ] 实现 `CreateSession.tsx`：选择 provider、登记项目、模型/权限配置和任务文本；不开放任意 shell 可执行路径；重复点击复用 commandId。
- [ ] 实现 `SessionDetail.tsx`：概览、活动时间线、日志分页、任务清单、文件变更、授权提示、输入和中断/结束控制。
- [ ] 实现 `Terminal.tsx`：仅 capability 含 terminal 时挂载 xterm；支持 Enter、Esc、Ctrl+C、resize；卸载页面不结束服务器任务。
- [ ] 实现 `data/events.ts`：保存每个 session 的最后 seq，处理重连、重复事件、乱序和 gap；gap 必须显示给用户。
- [ ] 实现离线状态：断线时禁用发送，重连后先补读历史再恢复 WebSocket，禁止自动重发过期输入。
- [ ] 用 Playwright 覆盖配对、任务列表、两个任务并行、创建、追加输入、授权拒绝、中断、刷新、断线重连和重复点击。
- [ ] 在窄屏 Android Chrome/iOS Safari 检查中文输入法、软键盘、安全区、滚动和发送按钮可见性。

## 阶段出口

手机可完成“配对 → 查看多个任务 → 创建任务 → 查看实时输出 → 追加输入 → 断线恢复”的完整流程；只读会话不能显示写入按钮。

## 验证命令

```powershell
npm run test:e2e
npm run build --workspace apps/web
```

预期：E2E 通过，生产构建成功，移动端无水平溢出和软键盘遮挡关键操作。
