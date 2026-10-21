# LOG

## [2026-10-21 15:40 PDT] mac-dev -> win-build  状态: REQUEST
- 分支/commit: fix/782-extensions-mkdir-startup（见该分支最新 commit）
- 背景: issue graphif/project-graph#782。首次安装时 `%APPDATA%\<app>` 不存在，`main.tsx` 中 `Promise.all` 并发跑 7 个 init，
  `ExtensionManager` 用非递归 `mkdir(appData/extensions)` → ENOENT → 整个启动链 reject → `renderApp()` 不执行 → 5 秒后窗口被强制 show，透明空白。
- 改动:
  1. `app/src/core/extension/ExtensionManager.tsx`: `mkdir(extensionsDir, { recursive: true })`
  2. `app/src/core/extension/Extension.tsx`: 安装扩展时 `mkdir(base, { recursive: true })`
  3. `app/src/main.tsx`: `ExtensionManager.init()` 失败只记录日志，不再阻断启动
- Mac 上 `pnpm --filter @graphif/project-graph type-check` 已通过。
- 请求 win-build 做:
  1. 拉取分支，`pnpm install`，按项目文档编译（`pnpm dev:app` 或 tauri build 均可）。
  2. **复现旧 bug**（先在 master 上验证）：退出程序 → 结束所有进程 → 把应用数据目录
     （`%APPDATA%\liren.project-graph`，以 tauri.conf 的 identifier 为准）**整个改名备份** → 启动。预期 master 出现透明空白窗口。
  3. 同样操作在本分支上启动，预期正常显示主界面；且该目录及 `extensions` 子目录被创建。
  4. 再次启动（目录已存在）确认正常。
  5. 结果（含 DevTools Console 报错，如有）写回本 LOG，状态 RESULT。
