# agent-sync（仅供 Mac 开发 agent ↔ Windows 编译 agent 通信，不进入 PR）

这是 `Ezlmt/project-graph` fork 上的**孤立分支** `agent-sync`，与代码历史无关，永远不要合并或向上游推送。

## 角色
- **mac-dev**（Mac 上的 agent）：负责写代码、推送到功能分支、开 PR。
- **win-build**（Windows 上的 agent）：负责拉取功能分支、编译、在 Windows 上复现/验证，回报结果。

## 协议
1. 所有消息追加到 `LOG.md` 末尾（只追加，不改历史）。格式：
   ```
   ## [YYYY-MM-DD HH:MM TZ] <mac-dev|win-build> -> <对方>  状态: REQUEST|RESULT|QUESTION|DONE
   - 分支/commit: <branch>@<short-sha>
   - 内容: ...
   ```
2. 大段日志放到 `logs/<日期>-<主题>.txt`，在 LOG.md 中引用路径。
3. 每次写完：`git add -A && git commit -m "<role>: <摘要>" && git pull --rebase && git push`。
4. 读取前先 `git pull --rebase`。

## Windows 端初始化
```powershell
git clone https://github.com/Ezlmt/project-graph.git
cd project-graph
git fetch origin fix/782-extensions-mkdir-startup agent-sync
git worktree add ..\project-graph-agent-sync agent-sync   # 通信目录
git checkout fix/782-extensions-mkdir-startup             # 代码目录
```
