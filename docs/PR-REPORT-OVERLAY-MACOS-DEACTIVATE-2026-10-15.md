# PR 报告：overlay 合成窗在 macOS 应用失活时被 Cocoa 隐藏

> **基线**：`579b401`（2026-10-15，`main`）
> **分支**：`fix/overlay-macos-deactivate-visibility`　**日期**：2026-10-15
> **范围**：2 个文件（实现 1、测试 1）
> **关联**：[`PR-REPORT-BUBBLE-ON-TOP-2026-10-13.md`](PR-REPORT-BUBBLE-ON-TOP-2026-10-13.md)（其中已登记「`WA_MacAlwaysShowToolWindow` 路径未验；overlay 拓扑行为不变故未验」）、issue #229 同族的 `platform_mac` 平台层

## 一、核心特性

**问题**：macOS 上（默认 overlay 拓扑）切到任何其他应用，整只桌宠从桌面消失；点 Dock 图标把应用激活回来，它又出现。用户观感是「桌宠没退出但形象不见了」。

**根因**：`OverlayWindow` 带 `Qt.Tool` 标志，而 Cocoa 会在应用失活（resigns active）时自动把 Tool 窗口 order out。项目**早就知道**这件事，也**早就有修复**——`pet/platform_mac.py::_keep_macos_tool_window_visible()`（设 `WA_MacAlwaysShowToolWindow`），并从 2026-08-18 起在 legacy 的 `PetWindow` 里无条件调用（`pet/window.py:567`、`:1135`）。

**漏点**：overlay 单合成窗新架构 2026-09-23 才落成（`6032c30`），**没有把这条平台补丁移植过来**。全历史核查：

```
$ git log -S "_keep_macos_tool_window_visible" -- pet/overlay_window.py
(空 —— 从未出现)
$ git log -S "WA_MacAlwaysShowToolWindow" -- pet/overlay_window.py
(空 —— 从未出现)
```

又因为**默认拓扑就是 overlay**（`is_overlay_topology()` == `not stream_capture_mode`），绝大多数 macOS 用户走的正是唯一没打补丁的那条路；legacy 用户完全看不到，CI 三平台又都固定 `offscreen`，所以它能活到现在，只在真机暴露。

**用户可见变化**：修后 overlay 桌宠在切到别的应用时**保持可见**（与 legacy 行为一致）。

| # | 能力 | 说明 |
|---|---|---|
| 1 | overlay 桌宠跨应用切换保持可见 | 构造 `OverlayWindow` 时补下发 `WA_MacAlwaysShowToolWindow` |
| 2 | 不改变 z-order / 置顶语义 | 只设 native 属性，不动 flags，不重建原生窗口 |

**红线 / 不变量**：① `OverlayWindow` 的窗口标志（`FramelessWindowHint \| Tool \| WindowStaysOnTopHint \| NoDropShadowWindowHint`）不得被本改动改变；② 非 darwin 平台上不得新增任何行为；③ 不得引入运行时 objc 调用（保持 `platform_mac` 既有的「构造期设属性、不做 native 调用」口径）。

## 二、修改文件说明

### 实现

| 文件 | 增删 | 改动意图 |
|---|---|---|
| `pet/overlay_window.py` | **+8 / −1** | ① import 行补 `_keep_macos_tool_window_visible`（该文件本来就已从同一模块 import `_mac_no_window_shadow_flag`，故只加一个名字）；② `OverlayWindow.__init__` 在 `setAttribute(WA_NoSystemBackground)` 之后调用 `_keep_macos_tool_window_visible(self)`。**为什么放构造处**：`platform_mac._mac_no_window_shadow_flag` 的既有注释已确立「标志只能在 flags 构造处给」的口径；属性同理只在构造期设一次，避免可见窗口上 `setWindowFlag` 触发原生窗口重建（频闪）。**为什么不动 flags**：`WA_MacAlwaysShowToolWindow` 只是 native 属性，不参与 flags 计算，所以不重建窗口。 |

### 测试

| 文件 | 增删 | 覆盖 |
|---|---|---|
| `tests/test_overlay_window.py` | **+42 / −0** | ① `test_overlay_tool_window_stays_visible_when_application_deactivates`：darwin 下真实构造 `OverlayWindow()`，断言 `testAttribute(WA_MacAlwaysShowToolWindow) is True`。**用真实窗口断言、不用源码文本**——漏调用时该属性为 False，断言精确红在缺属性那一行（见 §六 红绿证据）。② `test_overlay_window_keeps_qt_tool_flag`：前提守卫——本修复只在窗口仍是 `Qt.Tool` 时才有意义；将来若有人把 overlay 改成普通 `Window`，这条会逼那次改动显式面对它，而不是悄悄绕过。 |

### 未改动（明确说明）

- `pet/window.py`（legacy）：**没动**。它一直是对的，本改动是让 overlay 与它对齐。
- `pet/platform_mac.py`：**没动**。helper 本身正确；问题在调用点缺失。
- `pet/overlay_shell.py`：**没动**。在 `OverlayWindow` 构造处修即可覆盖 shell 的全部创建路径（shell 只 `OverlayWindow(...)` 构造，不自行拼窗口），无需第二处调用。
- `pet/platform_passthrough.py` / `build_window_flags`：**没动**。与穿透、阴影无关。

## 三、实现要点

```python
from .platform_mac import _keep_macos_tool_window_visible, _mac_no_window_shadow_flag
...
        self.setWindowFlags(Qt.WindowType.FramelessWindowHint
                            | Qt.WindowType.Tool
                            | Qt.WindowType.WindowStaysOnTopHint
                            | _mac_no_window_shadow_flag())
        self.setAttribute(Qt.WidgetAttribute.WA_TranslucentBackground)
        self.setAttribute(Qt.WidgetAttribute.WA_NoSystemBackground)
        _keep_macos_tool_window_visible(self)          # ← 新增
```

`_keep_macos_tool_window_visible` 只在 `sys.platform == 'darwin'` 时下发，非 darwin 是空操作；它**不触碰 flags、不调 objc runtime**，所以不触发原生窗口重建，也不受 `WA_MacAlwaysShowToolWindow` 在运行时切换不可靠这一已知限制影响（既有的 `_mac_set_window_level` 才是走 objc 的那条路，本改动不涉及）。

**取舍与备选**：
1. *在 `OverlayShell` 里对 overlay 窗口补设属性* —— 否。窗口的所有权在 `OverlayWindow` 自己（flags/属性都在它的构造处给），放 shell 会造成「构造处与 shell 两处真相」。
2. *把 overlay 改成普通 `Window`* —— 否。那会让桌宠进入窗口列表/Mission Control，且与 legacy 捕获模式的取舍混为一谈。
3. *在 `platform_mac` 里加一个「对所有 Tool 窗口生效」的全局钩子* —— 否。超出本缺陷范围，且会把气泡/灵动岛等已有各自处理的窗口一并卷进来（它们已各自设过该属性：`speech_bubble.py:337/848`、`dynamic_island.py:540`、`desktop_notify.py:39`、`screen_capture/result_panel.py:317`）。

## 四、性能分析

**方法（可复现）**：`QT_QPA_PLATFORM=offscreen .venv/bin/python /tmp/bench_overlay_ctor.py`
**环境**：macOS 27.2 (26B5112a) · Apple Silicon · CPython 3.13.15 · PySide6 6.12.0 · offscreen 平台

| 指标 | 实测 | 归属 |
|---|---|---|
| `OverlayWindow()` 构造耗时（**含**修复，N=300 ×5 轮） | 0.055 / 0.047 / 0.045 / 0.047 / 0.048 ms，**中位数 0.047 ms** | 既有热路径（仅在窗口创建时） |
| `OverlayWindow()` 构造耗时（**撤掉**修复，同口径） | 0.056 / 0.049 / 0.043 / 0.048 / 0.049 ms，**中位数 0.049 ms** | 基线 |
| `_keep_macos_tool_window_visible` 单次绝对成本 | **0.414 µs** | 新增 |

**结论（逐条）**：

1. **稳态开销**：**无变化**。本属性只在 `OverlayWindow.__init__` 下发一次，不进入任何 tick/绘制/事件路径；构造耗时中位数 0.047 vs 0.049 ms，**差值的符号都不稳定、落在轮间区间 0.043–0.056 ms 内，属噪声**，故不声称「更快/更慢」。
2. **新增路径的绝对成本与触发频率**：单次 **0.414 µs**（一次性 `setAttribute`）；**触发频率 = 每个 `OverlayWindow` 实例一次**（= 进程启动、或跨拓扑重入时各一次），**不是每帧、不是每 tick**。按 0.414 µs 计，相对 8 ms 帧预算占比 **0.005%**，且不落在帧路径上。
3. **新系统调用 / 网络 / 磁盘 / 线程**：**都没有**。`setAttribute` 是 Qt 进程内属性位；非 darwin 平台该函数为纯空操作。特别地**未引入 objc runtime 调用**（`platform_mac._mac_set_window_level` 那条 ctypes/objc 路径本改动不涉及），因此不存在「offscreen 平台 winId() 非 NSView 导致 SIGSEGV」那类既有风险。
4. **内存与缓存**：**无增长**。不新增容器、不新增缓存、不改 `_dirty` / sprite 结构。

## 五、实机运行记录

**环境**：macOS 27.2（26B5112a）· Apple Silicon · 用户机真机（非 CI、非 mock）· 被测对象为已打包的 `dsh-pet-standalone-webm-chat.app`（PyInstaller onedir，运行 PySide6 6.12 / Qt 6.12 冻结构建）

### 5.1 根因现场复现（overlay 拓扑 —— 缺补丁）

探针：`CGWindowListCopyWindowInfo` 分别取 `OnScreenOnly`(1) 与 `All`(0)，统计 owner 为 `dsh-pet-standalone-webm-chat` 的窗口数。

```
### 前台 ASN:0x0-0x45045（桌宠）
[A. 桌宠为前台] 在屏=2  总计=7
    ON  layer=8  alpha=1.0  1728x1117@0,0     ← 全屏透明合成窗
    ON  layer=8  alpha=1.0  270x61@1186,768
>>> 切到 Finder <<<
### 前台 ASN:0x0-0x13013（Finder）
[B. Finder 为前台] 在屏=0  总计=7            ← 全部被 order out
>>> 切回桌宠 <<<
[C. 切回桌宠后] 在屏=1  总计=7
```

注意 `alpha` 全程 **1.0** —— 窗口不是变透明，是**被移出屏幕**；进程未退出，故 Dock 图标仍在，点它激活应用即 order 回来。**与用户报告的症状逐条吻合**。

### 5.2 机制确认（legacy 拓扑 —— 带补丁），同一台机器、同一套操作

`PET_RENDER_TOPOLOGY=legacy` 启动（该路径调用了 `_keep_macos_tool_window_visible`）。拓扑确认走日志：legacy 独有的一行出现 —— `共享全屏监视线程已启动`（overlay 拓扑刻意不启该共享 watcher）。

```
[A. legacy · 桌宠前台]       在屏=1    ON layer=3 461x281@782,550
[B. legacy · Finder 前台]    在屏=1    ON layer=3 461x281@782,550   ★跨应用仍在屏★
[C. legacy · Sublime 前台]   在屏=1    ON layer=3 461x281@782,550   ★第三个应用仍在屏★
```

**这是本 PR 最关键的证据**：同一台 macOS 27.2 + Qt 6.12 上，**带该属性的路径跨应用切换保持可见，不带该属性的路径全部被 order out**。因此「补这一条属性即可修复」不是推断，而是同机 A/B 的直接结论。

### 5.3 用户可见行为确认

修复前（overlay）：用户报「一切换到别的应用桌宠形象就不见，点 Dock 栏选回去才显示」。本地探针在 5.1 完整复现该现象。修复后（legacy 实测）：切到 Finder / Sublime Text 后桌宠仍在屏。**修复后的 overlay 构建本体未在真机运行**（见 §七 限制 1），但 5.2 已证明差量正是该属性。

### 5.4 边界与「本该失败」路径

- **非 darwin**：`_keep_macos_tool_window_visible` 为空操作，属性不下发；由 `test_overlay_window_keeps_qt_tool_flag` 守卫「窗口仍是 Qt.Tool」这一前提，避免将来改成普通 Window 后本修复变成无声的摆设。
- **全屏应用自动隐藏**：用户配置里 `auto_hide_fullscreen=true` 是**另一条**独立机制。诊断时把它临时置 false 复测，**现象不变**（Finder 前台仍在屏 0），据此排除它；随后已还原为 `true`。
- **Stage Manager**：`defaults read com.apple.WindowManager` → `GloballyEnabled = 0`，排除。

## 六、测试与验证

| 门 | 命令 | 结果 |
|---|---|---|
| 静态检查 | `.venv/bin/python -m ruff check pet tests scripts` | **All checks passed!** |
| 聚焦（本文件全量） | `QT_QPA_PLATFORM=offscreen pytest -q tests/test_overlay_window.py` | **19 passed, 1 skipped** |
| 新增两条用例 | `pytest -q tests/test_overlay_window.py -k "deactivates or qt_tool_flag"` | **2 passed, 18 deselected** |
| 断言有效性（红→绿） | `git stash push pet/overlay_window.py` 后重跑上面的 `-k deactivates` | **1 failed** —— `AssertionError: assert False is True`（`testAttribute(WA_MacAlwaysShowToolWindow)` 为 False），精确红在缺属性那一行 |
| 既有 legacy 用例未被影响 | `pytest -q tests/test_desktop_pet_features.py -k macos_tool_window` | **1 passed** |
| 全量 | `python -m pytest -q` | **未跑**（见下） |

**关于「未跑全量」的诚实登记**：本机只建了最小复现环境（`.venv` 内仅 `PySide6==6.12.0` + `pytest` + `ruff`），用于跑上面的聚焦门；`requirements.txt` 里的 `numpy / rapidocr / onnxruntime / opencv` 等未安装，全量套件在本机跑不起来。本改动是构造期一行属性下发、不触及数据模型/配置迁移/生命周期/线程/平台分派以外的分支，按 `AGENTS.md`「聚焦加相关测试」口径提交，**全量门交由 CI 执行**。

## 七、已知限制与后续

1. **修复后的 overlay 冻结构建未在真机运行** —— 无法直接验证「打了补丁的 overlay 窗口在真机失活时保持可见」。不能自动验证的原因：用户手上是 PyInstaller 冻结的 `.app`（`Contents/Resources/base_library.zip` + 内嵌 PYZ），改字节码会破坏 ad-hoc 代码签名、需重新签名，不适合作为取证手段。**替代证据**是本机同机 A/B（§5.1 vs §5.2）：两条路径唯一相关的差量就是该属性，且结果与预期方向一致。合入后在真机重跑 `CGWindowListCopyWindowInfo` 探针即可闭环。
2. **未验证 macOS 26 及更早版本** —— 本机只有 macOS 27.2。该属性是 Qt 的公开 macOS 属性（`Qt::WA_MacAlwaysShowToolWindow`），行为在 Qt 6.x 稳定，但更早系统未实测。
3. **未覆盖 overlay 的其它子窗** —— 气泡 / 灵动岛 / 桌面通知 / 截图结果浮层已各自设过该属性（见 §三 备选 3 的清单），本 PR 不动它们。若后续出现「overlay 主窗可见但某个子窗仍消失」，按同一 helper 补即可。
4. **不影响 topology 判定** —— 本修复不能替代「overlay 为何是默认拓扑」的设计讨论；它只让默认拓扑在 macOS 上行为正确。

## 八、风险与回滚

- **影响面**：仅 `OverlayWindow` 构造期多一次属性下发；对 Windows / Linux 是空操作。不改 flags、不改 z-order、不改输入穿透、不改脏矩形。
- **开关**：无。没有新增配置键，**无配置迁移**。
- **回滚**：`git revert` 即可，无落盘残留状态（不写文件、不写配置）。回滚后 overlay 桌宠恢复「切应用即被 Cocoa 隐藏」的既有行为，legacy 不受影响。
- **签名 / 打包**：不涉及构建脚本、不改 `packaging/`、不新增随包二进制。
