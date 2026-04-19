# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.1] - 2026-04-18

### Bug 修复：Codex 不回复 / 回复异常

排查并修复了 `CODEX_APP_SERVER_ENABLED=true`（app_server 模式）下的四个关键问题。

#### P1: 消息重复发送（每条回复 token 发两遍）

**根因**：`appServerBridge.js` 的 `handleMessage()` 中，同一条 notification 被触发了两次 `emit("notification")` — 有 `method` 的消息先在第 114 行触发一次，经过中间的 request/response 分支后，又在第 134 行触发一次。

**修复**：去掉第一个重复的 `emit("notification")`，只保留末尾逻辑分支中的那一次。同时 `codexRunner.js` 的 `handleAppServerTurnResult()` 增加 `turnHadVisibleOutput` 守卫——当 streaming delta 已经通过异步通知广播过时，跳过同步返回结果中的重复广播。

**影响文件**：`src/appServerBridge.js`、`src/sessions/runners/codexRunner.js`

#### P2: app-server 进程崩溃后永久卡死，无法重启

**根因**：`startProcess()` 的进程复用守卫 `if (this.proc && !this.proc.killed) { return; }` 在进程自然退出后仍然通过——因为 Node.js 的 `ChildProcess.killed` 属性只在 `proc.kill()` 主动杀死时为 `true`，自然退出为 `false`。导致旧进程退出后，`this.proc` 仍指向已死进程对象，永远不会重新 spawn。

**修复**：在 `proc.on("exit")` handler 中增加 `this.proc = null`，确保下次调用 `startProcess()` 时能正确重新启动。

**影响文件**：`src/appServerBridge.js`

#### P3: 首次发送消息无反馈（冷启动 25-30 秒黑屏）

**根因**：`codex app-server` 采用懒启动——第一次用户发消息时才 spawn 进程。完整链路 `spawn → wait(350ms) → WS connect → initialize → thread/start → turn/start` 在冷启动时需要 15-30 秒，期间客户端只看到 "等待 Codex 回复…" 没有任何进度提示。

**修复**：在 `startAppServerTurn()` 中检测 `appServerBridge` 是否已完成初始化。首次启动时先广播一条系统消息 "正在启动 Codex 引擎，首次启动约需 15-30 秒…"，让用户知道系统在工作。

**影响文件**：`src/sessions/runners/codexRunner.js`

#### P4: 前端过滤误杀 streaming delta 回复

**根因**：三层过滤链对增量 token 过于激进：

1. `appendNormalizedParts()` 对 streaming delta 调用 `sanitizeAssistantText()`，会过滤掉短 token（如 `"=="`、`"25"`、`" DEC"`）
2. `appendAssistantChunk()` 中的 `isDisposableAssistantFragment()` 会丢弃 `===`、`---` 等片段，即使 stream 已经积累了大量内容
3. `echo suppression` 在 buffer 任意长度时都会触发，可能误删以用户消息开头的合法回复

**修复**：

- streaming delta 跳过 `sanitizeAssistantText()` 过滤，只做 `normalizeLine()` — 噪声过滤应在完整消息上做，不是每个增量 token 上做
- 已有 streaming 消息在显示时，跳过 `isDisposableAssistantFragment()` 检查
- echo suppression 只在 `activeStreamBuffer.length < 200` 时触发，避免长 stream 中的误判

**影响文件**：`web/src/App.vue`

#### 排查方法

通过模拟 WebSocket 连接直接发送消息并 dump 完整消息流，发现：
- 每条 streaming delta 确实被发送了两次（确认 P1）
- 冷启动后 app-server 能正常启动，端口监听正常，但 `proc` 重启逻辑有 bug（确认 P2）
- 首次消息到回复约 15-30 秒，期间无任何消息到达客户端（确认 P3）
- 回复内容包含 `"=="`、`"OCT"` 等短 token，会被前端过滤吞掉（确认 P4）

---

## [0.2.0] - 2026-04-18

### V2 手机端 UI 重构

基于 `/UI_PC/v2/` 设计原型，对手机端界面进行全面重构。脱离微信气泡美学，采用冷色调技术风（teal 主色 `#0D8F7C`），重新设计为事件时间轴 + 类型化事件卡。

#### 会话列表 (SessionListView)

- **按状态分组**：「需要你」→「运行中」→「安静」，替代旧的 workspace 分组
- **全局状态条**：列表顶部显示连接状态、会话统计（待审/运行数量）
- **会话行重构**：
  - 左侧 2px 状态竖条（spine），颜色按状态编码（amber/teal/slate）
  - RepoMark 方块头像：每个 workspace 独立配色（8 色调色板 + 白字），替代旧的单色 teal
  - 顶部行：repo mark + 会话名 + 时间
  - 分支行：mono 字体显示 workspace 名称
  - 底部行：EventBadge + 事件预览文本（`margin-left: 42px`）
  - 运行中会话显示动画进度条
- **低信噪比过滤**：自动隐藏标题和预览都无意义的历史会话（如系统指令、"恢复这个会话"等）
- **EventBadge tag 样式**：彩色文字 + soft background（`border-radius: 3px`），替代旧的 solid pill

#### 聊天详情 (ChatView)

- **事件时间轴布局**：
  - 左侧单条 absolute spine 竖线（`left: 27px`），替代旧的 `::before` 伪元素
  - 彩色时间轴节点：14px 空心圆 + 8px 彩色内点，颜色按事件类型
  - 每个事件：meta 行（badge + time）→ 内容区域（垂直排列）
- **类型化事件卡**：
  - **用户指令**：深色实心块（`#0A0C10`），白色文字，`border-radius: 8px`
  - **思考**：纯灰色文本，无卡片包裹
  - **执行（ActionCard）**：深色代码块，`$ command` 头部 + 分隔线 + 输出 + `→ result` 前缀
  - **回复**：白色卡片 + markdown 渲染
- **AppBarV2 头部**：
  - meta 行（mono uppercase workspace 名称）+ 大字标题
  - 状态胶囊（带 dot-pulse 动画）+ 更多按钮
- **Slash command 条**：`/继续` `/回滚` `/解释这段` `/跑测试`，点击插入文本
- **Composer 输入区**：
  - 输入框无边框，`background: var(--chat-bg)`
  - teal 方形发送按钮（38x38, `border-radius: 8px`），SVG 发送图标
  - amber 色中断按钮
- **视口感知渲染**：仅渲染可见范围 ±8 条消息，超出范围显示 placeholder
- **渲染缓存**：`renderedHtmlCache` Map 缓存 markdown 渲染结果

#### CSS 设计系统

- **V2 设计 Token**（`public/styles.css`）：
  - 主色 teal `#0D8F7C`，deep `#0A7264`
  - 事件色：user `#0A0C10`、thought `#8B95A3`、action `#0D8F7C`、diff `#5B6CFF`、approval `#C97A14`、error `#C24D4D`
  - 状态色：running `#C97A14`、success `#0D8F7C`、error `#C24D4D`、idle `#8B95A3`
  - 代码块背景 `#0E1116`
- **清理**：移除旧的 V1 微信气泡样式（`.message-stream`、`.message-item`、`.message-bubble`、`.message-text`、`.composer` 等）
- **移除旧 session 列表全局样式**，避免与 scoped 样式冲突

#### 性能优化

- **highlight.js 精简**：从全量 180+ 语言（~700KB）减至 12 个常用语言，最大 chunk 从 ~700KB 降至 122KB
- **代码分包**：Vite manualChunks 分离 vendor-vue、vendor-markdown、vendor-highlight，总包 1.1MB → 340KB
- **Vite 8 rolldown 适配**：`manualChunks` 必须使用函数形式

#### 未实现（计划后续）

- ❌ 双指按压审批 — 用普通按钮替代
- ❌ Diff 全屏查看器 — 独立页面，后续再做
- ❌ TabBar 底部导航 — 当前只有两个视图，不需要
- ❌ RTT 延迟显示 — 后端无此 API
- ❌ Context chips（文件/文件夹附加） — 前端无此功能

---

## [0.1.0] - 2026-04-13

### 初始版本

- 浏览器端 Codex 终端，支持手机访问
- 多会话管理（创建、切换、恢复）
- WebSocket 实时通信
- Token 认证
- Tailscale 外网访问支持
- PM2 部署支持
