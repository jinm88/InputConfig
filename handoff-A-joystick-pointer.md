# Handoff A：摇杆 → 光标操作体验（InputConfig）

> 新会话开场白建议：「读 handoff-A-joystick-pointer.md，我们继续讨论摇杆光标的场景分析和设计。」
> 本文档与 handoff-B（界面收敛）是两个独立子任务，不要混做。

## 项目背景

- macOS 应用 InputConfig（Swift/SwiftUI），手柄→键鼠映射，仓库 `/Users/jin/dev/InputConfig`。
- 唯一真实用户：用手柄办公，LB/RB/LT/RT 当左侧修饰键叠加按键输出快捷键；摇杆控制光标是核心诉求。
- 构建：`xcodebuild -project InputConfig.xcodeproj -scheme InputConfig -configuration Debug build`（沙盒构建需 `dangerouslyDisableSandbox: true`）。
- 产物在 Xcode 默认 DerivedData：`~/Library/Developer/Xcode/DerivedData/InputConfig-ezqbyffrlomwwedycjtmxuupcica/Build/Products/Debug/InputConfig.app`，**不要放 /tmp**（会被系统清理）。
- 签名 Apple Development / Team 977HDY9GMK，bundle id `com.inputconfig.app`，跨构建保留辅助功能权限。
- 回归测试：TestBench headless（DistributedNotification `inputconfig.debug.testbench`，脚本 `/tmp/post_dnc.swift` 若被清理需重建；报告在容器 `tmp/testbench.txt`，首行 `passed=N failed=M`）。改引擎后必跑，上次基线 473 passed / 0 failed。
- 沟通用中文；最小改动；不删常驻修饰键行；修复二（同修饰键组赢家仲裁+扳机滞回）用户明确暂缓，不要主动开始。

## 本任务目标

把摇杆控制光标做得真正符合日常操作电脑的需求。用户要求**先详细分析使用场景再动手**，不要直接写代码。

## 已完成调研（事实，带行号）

管线：轴值经低通、滞后、径向内外死区线性重映射、逐轴方向筛选及 Linear/Expo/SCurve、可选按住 ramp → `speed×幅值`；120Hz 轮询（空闲 30Hz）；MouseMotionPump 约 125Hz 按 dt 积分、余数结转，发 `mouseMoved`。**无独立 maxSpeed，无随时间/位移的指针加速。**
- `InputConfig/Services/MappingEngine.swift` L4831-5034（曲线/死区/ramp）
- `InputConfig/Services/MouseMotionPump.swift` L96-212（积分泵）

可调参数：`deadzone`、`outerDeadzone`、`sensitivityCurve`、`variableSensitivity`、`rampMs`、输出 `speed`（UI 1–50）、全局 `sensitivityMultiplier/scrollMultiplier`。见 `InputConfig/Models/Preset.swift` L179-326；UI 在 `InputConfig/Views/Components/BindingRowView.swift` L4117-4305 和 `ContentView.swift` L5593-5613。

其他指针路径：触摸板、陀螺 pointing（锚点差值→位移）与摇杆共用速率累加器，非互斥。`clickX/Y` 定点跳转点击不恢复；`recenterPointer` 居中。MappingEngine L2670-2692、L4201-4216、L4890-5014。

点击/滚动/拖拽：`mouseButton` 支持按住与 Toggle 锁存；拖拽=左键保持+`mouseMotion`→`leftMouseDragged`；滚动分连续 `mouseWheel` 与固定 `mouseWheelStep`。MappingEngine L2491-2552、InputSimulator L821-844。

精细机制：只有起步 ramp（初速 1/8 渐满）和 Expo 曲线；**无专用慢速/加速档**，但可用 chord 门控 + 不同 speed 的两行映射不改代码先实现「按住某键=精细模式」。

## 已确认的短板（代码证据）

1. 无指针加速 → 远快近准不可兼得（MappingEngine L4956-4981）
2. 响应曲线逐轴计算，非线性会改变运动方向（斜推漂移），应改极坐标幅值（L4976-5034）
3. 滚动无加速曲线
4. 无专用精细档（可用配置凑合，见上）

## 场景分析（上一轮已输出给用户，待确认）

① 远距离跳转→要快；② 中距离定位→不冲头；③ 精细操作→慢稳；④ 滚动阅读→起步匀速长滚加速；⑤ 点击拖拽→已够用。核心矛盾=快与准共用固定速度。

建议实现顺序（未获用户最终确认）：
- A1 指针加速（持续满推随时间/幅值爬升，轻推保持精细）——价值最大
- A2 曲线改极坐标——与 A1 同段代码顺手做
- A3 精细档修饰键——先配置验证手感
- A4 滚动加速——独立小改最后做

## 待用户回答的开放问题

1. 摇杆光标是否纯办公场景（网页/文档/切窗口），有无游戏内视角控制？决定 A1 加速曲线调参方向。
2. A1-A4 的优先级与手感偏好（加速快慢、是否喜欢系统鼠标那种加速感）。
3. 精细档想绑哪个键/触发方式（LT 半按？按住 RB？）。

## Suggested skills

- 讨论/打磨方案：`grill-me` 或 `codebase-design`
- 定稿后落地：`to-spec` / `implement-spec`，引擎改动配 `tdd`（TestBench 回归）
- 复杂运行时问题才用 `TRAE-debugger`
