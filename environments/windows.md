# 机器环境检查：Windows

> 本仓库跨 OS 开发（主 macOS / Windows），机器环境检查按 OS 拆分为独立文件。
> 本文件是被 [`TYPESAFE_BALATRO_FEASIBILITY.md`](../TYPESAFE_BALATRO_FEASIBILITY.md) §1 引用的 **Windows 检查清单**。
> ✅ 状态：**已实测（2026-09-22，静态检查）**——Steam / Balatro / Lovely / Steamodded / balatrobot mod 全部就位，且 2026-09-21 23:24 的 Lovely 日志证实 mod 成功加载、HTTP 服务在 `127.0.0.1:12346` 监听。**实连（`uvx balatrobot serve` + `health`）尚未跑**（首次验收清单第 2–4 项待办）。
> 维护约定：换机器 / 重装环境后照"如何复检"跑一遍并更新"检查快照"与依赖表；**不要反向修改主文档的结论**。

## 检查快照

| 项 | 值 |
| --- | --- |
| 检查时间 | 2026-09-22（静态；游戏 / mod 加载证据取自 2026-09-21 23:24 的 Lovely 日志） |
| OS / 硬件 | Windows 11 Pro，10.0.26200 x64；13th Gen Intel i7-13700KF；C 盘可用 832 GB |
| 状态 | ✅ 环境就位（游戏 + 全部 mod）；⬜ 实连未跑 |
| 复检命令 | 见文末"如何复检"（PowerShell） |

## 路径体系（本机实测）

主文档（评估机 macOS）引用 `/Users/huangqingming/Workspace/...`；本 Windows 机的对应位置在 `C:\Users\hqm\Documents\github\`：

| 主文档路径（macOS） | 本机路径（Windows） |
| --- | --- |
| `/Users/huangqingming/Workspace/balatrollm` | `C:\Users\hqm\Documents\github\balatrollm`（有 `uv.lock`，无 venv） |
| `/Users/huangqingming/Workspace/typesafe-mario` | `C:\Users\hqm\Documents\github\typesafe-mario`（无 `uv.lock` / venv） |
| `/Users/huangqingming/Workspace/typesafe-balatro` | `C:\Users\hqm\Documents\github\typesafe-balatro`（本仓库） |

另：`uv` 装在 `C:\Users\hqm\scoop\shims\uv.exe`（Scoop 管理）。

## 依赖检查（2026-09-22 实测）

| 依赖 | 实测结果 | 状态 |
| --- | --- | --- |
| Steam | `C:\Program Files (x86)\Steam\`（默认库目录；`libraryfolders.vdf` 只有一个库，无第二盘） | ✅ |
| Balatro 本体 | `...\steamapps\common\Balatro\Balatro.exe`（LÖVE 11.5；appmanifest `buildid = 17459173`） | ✅ |
| Lovely（Windows 版） | 与 `Balatro.exe` 同目录的 `version.dll`（= 默认 `lovely_path`）；Lovely **0.9.0**，注入实测成功 | ✅ |
| Balatro Mods 目录 | `%APPDATA%\Balatro\Mods\`，内含 `balatrobot/`、`lovely/`、`smods/` | ✅ |
| balatrobot mod | 源码为 **v1.5.2**（含 #193 并发修复 `server.lua`）；`balatrobot.json` 标 `1.5.1` 是上游 manifest 滞后（v1.5.2 tag 即如此，main 已改为 1.5.2）。加载日志：21 个 endpoint、`HTTP server listening on http://127.0.0.1:12346` | ✅ |
| `balatrobot` CLI | **未持久安装**（`uv tool list` 空、PATH 无、`~\.local\bin` 无）；`uvx balatrobot` 可用（PyPI 最新 1.5.2，满足 >=1.5.2）。注意 CLI 无 `--version` 选项，用 `--help` 验证 | ⚠️ 仅 uvx |
| `typesafe-sdk` | 任何 venv / uv tool 均未安装；PyPI 最新 **0.7.1**（评估文档基于 0.7.0，patch 级差异） | ⬜ 缺 |
| `TYPESAFE_API_KEY` | 进程 / 用户 / 机器三级均未设置 | ⬜ 缺 |
| Python | 系统 3.14.5（`py` 只注册 3.14）；uv 托管 3.12.9 + **3.13.12**；`uv python find 3.13` ✅、`py -3.13` ❌ → 统一走 `uv` | ✅（走 uv） |
| 参照仓库 | `balatrollm` / `typesafe-mario` 均已 clone；均无 venv；`balatrollm/pyproject.toml` 仍 pin `balatrobot>=1.4.1`（主文档 §1.1 已警告需升到 >=1.5.2） | ⚠️ 待 `uv sync` |

**已知噪音（无害）**：`Mods\balatrobot\` 里带 `.claude/settings.json` 与 `.mux/mcp.jsonc`，Steamodded 递归扫描元数据时会各打一条 `Found invalid metadata JSON file ... ignoring`（删掉这两个目录即可消除）。

## Windows 侧的启动路径（已对照上游 `balatrobot/platforms/windows.py` 核实，本机实测一致）

- `WindowsLauncher` 存在，内置默认路径：
  - `love_path` → `C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe`
  - `lovely_path` → 同目录的 `version.dll`
- 启动方式：直接 exec `Balatro.exe`（不经过 Steam 客户端），Lovely 靠放在游戏目录里的 `version.dll` 完成注入——**没有** macOS 那种 `DYLD_INSERT_LIBRARIES` 环境变量机制，路径不对 / 文件缺失会直接 `RuntimeError`。本机实测：直接运行 `Balatro.exe` 即触发注入（Lovely 日志 `Game directory is at "C:\Program Files (x86)\Steam\steamapps\common\Balatro"`）。
- 平台自动探测：`platform.system().lower() == "windows"`；`BALATROBOT_PLATFORM=windows` 可强制（合法值：`darwin` / `linux` / `windows` / `native`）。
- `BALATROBOT_*` 环境变量与 macOS **完全一致**（`BALATROBOT_FAST=1`、`BALATROBOT_HEADLESS=1`、`BALATROBOT_LOVE_PATH=…`、`BALATROBOT_LOVELY_PATH=…`；全集见上游 `config.py` 的 `ENV_MAP`）。本机当前**未设置任何** `BALATROBOT_*` / `TYPESAFE_*` 环境变量，默认路径即可工作 → 主文档 §1"executor 的 `BalatroInstance(cfg, port=port)` 启动方式可以保留"的结论对 Windows 同样成立。
- 与 macOS 相同的**平台无关约束**照旧生效（主文档 §1.1 + 附录 A）：多实例共享同一存档目录（Windows 为 `%APPDATA%\Balatro`，无隔离/文件锁）、一局一进程、计分实验不用 `--fast`、pin `balatrobot>=1.5.2`、单客户端串行调用、随机端口。
- 本机实测日志路径：`%APPDATA%\Balatro\Mods\lovely\log\lovely-2026.09.21-23.24.51.log`（Lovely 0.9.0 → Steamodded v26.829.0 → BalatroBot 加载 → 服务监听 `127.0.0.1:12346`；另有一条 `Failed to connect to the debug server`，为 Lovely 调试口未开，正常）。

## 安装链（M0，Windows 版）

Steam → Balatro → Lovely（`version.dll` 放到游戏目录）→ Steamodded → balatrobot mod → `uv tool install balatrobot`（symlink 方式：管理员 PowerShell 执行 `New-Item -ItemType SymbolicLink -Path "$env:APPDATA\Balatro\Mods\balatrobot" -Target <仓库路径>`）。

⚠️ 与 macOS 相同的坑：**mod 没有预编译发行包**（上游 29 个 release 全部零附件），"下载最新版"指下载**整仓库源码压缩包**并把根目录改名成 `%APPDATA%\Balatro\Mods\balatrobot\`（对应官方文档 Windows 行路径）。

实测补充：
- 本机 mod 目录是**整仓库复制（普通目录，非 symlink）**，Lovely 注入与 mod 加载均正常 → symlink 并非必需，可留作偏好。
- CLI 未做 `uv tool install`，但 `uvx balatrobot --help` 可直接工作（首次会连带下载 uv 托管的 CPython 3.13）；正式跑批量前建议 `uv tool install "balatrobot>=1.5.2"` 固定版本。

## 首次验收清单（跑通一次后回填）

1. ✅ **已做**（2026-09-21 23:24 日志）：Lovely 0.9.0 注入成功、Steamodded v26.829.0 加载、BalatroBot v1.5.2 源码加载、21 个 endpoint 注册、HTTP 监听 `127.0.0.1:12346`。用户端视觉确认（是否进主菜单）未记录。
2. ⬜ **待跑**：`uvx balatrobot serve`（或 `uv tool install "balatrobot>=1.5.2"` 后 `balatrobot serve`），确认 CLI 能拉起游戏。
3. ⬜ **待跑**：用 `health` 打通连接（见下方命令），预期返回 `status = ok`。
4. ⬜ **待做**：拿一份**真实 gamestate JSON 存档**进仓库（主文档 §7 M0 的核心产出，后续离线开发的燃料）。
5. 回填状态：本文件已回填；主文档 §1 表格中 Windows 行仍为"⬜ 待实测"（按维护约定未反向修改，实连跑通后再定）。

## 如何复检（PowerShell）

```powershell
# OS / 硬件 / 磁盘
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, OSArchitecture, BuildNumber
Get-PSDrive C | Select-Object @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}

# Steam / Balatro 本体 / Lovely（version.dll）
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe"
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\Balatro\version.dll"
(Get-Item "C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe").VersionInfo
# 非默认库目录（例如装在 D 盘）
Get-Content "C:\Program Files (x86)\Steam\steamapps\libraryfolders.vdf" | Select-String "path"
# Balatro 游戏版本（buildid）
$acf = Get-Content "C:\Program Files (x86)\Steam\steamapps\appmanifest_2379780.acf" -Raw
foreach ($k in @('name','buildid','LastUpdated','StateFlags')) { if ($acf -match "`"$k`"\s+`"([^`"]*)`"") { "$k = $($Matches[1])" } }

# Mods 目录与三个 mod
Test-Path "$env:APPDATA\Balatro\Mods"
Get-ChildItem "$env:APPDATA\Balatro\Mods" | Select-Object Name
(Get-Content "$env:APPDATA\Balatro\Mods\smods\version.lua")  # Steamodded 版本
# balatrobot mod 是否含 1.5.2 的 #193 并发修复
Select-String -Path "$env:APPDATA\Balatro\Mods\balatrobot\src\lua\core\server.lua" -Pattern "only when no active client"
# 最近一次加载日志
Get-ChildItem "$env:APPDATA\Balatro\Mods\lovely\log" | Sort-Object LastWriteTime -Descending | Select-Object -First 1

# balatrobot CLI（未持久安装时用 uvx；CLI 无 --version）
uv tool list
uvx balatrobot --help
# 若已通过 uv tool 安装
Get-Command balatrobot

# Python（注意 py launcher 可能只注册 3.14）
uv python find 3.13
py -0p

# TYPESAFE_API_KEY（三级作用域）
[Environment]::GetEnvironmentVariable("TYPESAFE_API_KEY","Process")
[Environment]::GetEnvironmentVariable("TYPESAFE_API_KEY","User")
[Environment]::GetEnvironmentVariable("TYPESAFE_API_KEY","Machine")

# 依赖最新版本（PyPI）
(Invoke-RestMethod https://pypi.org/pypi/balatrobot/json).info.version
(Invoke-RestMethod https://pypi.org/pypi/typesafe-sdk/json).info.version

# health 实连（需先 uvx balatrobot serve 拉起游戏）
$body = '{"jsonrpc":"2.0","method":"health","id":1}'
Invoke-RestMethod -Uri http://127.0.0.1:12346 -Method Post -ContentType "application/json" -Body $body
```
