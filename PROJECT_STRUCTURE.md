# Project Structure

> codex-mobile-terminal v0.2.1
> Last updated: 2026-04-19

面向公开仓库的目录说明，帮助贡献者快速理解主要模块和发布边界。

## 顶层目录

```text
codex-mobile-terminal/
├── src/                  Node.js + Koa 后端
├── web/                  Vue 3 + Vite 前端
├── tests/                后端测试
├── scripts/              开发、PM2、launchd 脚本
├── docs/                 公开文档与截图
├── public/               兼容静态资源
├── package.json          npm 脚本与依赖
├── ecosystem.config.cjs  PM2 配置
└── .env.example          环境变量模板
```

## 后端

```text
src/
├── server.js             服务入口
├── koa-app.js            Koa 应用组装
├── config.js             环境变量解析
├── sessionManager.js     会话生命周期管理
├── appServerBridge.js    codex app-server 桥接
├── routes/               HTTP 与 WebSocket 路由
├── runtime/              运行时 HTTP、安全、资源服务
└── sessions/             live/history/provider 逻辑
```

关键链路：

```text
浏览器 -> /api/* 与 /ws
      -> sessionManager
      -> node-pty / codex app-server
```

## 前端

```text
web/
├── index.html
├── vite.config.js
└── src/
    ├── main.js
    ├── router.js
    ├── App.vue
    ├── components/
    ├── lib/
    └── styles/
```

- `components/` 放页面和视图组件
- `lib/` 放 API 与消息规范化工具
- `styles/` 放全局样式变量

## 测试与脚本

```text
tests/
├── koa-app.test.js
├── ws.test.js
├── session-manager.test.js
└── helpers/
```

```text
scripts/
├── setup.mjs
├── service.mjs
├── dev-up.sh
├── dev-down.sh
└── launchd/
```

## 非仓库内容

以下内容属于本地运行产物或私人配置，不应进入 Git：

- `.env`
- `node_modules/`
- `web/dist/`
- `logs/`
- `data/`
- 系统生成文件与临时测试文件
