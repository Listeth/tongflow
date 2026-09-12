# Safa 自部署快速上手（中文）

本分支包含在 macOS/Linux 本机把 TongFlow 完整跑起来所需的**非显然**步骤，全部在 2026-09 实测。
上游 README 的命令是对的，但会踩以下三个坑：

## 坑 1：pnpm 12 的构建脚本白名单

`pnpm install` 会拒绝执行 better-sqlite3 / sharp / esbuild 等的 postinstall 并报错。
修复：在 `pnpm-workspace.yaml` 增加（本分支已带）：

```yaml
allowBuilds:
  '@parcel/watcher': true
  '@swc/core': true
  better-sqlite3: true
  esbuild: true
  protobufjs: true
  sharp: true
```

注意：pnpm 10 的文档会让你写 `onlyBuiltDependencies`，**pnpm 12 已换成 `allowBuilds` 映射**，写错不生效。

## 坑 2：corepack shim

没有全局 pnpm 时：

```bash
mkdir -p ~/.localbin
corepack enable --install-directory ~/.localbin pnpm
export PATH="$HOME/.localbin:$PATH"
pnpm install && pnpm dev   # http://localhost:3000
```

`pnpm dev` 的 predev 会在内部调用 `pnpm`，所以必须让 pnpm 在 PATH 里，仅 `corepack pnpm dev` 不够。

## 坑 3：插件凭据必须是纯 ASCII

插件进程用 `Authorization: Bearer <key>` 发请求，Node 的 HTTP 头只允许 latin-1。
密钥里出现任何非 ASCII 字符（包括看起来无害的省略号）都会导致请求以
`'latin-1' codec can't encode character` 失败，且 UI 不显示明确错误。

## 运行环境要求

- Node ≥ 20（实测 v24）、Python ≥ 3.10 在 PATH（插件子进程用，自动建托管 venv）
- 无 Docker 也能源码跑；SQLite 落在 `data/tongflow.db`
- 无登录墙；`/workspace` 直接可画

## 插件开发最小路径

1. `plugins/<name>/entry.py`（目录名小写，前缀 `tongflow-api-|tongflow-modal-|tongflow-router-`）
2. 函数加 `@node_slot(NodeSlots.X)` 装饰器 + 生成类型注解 —— 无注册清单，扫描靠 AST
3. 模型下拉 = `TONGFLOW_SLOT_MODELS` 纯字面量；节点高级参数 = `TONGFLOW_SLOT_PARAMS`
4. 凭据 = 应用内 Settings 写 env（存 `/data/settings.json`），或项目 `.env`
5. 放入 `plugins/` 刷新即生效，首跑自动建 venv 装 SDK

参考实现（本 fork 姊妹仓库）：`Listeth/tongflow-router-safa`
