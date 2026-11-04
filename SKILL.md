---
name: cadnaa-lua
description: "Use when a task must drive CadnaA (DataKustik environmental-noise software) through its embedded Lua interface: author or adapt a CadnaA-Lua script, run grid/receiver calculations, sweep sources or variants, export levels or reports, or debug a script that executes inside CadnaA. Covers the cna namespace, cna.tables object model, polygons, variants, evaluation parameters, partial levels, LUA-Events, plus an offline mock runtime for validating scripts before touching the application."
metadata:
  triggers:
    - CadnaA
    - CadnaA-Lua
    - cna.lua
    - 噪声地图
    - 声学仿真
    - 声屏障
    - 网格计算
    - 受音者
    - noise map
    - grid calculation
---

# CadnaA-Lua 技能

> ⚠️ **重要声明（请先读）**
> 1. **未在真实 CadnaA 上完成端到端验证。** 开发机上可用的版本是 4.3 与 2022 Demo：
>    4.3 的二进制不含 Lua 运行时；2022 Demo 虽含 Lua，但被演示授权禁用。因此 `real-run`
>    通道**只完成了代码实现，没有跑通过一次真实计算**。
> 2. **仅适用于正式版（含 PRO 模块授权）的 CadnaA。** Demo/试用版无法使用本技能：
>    其 Lua 菜单项全部灰显（Execute Scripts / Open Lua Command File / Lua-Script Presets）。
> 3. **需要自行调试。** 不同版本的 API 细节、对话框、菜单文案可能不同，请按第 0 步自检后逐步调整。
> 4. `mock-run` 的物理模型是**合成的**（自由场点源衰减），只能验证脚本逻辑与数据流，
>    **不能**作为声学结果。

用 Lua 驱动 CadnaA：写脚本 → 静态检查 → 离线试跑 → 在 CadnaA 内执行 → 取回结果。

## 0. 前置检查（每次任务第一步）

在技能根目录执行（下文命令均默认在此目录）：

```powershell
python scripts\cna.py probe      # 二进制里有没有 Lua 运行时
python scripts\cna.py license    # 授权允不允许用 Lua（需 CadnaA 运行，或加 --launch）
```

两道闸门必须都过：

| 结果 | 含义 | 处理 |
|---|---|---|
| `probe` 退出 0，`"lua": true` | 构建内嵌了 Lua | 继续查授权 |
| `probe` 退出 3 | 该构建没有 Lua（如 4.3） | 真跑不可能，改 mock |
| `license` 退出 0 | Extras 下 Lua 菜单可用 | 可以 `real-run` |
| `license` 退出 4 | 授权未启用 Lua（Demo 版典型） | 真跑不可能，改 mock |

`probe` 会按常见路径查找可执行文件；找不到时用 `--exe "D:\...\cna64d.exe"` 指定。

## 1. 写脚本

```powershell
python scripts\cna.py new receiver_levels --dest runs\my_task.lua
```

模板：`receiver_levels`（受音者电平）、`grid_sweep`（网格扫描）、`variant_sweep`（变体遍历）。

```lua
assert(cna, "must run inside CadnaA-Lua")
cna.messagebox_timeout(2)              -- 无人值守必须，避免模态框卡死

for imm in cna.tables.imm:all() do
  cna.print(string.format("%s\t%.2f", imm.ID, imm.LP1 or -99))
end
```

硬规则：

| 规则 | 说明 |
|---|---|
| 路径用 `/` | Lua 把 `\x` 当转义；写 `"D:/proj/a.cna"` |
| 对象别名会失效 | `cna.load_file()` 之后旧引用不可用，需重新 `first_by_id` |
| 变体从 1 开始 | 遍历优先用 `cna.variant.all()`，不要手写 `current` |
| `save_grid` 会静默覆盖 | 文件名里带坐标/变体等参数 |
| `quit_at_once()` 丢改动 | 先 `save_file()`，且先设 `messagebox_timeout` |

API 速查见 `references/api-cheatsheet.md`；
官方文档：https://doku.datakustik.com/CadnaA/en/LUA_scripting.html

## 2. 静态检查

```powershell
python scripts\cna.py lint runs\my_task.lua --strict
```

拦截：未记录的 `cna.*` 调用、未知方法、Windows 路径反斜杠、无超时的 `quit_at_once`、
`function/end` 不配平。

## 3. 离线试跑（不需要 CadnaA）

```powershell
python -m pip install lupa
python scripts\cna.py mock-run runs\my_task.lua `
  --spec examples\spec_demo.json --out runs\my_task_out
```

产出 `stdout.txt` / `levels.csv` / `grid.csv` / `summary.json`。
mock 用真实 Lua（lupa）执行脚本，`cna` 是纯 Lua 实现；**物理是合成模型**。

spec 结构：`sources[]`（table/id/x/y/z/lw）、`receivers[]`、`grid{}`、`variants[]`、`area`。

## 4. 真跑（必须正式版 + PRO 授权）

```powershell
python scripts\cna.py real-run runs\my_task.lua `
  --project D:\path\project.cna --out runs\my_task_out --ini-write
```

机制：脚本经 `cnaevent.lua` 的 `Load` 回调注入，CadnaA 启动即打开工程并执行，
结束时 `cna.quit_at_once()` 退出；运行前后自动备份/还原已有的 `cnaevent.lua`。

`--ini-write` 必须显式给出：CadnaA 只从 `%LOCALAPPDATA%\Datakustik\CadnaA` 读事件文件
（应用本身的限制）。**该通道未在授权环境验证过**，首次使用时盯住 `out\done.txt` 的
`OK` / `ERROR: ...` / `LOAD_ERROR: ...` / `TIMEOUT`。

## 5. 交付纪律

报告里必须写清：脚本路径、spec 路径、`summary.json` 的数字、产物路径，以及本次是
**mock 还是 real**。绝不把 mock 结果当成 CadnaA 结果。

## 目录

```
SKILL.md                     本文件
scripts/cna.py               probe | license | lint | mock-run | real-run | new
scripts/selfcheck.py         一键自检（probe + 全部模板 lint + mock-run）
scripts/mock/cna_mock.lua    cna API 的纯 Lua 模拟实现
assets/templates/*.lua       可复制模板
examples/spec_demo.json      示例工程
references/api-cheatsheet.md API 速查
references/patterns.md       常见任务写法
references/runtime.md        运行机制、事件、故障排查
```

自检：`python scripts\selfcheck.py`（开发机上 10/10 通过，全部走 mock 通道）。
