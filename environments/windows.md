# 机器环境检查：Windows

> 本仓库跨 OS 开发（主 macOS / Windows），机器环境检查按 OS 拆分为独立文件。
> 本文件是被 [`TYPESAFE_BALATRO_FEASIBILITY.md`](../TYPESAFE_BALATRO_FEASIBILITY.md) §1 引用的 **Windows 检查清单**。
> ⚠️ 状态：⬜ **尚未实测**——下表"预期"一栏全部来自已核对的上游源码 / 官方文档（与 macOS 侧"已实测"是不同量级），拿到 Windows 机器后按"首次验收清单"跑一遍、回填状态，再更新主文档结论。
> 维护约定：换机器 / 重装环境后照"如何复检"跑一遍并更新"检查快照"与依赖表；**不要反向修改主文档的结论**。

## 检查快照

| 项 | 值 |
| --- | --- |
| 检查时间 | ⬜ 未检查 |
| OS / 硬件 | ⬜ 待填（Windows 10 / 11 x64；Balatro.exe 是 x64，ARM64 Windows 需 x64 模拟，不在本评估范围） |
| 状态 | ⬜ **待实测** |
| 复检命令 | 见文末"如何复检"（PowerShell） |

## 依赖检查（预期路径；已对照上游 `balatrobot/platforms/windows.py` 与官方安装文档核实）

| 依赖 | 预期路径 / 判定 | 状态 |
| --- | --- | --- |
| Steam | 默认库目录 `C:\Program Files (x86)\Steam\`；若装到别的盘，读 `<Steam>\steamapps\libraryfolders.vdf` 找实际库位置 | ⬜ 待检查 |
| Balatro 本体 | `C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe`（= Windows launcher 的默认 `love_path`） | ⬜ 待检查 |
| Lovely（Windows 版） | 与 `Balatro.exe` 同目录的 `version.dll`（= 默认 `lovely_path`；Windows 上 Lovely 走 DLL 注入，不是 dylib） | ⬜ 待检查 |
| Balatro Mods 目录 | `%APPDATA%\Balatro\Mods\`（= `C:\Users\<用户名>\AppData\Roaming\Balatro\Mods\`；官方安装文档 Windows 行） | ⬜ 待检查 |
| `balatrobot` CLI | `uv tool install balatrobot` 后位于 `%USERPROFILE%\.local\bin\balatrobot.exe` | ⬜ 待检查 |
| `typesafe-sdk` | 项目 venv / `uv tool` 安装；wheel 是 `py3-none-any`，平台无关 | ⬜ 待检查 |
| `TYPESAFE_API_KEY` | 用户环境变量 | ⬜ 待检查 |
| Python | `uv python find 3.13`（或 `py -3.13 --version`）；两个参照仓库都要求 `>=3.13` | ⬜ 待检查 |

## Windows 侧的启动路径（已对照上游 `balatrobot/platforms/windows.py` 核实）

- `WindowsLauncher` 存在，内置默认路径：
  - `love_path` → `C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe`
  - `lovely_path` → 同目录的 `version.dll`
- 启动方式：直接 exec `Balatro.exe`（不经过 Steam 客户端），Lovely 靠放在游戏目录里的 `version.dll` 完成注入——**没有** macOS 那种 `DYLD_INSERT_LIBRARIES` 环境变量机制，路径不对 / 文件缺失会直接 `RuntimeError`。
- 平台自动探测：`platform.system().lower() == "windows"`；`BALATROBOT_PLATFORM=windows` 可强制（合法值：`darwin` / `linux` / `windows` / `native`）。
- `BALATROBOT_*` 环境变量与 macOS **完全一致**（`BALATROBOT_FAST=1`、`BALATROBOT_HEADLESS=1`、`BALATROBOT_LOVE_PATH=…`、`BALATROBOT_LOVELY_PATH=…`；全集见上游 `config.py` 的 `ENV_MAP`）→ 主文档 §1"executor 的 `BalatroInstance(cfg, port=port)` 启动方式可以保留"的结论对 Windows 同样成立。
- 与 macOS 相同的**平台无关约束**照旧生效（主文档 §1.1 + 附录 A）：多实例共享同一存档目录（Windows 为 `%APPDATA%\Balatro`，无隔离/文件锁）、一局一进程、计分实验不用 `--fast`、pin `balatrobot>=1.5.2`、单客户端串行调用、随机端口。

## 安装链（M0，Windows 版）

Steam → Balatro → Lovely（`version.dll` 放到游戏目录）→ Steamodded → balatrobot mod → `uv tool install balatrobot`（symlink 方式：管理员 PowerShell 执行 `New-Item -ItemType SymbolicLink -Path "$env:APPDATA\Balatro\Mods\balatrobot" -Target <仓库路径>`）。

⚠️ 与 macOS 相同的坑：**mod 没有预编译发行包**（上游 29 个 release 全部零附件），"下载最新版"指下载**整仓库源码压缩包**并把根目录改名成 `%APPDATA%\Balatro\Mods\balatrobot\`（对应官方文档 Windows 行路径）。

## 首次验收清单（跑通一次后回填）

1. 裸跑一次游戏（不接任何代码），确认 `Balatro.exe` + `version.dll` 注入能进主菜单；记录 Windows 版本与 Lovely / Steamodded 实际版本。
2. `uvx balatrobot serve`（或安装后 `balatrobot`），确认能拉起游戏。
3. 用 `health` 打通连接：

   ```powershell
   $body = '{"jsonrpc":"2.0","method":"health","id":1}'
   Invoke-RestMethod -Uri http://127.0.0.1:12346 -Method Post -ContentType "application/json" -Body $body
   ```

   预期返回 `status = ok`（注意 PowerShell 里 `curl` 是 `Invoke-WebRequest` 的别名，别用 `-d` 那套）。
4. 拿一份**真实 gamestate JSON 存档**进仓库（主文档 §7 M0 的核心产出，后续离线开发的燃料）。
5. 把结果回填本文件的"检查快照"与依赖表；如有版本 / 行为差异（Lovely 注入失败、Steam 库在非默认盘等），补记到主文档 §8"仍待实测"。

## 如何复检（PowerShell）

```powershell
# Steam / Balatro 本体 / Lovely（version.dll）
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe"
Test-Path "C:\Program Files (x86)\Steam\steamapps\common\Balatro\version.dll"
# 非默认库目录（例如装在 D 盘）
Get-Content "C:\Program Files (x86)\Steam\steamapps\libraryfolders.vdf" | Select-String "path"
# Mods 目录
Test-Path "$env:APPDATA\Balatro\Mods"
# balatrobot CLI
Get-Command balatrobot
# typesafe-sdk（本仓库使用方）
uv tool list
# TYPESAFE_API_KEY
Get-ChildItem Env:TYPESAFE_API_KEY
# Python
uv python find 3.13
```