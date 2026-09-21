# 机器环境检查：macOS

> 本仓库跨 OS 开发（主 macOS / Windows），机器环境检查按 OS 拆分为独立文件。
> 本文件是被 [`TYPESAFE_BALATRO_FEASIBILITY.md`](../TYPESAFE_BALATRO_FEASIBILITY.md) §1 引用的 **macOS 实测记录**。
> 维护约定：换机器 / 重装环境后，照"如何复检"跑一遍并更新"检查快照"与依赖表；**不要反向修改主文档的结论**，主文档只引用本文件的状态摘要。

## 检查快照

| 项 | 值 |
| --- | --- |
| 检查时间 | 评估时（`typesafe-balatro` 尚无 git 提交） |
| OS / 硬件 | macOS 26.6.2（Apple Silicon M1） |
| 状态 | ✅ **已实测确认（不是推测）** |
| 复检命令 | 见文末"如何复检" |

## 依赖检查结果

| 依赖 | 状态 | 说明 |
| --- | --- | --- |
| Steam | ❌ 未安装 | `~/Library/Application Support/Steam` 不存在 |
| Balatro 本体 | ❌ 未安装 | 无 `Balatro.app`，无 Steam 库目录 |
| Balatro Mods 目录 | ❌ 不存在 | `~/Library/Application Support/Balatro/Mods` 不存在 |
| `balatrobot` CLI | ❌ 未安装 | `~/.local/bin` 下无此命令 |
| `typesafe-sdk` | ❌ 未安装 | 不在全局 site-packages（wheel 可正常下载，纯 Python，macOS 支持） |
| `TYPESAFE_API_KEY` | ❌ 未设置 | 环境变量中无 |
| Python | ✅ 3.14.6（uv 可用） | 两个参照仓库都要求 `>=3.13`，uv 会自动拉 3.13 |

**结论**：写代码可以立刻开始（大部分模块可离线单测），但"跑通第一局"之前必须先完成下面的安装链。新建的 macOS 机器也按本文件复检一遍再开工。

## macOS 侧的启动路径（已对照上游 `balatrobot/platforms/macos.py` 与 1.5.2 wheel 逐文件核实，不是推测）

- `MacOSLauncher` 存在，并内置了正确的默认路径：
  - `love_path` → `~/Library/Application Support/Steam/steamapps/common/Balatro/Balatro.app/Contents/MacOS/love`
  - `lovely_path` → 同目录的 `liblovely.dylib`
- 注入方式：`DYLD_INSERT_LIBRARIES=liblovely.dylib`，直接 exec LÖVE 二进制。**游戏不能用 Steam 客户端启动**（Steam 的已知 bug），launcher 绕开它。
- `balatrobot.Config` 暴露了本项目最需要的几个开关：**`fast`（官方注释"10x speed"）**、**`headless`**、`no_shaders`、`fps_cap`、**`gamespeed`（默认 4）**、**`animation_fps`（默认 10）**、`no_reduced_motion`、`render_on_api`，以及 `balatro_path` / `lovely_path` / `love_path`。
- 全部可以通过 `BALATROBOT_*` 环境变量配置（`BALATROBOT_FAST=1`、`BALATROBOT_HEADLESS=1`、`BALATROBOT_LOVE_PATH=…`、`BALATROBOT_LOVELY_PATH=…`；全集见上游 `config.py` 的 `ENV_MAP`），而 `BalatroInstance(config, session_id, **overrides)` 支持 `port=` 之类的覆盖。
- 平台自动探测：`platform.system().lower() == "darwin"`；需要时可用 `BALATROBOT_PLATFORM` 强制。

## 安装链（M0）

Steam → Balatro → Lovely（`liblovely.dylib`）→ Steamodded → balatrobot mod → `uv tool install balatrobot`。

⚠️ 两个已知坑（源自主文档 §7 M0）：

1. **mod 没有预编译发行包**（上游 29 个 release 全部零附件），"下载最新版"指的是下载**整仓库源码压缩包**并把根目录改名成 `~/Library/Application Support/Balatro/Mods/balatrobot/`；
2. **先裸跑一次游戏**确认 macOS 26.6.2 + M1 能起来（主文档 R12：若原生崩溃，退回 x86_64 `liblovely.dylib` + Rosetta）。

`/Users/huangqingming/Workspace/evalatro/scripts/setup-local.mjs`（同工作区）已经把这套流程写成脚本，可直接照抄步骤省去踩坑。

## 如何复检

```bash
# Steam 与 Balatro 本体（LÖVE 二进制）
ls "$HOME/Library/Application Support/Steam/steamapps/common/Balatro/Balatro.app/Contents/MacOS/love"
# Lovely（liblovely.dylib）
ls "$HOME/Library/Application Support/Steam/steamapps/common/Balatro/liblovely.dylib"
# Mods 目录
ls "$HOME/Library/Application Support/Balatro/Mods"
# balatrobot CLI
command -v balatrobot
# typesafe-sdk（本仓库使用方）
uv tool list
# TYPESAFE_API_KEY
env | grep TYPESAFE_API_KEY
# Python
python3 --version
```

复检通过后：更新本文件"检查快照"（时间 / OS / 状态），把新发现的问题（版本不兼容等）补进主文档 §8"仍待实测"。