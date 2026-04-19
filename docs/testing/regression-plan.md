# Regression Plan

Last updated: 2026-04-19
Status: active

这份清单用于回归 `codex-mobile-terminal` 的核心行为，重点覆盖登录安全、会话管理、WebSocket 通道和手机端消息流。

## 已落地的可执行测试

- `tests/koa-app.test.js`
  - 登录成功/失败
  - 登录限流封禁与窗口恢复
  - 认证会话续期（TTL 刷新）
  - 未登录访问受保护接口
  - 跨域登录拦截
  - session CRUD 路由代理
  - 复用同 `provider + resumeSessionId` 的 live session
  - 健康检查公开访问
  - 重启保护（`shuttingDown`）对非健康接口返回 503
  - 文件浏览接口
  - 非法路径与非法 JSON 返回 400
  - 历史会话相关路由代理
  - history 资源风格路由（`/api/history-sessions/:provider/:resume/messages`）
- `tests/ws.test.js`
  - 未授权 ws 握手拒绝
  - 已授权 ws 握手成功
  - 未信任 Origin 拒绝
  - session 不存在时拒绝
  - input / resize 消息转发
  - 非法帧 payload 返回 `error` 消息
- `tests/session-manager.test.js`
  - provider 目录暴露
  - 非法 provider 创建失败
  - Codex live session 创建
  - `findRunningLiveSessionByResume` 选择最新运行会话
  - app-server 回包写入 buffer
  - 自动命名
  - 手动重命名持久化
  - 历史会话扫描 / 解析 / 归档 / 恢复 / 删除
  - 缺失历史会话时报错路径
  - `listAll` 对 live/history 同 resume 去重
  - provider 命令构建
  - Claude auto-named 场景不注入 `--name`

## 建议测试入口

- 全量后端回归（推荐）
  - `npm run test:backend:all`
- 路由 + ws 快速回归
  - `npm run test:backend:http`
- `SessionManager` 子域回归
  - `npm run test:backend:session`

## 必测回归点

- 认证与安全
  - Cookie 续期
  - 登录限流窗口与封禁恢复
  - `trustedCidrs` / `tailscaleOnly` 行为
  - `Origin` 白名单
- 运行时 HTTP 能力
  - 请求体大小限制
  - 非法 JSON 处理
  - 静态资源查找顺序
  - 缺失前端构建时的首页兜底行为
- Live Session
  - `findRunningLiveSessionByResume`
  - `json_exec` 队列消费
  - app-server 通知流增量消息
  - shell / pty exit 后状态刷新
  - ws snapshot 内容完整性
- Historical Session
  - 图片消息恢复
  - 标题候选与摘要候选提取
  - Codex `session_index.jsonl` 缓存命中/失效
  - 临时目录与非 CLI 会话过滤规则
## 前端回归检查点

每次修改 `web/src/` 后需要人工验证（前端无自动化测试）：

- **消息气泡样式**
  - 用户消息显示在右侧，背景为 `--user-bg`（绿色），文字为 `--user-text`（深色），可读
  - Codex 消息显示在左侧时间线结构中
- **历史消息分页**
  - 打开消息数 > 25 条的会话，初始仅显示最新 25 条
  - 上滑到顶触发分批加载，每次追加 25 条，滚动位置不跳变
  - 新消息流入时（底部锁定状态）最新消息可见
- **流式输出完整性**
  - 发送问题后 Codex 回复不带上一条内容（snapshot 不重复追加）
  - 流式输出内容完整，无需退出重进
  - 含 markdown blockquote（`> 文字`）的回复正常显示，不被噪音过滤器误删
- **构建产物同步**
  - 每次源码改动后在本机执行 `npm run build`，确保 `web/dist/` 为最新版本

## 建议执行顺序

1. 改后端逻辑时先跑对应专项测试。
2. 改路由、安全或 WebSocket 逻辑后补跑 `npm run test:backend:http`。
3. 改会话管理后补跑 `npm run test:backend:session`。
4. 准备提交前统一跑 `npm run test:backend:all`。
