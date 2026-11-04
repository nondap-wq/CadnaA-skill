# cadnaA-skill

用 Lua 驱动 **CadnaA**（DataKustik 环境噪声计算软件）的 AI Agent 技能包：
写脚本 → 静态检查 → 离线试跑 → 在 CadnaA 内执行 → 取回结果。

> **English abstract**: A skill/CLI toolkit that lets an AI agent author and run CadnaA-Lua
> scripts to drive grid/receiver calculations, variant sweeps and result export. Includes a
> static linter and an offline mock runtime. **Not yet validated against a licensed CadnaA
> installation** — full/PRO licence required, the demo build cannot run Lua.

---

## ⚠️ 先读：三条必须知道的事

1. **未做实机测试。**
   本项目的 `real-run` 通道（把脚本注入 CadnaA 执行）**只完成了代码实现，没有跑通过一次真实计算**。
   开发机上的两个可执行版本分别是 CadnaA **4.3** 和 **2022 Demo**：
   4.3 的二进制里没有任何 Lua 符号；2022 Demo 虽含 Lua 运行时，但被演示授权禁用。
   因此函数签名、事件回调、文件路径行为等**均未经真实环境确认**，需要使用者自行调试。

2. **仅适用于正式版（含 PRO 模块授权）。**
   Demo / 试用版无法使用：其 `Extras > Execute Scripts`、`Open Lua Command File`、
   `Lua-Script Presets` 三个入口全部灰显，`Lua-Script Presets` 的子项文案直接写着
   “此功能需要PRO模块”。`probe`/`license` 两个命令会先帮你判定，不要跳过。

3. **需要自行调试。**
   不同 CadnaA 版本（2022 / 2023 / 2024 / 2025 / 2026）的 API 有增量变化：
   `cna.scandir`、`cna.get_epsg`、`cnaobj.set/get`、`cna.import_citygml` 等都是逐年加入的。
   脚本在旧版本报“unknown function”属正常，请按 `references/api-cheatsheet.md` 对照调整。

---

## 这个技能能做什么

| 能力 | 说明 |
|---|---|
| 写脚本 | 三条模板：受音者电平、网格扫描、变体遍历；附 `cna` API 速查与常见写法 |
| 静态检查 | 未记录的 `cna.*` 调用、未知方法、Windows 反斜杠路径、缺超时的 `quit_at_once`、`function/end` 不配平 |
| 离线试跑 | 纯 Lua 模拟的 `cna` 运行时（lupa），无需安装 CadnaA；产出 `levels.csv` / `grid.csv` / `summary.json` |
| 真跑 | 通过 `cnaevent.lua` 的 `Load` 回调注入脚本，启动工程即执行，结束自动退出并还原配置 |
| 环境判定 | `probe` 判断二进制是否内嵌 Lua；`license` 判断授权是否允许 Lua |

## 安装

本仓库根目录就是一个技能目录（`SKILL.md` 在根）。

```powershell
git clone https://github.com/<your-name>/cadnaA-skill.git
# 放进 Agent 的技能目录（以 DeepSeek Harness 为例）
#   %USERPROFILE%\.dsh\skills\cadnaA-skill\
```

依赖：Python 3.9+；离线试跑需要 `pip install lupa`。真跑需要 Windows + 已安装的正式版 CadnaA。

## 快速开始

```powershell
cd cadnaA-skill

python scripts\cna.py probe                 # 1. 二进制有没有 Lua
python scripts\cna.py license --launch      # 2. 授权允不允许用 Lua

python scripts\cna.py new receiver_levels --dest runs\demo.lua
python scripts\cna.py lint runs\demo.lua --strict
python scripts\cna.py mock-run runs\demo.lua --spec examples\spec_demo.json --out runs\demo_out

# 仅在 1、2 都通过时：
python scripts\cna.py real-run runs\demo.lua --project D:\proj\site.cna --out runs\demo_real --ini-write
```

`probe` / `license` 的退出码约定：

| 命令 | 退出码 | 含义 |
|---|---|---|
| `probe` | `0` | 二进制内嵌 Lua |
| `probe` | `3` | 该构建没有 Lua（如 4.3 系列） |
| `license` | `0` | 授权允许 Lua，可 `real-run` |
| `license` | `4` | 授权未启用 Lua（Demo 版典型） |

## 已验证 / 未验证

| 项 | 状态 |
|---|---|
| `lint` 规则（正确脚本全部通过、故意写错的脚本被拦下） | ✅ 已验证 |
| `mock-run` 端到端（3 个模板，真实 Lua 执行、产物落盘） | ✅ 已验证（`scripts/selfcheck.py` 10/10） |
| `probe`（4.3 判为无 Lua、2022 判为有 Lua） | ✅ 在两台真实安装上验证 |
| `license`（2022 Demo 判为未授权） | ✅ 在真实安装上验证 |
| `real-run`（注入 CadnaA 执行并取回结果） | ❌ **未验证**，缺授权环境 |
| `cnaevent.lua` 的 `Load` 回调是否真会被调用 | ❌ **未验证**（Demo 下事件子系统未启用） |

复现自检：

```powershell
python scripts\selfcheck.py
```

## 目录结构

```
SKILL.md                     Agent 技能入口（触发词、工作流、硬规则）
scripts/cna.py               命令行：probe | license | lint | mock-run | real-run | new
scripts/selfcheck.py         一键自检
scripts/mock/cna_mock.lua    cna API 的纯 Lua 模拟实现（合成物理）
assets/templates/*.lua       可复制模板
examples/spec_demo.json      示例工程 spec
references/api-cheatsheet.md API 速查（按官方文档整理）
references/patterns.md       常见任务写法（9 个片段）
references/runtime.md        运行机制、事件、故障排查
```

## 已知限制

- **mock 的物理是合成的**：点源 `Lp = LW − 20·lg(r) − 11`，线源按 `LW−3` 处理。
  只保证“接线正确”，**不能作为声学结果**，也不代表 CadnaA 的算法。
- **`real-run` 会临时改写一个系统文件**：CadnaA 只从 `%LOCALAPPDATA%\Datakustik\CadnaA\cnaevent.lua`
  读事件脚本。命令要求显式 `--ini-write`，运行前备份、结束后还原；请勿在他人正在使用 CadnaA 时运行。
- **版本差异**：模板与速查以官方在线文档（2026 时代）为准，2022 版可能缺少部分函数。
- **未覆盖**：建筑评价、隔音墙优化、SMT 等高级流程的脚本化。

## 免责声明

本项目与 DataKustik 无关，未获其背书；CadnaA 是其注册商标。
使用者需自行持有合法的 CadnaA 授权。因脚本错误、版本差异或误用产生的计算结果与工程后果，
由使用者自行承担。**在把任何输出用于正式工程前，请用 CadnaA 官方界面复核。**

## 许可

代码与文档以 MIT 许可发布，详见 [LICENSE](LICENSE)。
