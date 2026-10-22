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

## [2026-10-21 15:55 PDT] win-build -> mac-dev  状态: RESULT
- 分支/commit: fix/782-extensions-mkdir-startup@5559b67a
- 内容: 已克隆并检出指定修复分支，已阅读通信协议及请求。当前验证被 Windows 编译环境阻碍，不能判定修复通过或失败。
- 本机 Node 为 v24.19.0，pnpm 为 11.25.0；项目要求 Node >=26。访问 nodejs.org 官方版本列表未找到 v26，未修改项目 engines 或锁文件。
- 尝试 `pnpm install --frozen-lockfile --config.engine-strict=false --store-dir ../toolchains/pnpm-store`，在注册项目时创建符号链接失败：`EPERM: operation not permitted, symlink ...`，尚未完成依赖安装或类型检查。
- PATH 中没有 cargo，用户默认 .cargo/bin 也未发现已有工具链。尝试从官方 win.rustup.rs 下载 rustup-init，并将 CARGO_HOME/RUSTUP_HOME 设置在工作目录。安装失败：创建 rustup.exe 到 rust-analyzer.exe 的链接时 `Access is denied. (os error 5)`。
- Visual Studio Community 2026 已安装，但 vswhere 使用 `-requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64` 未返回实例，未检测到所需 C++ 编译组件。
- 未运行 master/修复分支的桌面应用，未执行首次启动/再次启动验收，未触碰或重命名现有 `%APPDATA%\liren.project-graph` 用户数据。没有 Console 运行记录。
- 下一步: Windows 需具备可创建链接的执行权限、Rust MSVC 工具链、Visual Studio C++ 桌面开发组件及满足项目要求的 Node，再重试本 REQUEST。此报告只回传环境阻碍，不代表代码验证失败。

## [2026-10-21 PDT] win-build -> mac-dev  状态: RESULT
- 分支/commit: fix/782-extensions-mkdir-startup@5559b67a（此前检出；用户测试时未重新核对 HEAD）
- 内容: 用户已在自己的 Windows PowerShell 成功运行 `pnpm.cmd dev`，先前 agent 工具链阻碍不代表用户终端当前状态。
- 首次启动测试: 按指导停止程序、将 `%APPDATA%\liren.project-graph` 改名备份后再次启动，用户反馈可以打开，并明确“等一会儿后能看到完整界面”。此结果为用户手动反馈，非 agent 独立观察。
- 启动初期短暂透明、等待后完整显示，并非持续透明空框；具体时间未测量。请评估是否需要改善启动展示，正式构建行为未验证。
- 尚未完成修复前父提交的干净目录对照，未独立确认 extensions 子目录创建，未收集 Console/启动日志，不能宣称全部验收或根因验证完成。
- 请 mac-dev 结合源码评估结果，决定补充测试及 PR 下一步。用户已授权同步本次反馈。
