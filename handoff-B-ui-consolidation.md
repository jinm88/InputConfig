# Handoff B：界面收敛 / 控制面与管理面分离（InputConfig）

> 新会话开场白建议：「读 handoff-B-ui-consolidation.md，我们继续讨论界面收敛和渐进式披露。」
> 本文档与 handoff-A（摇杆光标）是两个独立子任务，不要混做。

## 项目背景

- macOS 应用 InputConfig（Swift/SwiftUI），手柄→键鼠映射，仓库 `/Users/jin/dev/InputConfig`。
- 构建：`xcodebuild -project InputConfig.xcodeproj -scheme InputConfig -configuration Debug build`（沙盒构建需 `dangerouslyDisableSandbox: true`）。
- 产物在 Xcode 默认 DerivedData：`~/Library/Developer/Xcode/DerivedData/InputConfig-ezqbyffrlomwwedycjtmxuupcica/Build/Products/Debug/InputConfig.app`，**不要放 /tmp**。
- 签名 Apple Development / Team 977HDY9GMK，bundle id `com.inputconfig.app`。
- 回归：TestBench headless 基线 473 passed / 0 failed（纯 UI 改动也建议跑一遍确认无意外）。
- 沟通用中文；最小改动；本任务以 UI/信息架构为主，不动引擎。

## 本任务目标

1. 整个应用界面收敛（用户原话「现在看起来就是很乱」）。
2. 控制面（日常：激活/停止、状态、设备）与管理面（配置：编辑预设/绑定、校准、设置）分离，对用户好理解。
3. 渐进式信息披露：默认只给「下一步该做什么」，高级项收起来。

## 已完成调研（事实，带行号）

主窗口：`NavigationSplitView`；侧栏含搜索、Favorites、My/Built-in Presets、Trash、Input Devices；底部激活预设/设备状态；工具栏混放 Home、Stats、Settings。`InputConfig/Views/ContentView.swift` L364-437、L822-1085。

详情页（在 ContentView.swift 内，L5121-6484，约 1364 行）：头部（名称/Activate/Edit/急停）→ 速度区（条件出现）→ **Live Visualizer（约 248 行，默认展开）** → Joystick Slots（仅组名/绑定数）→ Notes → Details（Usage/Inputs/Outputs/Controllers/Storage）。Automation 在编辑器不在详情页。

编辑器：`PresetEditorView.swift` sheet 同时容纳 Gaming/Motion、可视化、分组、Automation、Storage（L355-439）。每个 binding 主行约 10 个交互点（拖拽/编号菜单/Scan/输入/组合/输出/加输出/Options/删除/常显 Notes），`InputConfig/Views/Components/BindingRowView.swift` L230-569；Options 默认关但展开为双栏十几项（L3010-3181、L3866-3974、L4117-4305、L4438-4609）。

菜单栏弹窗：`InputConfig/Services/MenuBarController.swift` L826-1216，已有控制面雏形（设备状态、当前预设、启动/停止、切换预设、急停），混入 Configure/Settings/Help/Quit（footer 已有齿轮等图标位）。菜单栏图标刚改为：未激活=空心描边、激活=实心 template、告警=橙色+!（本仓库最新改动）。

Settings：General、Controllers、Advanced、About 四个自绘侧栏 tab（`InputConfig/Views/SettingsView.swift` L146-172）。

### 「乱」的五个来源

1. Home 同屏堆 What's New + Smart Maker + 教程 + 5 个演示 + Recent + Quick Tips（ContentView L2510-2827）
2. 详情页运行/编辑/可视化/笔记/元数据混排 + Visualizer 默认展开 + Activity Log 容器默认显示（内容折叠，DebugLogView.swift L15）
3. 编辑器行信息密度过高
4. 弹窗控制面与管理面入口混排
5. 调试入口（Test Bench）在应用菜单（InputConfigMain.swift L1816-1845），分散

## 已提出的目标架构（待用户确认）

```
L0 菜单栏弹窗 = 控制面：状态/当前预设/启动停止/切换预设/设备/急停；
              Configure/Settings/Help 收 footer
L1 主窗口     = 管理面：详情页瘦身=头部+绑定概览+速度；
              Visualizer/Notes/Details 默认折叠；Activity Log 默认隐藏；
              Home 新手元素只在首次运行或主动点教程出现
L2 编辑器     = 深度配置：行内只留 输入→输出 主链路，其余进 Options
```

分期建议：B1 详情页+Home 收敛（默认值/折叠态，小改见效快）→ B2 弹窗纯控制面化（小改）→ B3 编辑器行收敛（中改，动行布局）。

上一轮建议顺序 **B1/B2 → A1/A2**，用户尚未确认优先级。

## 待讨论问题

1. 三层架构是否符合用户心智？侧栏的 Input Devices、Stats 归哪层？
2. 新手引导元素的触发时机（仅首次？Home 常驻一个入口？）。
3. B1/B2/B3 的取舍与验收标准（什么叫「不乱了」）。

## Suggested skills

- 打磨方案：`grill-me`、`codebase-design`
- 定稿后落地：`to-spec` / `implement-spec`；UI 验证可用 `TRAE-computer-use-ptc`（操作 app UI 前必须先调用）
