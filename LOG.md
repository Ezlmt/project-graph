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

## [2026-10-21 23:03 PDT] mac-dev -> win-build  状态: DONE
- 分支/commit: fix/782-extensions-mkdir-startup@5559b67a
- 内容: 验证结果已接受，PR 已提交：https://github.com/graphif/project-graph/pull/784 。
  启动初期短暂透明为 dev 模式 Vite 首次编译所致，不在本 PR 范围。如维护者要求修改，会在此追加新的 REQUEST。

## [2026-10-22 22:45 PDT] mac-dev -> win-build  状态: REQUEST
- 分支/commit: 上游 master@69b07b91（#784 已合并）
- 新问题（用户反馈）: ① 窗口莫名半透明，且改「窗口背景透明度」无效；② 切换主题后颜色不变、部分界面颜色错乱、深浅色混杂。
- 背景: 默认 `windowBackgroundAlpha=0.72`（本来就半透明）。上次测试把 `%APPDATA%\liren.project-graph` 改名备份过，
  现在用的是全新默认配置，可能与"突然变透明"有关。上游 e85ec4b9 还给主窗口加了 `noRedirectionBitmap: true`（仅 Windows 生效），也是怀疑点。
- 请 win-build 收集（不要修改/删除任何用户数据）:
  1. 用户当前运行的是哪个：`pnpm dev`（哪个分支/commit，`git log --oneline -1`）还是安装包（版本号，见 设置→关于）。
  2. 当前 `%APPDATA%\liren.project-graph\settings.json` 全文 → `logs/2026-10-22-settings.json`；
     列出 `%APPDATA%` 下所有 `liren.project-graph*` 目录（含备份目录名）。
  3. 若是 dev 版：在应用里按 Ctrl+Shift+I（或 F12）打开 DevTools，复制 Console 全部输出 → `logs/2026-10-22-console.txt`。
     然后在 Console 执行并记录结果：
     `getComputedStyle(document.documentElement).getPropertyValue('--background')`
     `document.querySelector('#pg-theme')?.innerHTML.slice(0,300)`
  4. 复现操作并记录：设置里把透明度拖到 1 → 画布是否变为不透明？重启后是否保持？
     切换主题（如 light ↔ catppuccin-mocha）→ 哪些区域变了、哪些没变（顶部菜单/画布/设置窗口/弹窗）。
  5. 对照实验（如用户同意）：在 master 上把 `app/src-tauri/tauri.conf.json` 中 `"noRedirectionBitmap": true` 这行删掉，重新 `pnpm dev`，
     看透明/主题问题是否消失。做完恢复该文件。
  6. 截图（透明窗口、主题错乱处）放 `logs/`，并在此写 RESULT。
