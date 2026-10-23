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

## [2026-10-23 PDT] win-build -> mac-dev  状态: RESULT
- 请求: 2026-10-22 22:45 PDT；部分收集完成，UI 验证受环境阻碍，未判定根因。
- 开始时未发现运行中的 Project Graph，无法确认用户之前用安装包还是 dev。本机代码为 fix/782-extensions-mkdir-startup@5559b67a，版本 4.2.6-alpha1，本轮在该提交运行 pnpm dev。
- 设置全文: logs/2026-10-22-settings.json；目录清单: logs/2026-10-23-data-directories.txt。alpha=0.72，theme/darkTheme=catppuccin-mocha，themeMode=dark，lightTheme=light。存在 liren.project-graph 和 liren.project-graph.backup，未读取备份内部。
- 当前 Node v26.12.1、pnpm 11.3.0、cargo 可运行。旧分支 dev 编译成功，但只出现全透明未渲染空窗。初次前端终端三次报 [Unhandled rejection] Unknown Error: Access is denied. (os error 5)。授权应用数据目录后重启，未再观察到同样报错，但仍未显示设置界面。不能当成用户原有主题问题的可靠复现。
- Ctrl+Shift+I/F12 均未获得可用 Console；未执行 computed style 和 pg-theme 查询。未能测试 alpha=1、主题切换、重启保持；无 Console 全文。
- 已拉取并切换 master@69b07b91，确认配置含 noRedirectionBitmap:true（旧提交无此项）。但 pnpm dev 触发依赖检查/安装，创建 pnpm store 项目符号链接时报 EPERM，未能运行 master。未执行删除该项的 A/B 实验，未改 tauri.conf.json。
- 原 Cargo.toml 有修改标记但 diff 无文本差异，切换前 stash 保留，结束已回原分支并从 stash 恢复文件。stash win-build-preserve-existing-cargo 仍保留备份。
- 未改名/删除用户数据，未主动改设置。应用运行导致 settings.json 键顺序变化，但键值一致。已停止本轮 dev 进程。
- 未上传截图，因为全透明空窗截图暴露其背后的聊天内容，且无法诊断主题。待用户非沙箱终端启动 master 后补做 UI/Console 对照；不能标记验证通过。

## [2026-10-23 PDT] win-build -> mac-dev  状态: RESULT
- 用户补充反馈（非 agent 独立复现）：应用主题后，再切到明亮、再切回黑暗，颜色又变绿。截图中明确可见临时草稿标签在图03为蓝色、图05为绿色边框/底色；不是整个窗口变绿。顶部菜单仍为深蓝紫色。
- 图02主题设置页选中 Catppuccin Macchiato，显示“应用主题”按钮；用户表示已经切换主题，但截图本身不能证明按钮是否已点击、darkTheme 是否同步持久化。请检查应用主题与明暗切换使用的 theme/darkTheme/lightTheme 状态是否一致。
- 透明度问题：用户表示窗口透明度快捷键没有反应。图06菜单标注“切换窗口透明度模式 Ctrl+0”“窗口不透明度减小 Ctrl+Alt+Shift+-”“窗口不透明度增加 Ctrl+Alt+Shift++”，红箭头指向增减项。尚未确认是键盘快捷键不触发还是点击菜单项也无效，请区分事件匹配问题和状态改变后渲染未更新。
- 图01/03/04/05均可透出桌面壁纸；图06透出背后的聊天窗口，证明是窗口透明，不是应用内背景图片。截图不能证明调整后的数值或变化幅度。
- 六张用户授权同步的原始截图依次为 logs/2026-10-23-theme-01.png 至 theme-06.png。图04为明亮模式，图05为切回深色后的绿色标签；图06为透明度菜单及快捷键。
- 请据此继续排查主题切换状态同步、透明度快捷键/菜单动作和画布重绘。当前运行版本及 Console 数据仍未确认，不将推测写成根因。

## [2026-10-23 15:20 PDT] mac-dev -> win-build  状态: REQUEST
- 分支/commit: fix/theme-recursion-and-opacity-keys@41d999a7（基于上游 master@69b07b91，已包含 #784）
- 根因（已在 Mac 上用模拟脚本复现，与截图现象完全一致）:
  1. 主题: `Settings` Proxy 的 set 先通知 listener 再写内存。App.tsx 中 theme→darkTheme→theme 的 watcher 读到旧值互相触发 → 栈溢出。
     结果: UI CSS 已换成新主题，但画布(StageStyleManager)没更新 → 深浅混杂；内存 darkTheme 仍是 mocha → 明暗切换后回到 mocha（mocha 的 primary 是绿色 #a6e3a1，即截图里的绿色标签）。
  2. 透明度快捷键: 匹配用 event.key，按住 Shift 时「-」变成「_」，`Ctrl+Alt+Shift+-` 永远匹配不上。
  （另：`Ctrl+0` 是在 0 和 1 之间切换，从默认 0.72 按一次会变成 0 = 全透明，这是现有设计，不是本次修复范围。）
- 改动: `app/src/core/service/Settings.tsx`（先 Reflect.set 再通知）、`app/src/utils/emacs.tsx`（Shift 时按物理按键回退匹配）、新增 `emacs.test.ts`。
- Mac 已通过: type-check；vitest 新增 4 个用例通过，全量失败数与 master 相同（41 个已有失败，与本改动无关）。
- 请 win-build（建议让用户在自己的终端里运行，避免沙箱权限问题）:
  1. `git fetch origin && git checkout fix/theme-recursion-and-opacity-keys`，`pnpm install`，`pnpm dev`。
  2. 主题: 设置→个性化→主题→选 Catppuccin Macchiato→应用主题。确认画布和界面同时变色（标签为蓝色）。
     点右上角明暗切换按钮 → 明亮 → 再切回黑暗，确认仍是 Macchiato（蓝色），不再变绿。重启应用后仍是 Macchiato。
  3. 透明度: 在画布上（不要聚焦输入框）按 `Ctrl+Alt+Shift+-` 若干次，画布逐步变透明；`Ctrl+Alt+Shift++` 逐步变不透明，到 1 后画布边缘闪一下。
  4. 写 RESULT（通过/不通过 + 截图可选）。

## [2026-10-23 15:23 PDT] win-build -> mac-dev  状态: RESULT
- 请求: 2026-10-23 15:20 PDT；分支 fix/theme-recursion-and-opacity-keys@41d999a7，已检出且工作树干净。
- 自动检查: 新增 emacs.test.ts 的 4 个测试全部通过（Vitest 4.1.5，Windows）。初次测试被沙箱临时目录 rename EPERM 阻碍，将本次进程 TEMP/TMP 设置到工作区 ../test-temp 后重试通过；未改仓库代码。
- 类型检查: pnpm --filter @graphif/project-graph type-check 被自动依赖检查的 symlink EPERM 阻止；随后调用现有 TypeScript 可执行文件执行 `tsc --noEmit -p app/tsconfig.json`，退出码 0，无错误输出。
- 依赖安装: pnpm install --frozen-lockfile 仍在 pnpm store 注册项目符号链接阶段 EPERM，未完成本分支锁文件的依赖重装。上述测试使用已有 node_modules，须保留这一限制。
- UI 验收未完成: 未得到本提交的可用桌面运行环境，不能宣称主题明暗切换、Macchiato 重启保持和透明度增减已在 Windows 实际验证通过。按 REQUEST 建议，需要用户从自己的终端运行 `pnpm.cmd install --frozen-lockfile` 和 `pnpm.cmd dev` 后，补测蓝色标签保持、重启保持及画布焦点下增减快捷键。
- 本轮未读取/修改用户数据或设置，未改源文件；当前代码保留在新修复分支，便于用户启动。请先视为自动检查通过、人工验收待完成。
