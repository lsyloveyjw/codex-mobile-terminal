# Contributing

感谢贡献 `codex-mobile-terminal`。

## 本地开发

1. 安装 Node.js 22+ 和 `codex` CLI
2. 复制 `.env.example` 为 `.env`
3. 设置安全的 `ACCESS_TOKEN`
4. 安装依赖并启动开发环境

```bash
npm install
npm run dev
```

## 提交前检查

至少执行：

```bash
npm run check
```

如涉及后端逻辑或会话管理，请补跑：

```bash
npm run test:backend:all
```

## 变更约束

- 保持改动聚焦，不顺手重写无关模块
- 不要提交 `.env`、日志、运行数据、截图垃圾文件或本地绝对路径
- `web/dist/` 为构建产物，不进入 Git 历史
- 如果改了用户可见行为，请同步更新 `README.md` 或 `docs/`

## Pull Request 说明

- 标题直接说明结果，不写过程流水账
- 描述里说明改了什么、为什么改、如何验证
- 如果改动影响手机端体验，附一张截图更好
