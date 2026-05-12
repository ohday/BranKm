## [2026-05-11] explore | Topic defined — HiGame Mechanism Architecture

- 研究方法：**本地代码考古**（替代 km-websearch 的 web fetch），与 higame-ui-script 同样路径
- 数据源：
  - `E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame\Source\HiGame\Public\InteractSystem\` 9 头文件 + 9 cpp
  - `Projects\HiGame\Content\Script\actors\common\interactable\` 60+ 物件
  - `Content\Script\{Server,Client,Common}Script\actors\Puzzle\` 5 个范式机关
  - `Plugins\HiMissionPuzzle\Source\HiMissionPuzzle\Public\` 任务驱动子系统
  - `Plugins\LogicalChains\Source\LogicalChains\Public\` 第三方逻辑信号插件
- key_questions: 15 个
- mode: new

### 范围抉择记录

用户提了 4 个范围选项（A-D）。最初提议范围 B（解谜全栈）+ trap 单列 + interactable 基类多用例。
经多轮 AskUserQuestion 收敛到：

| 选项 | 用户答复 |
|---|---|
| Scope | B. 解谜全栈（推荐） |
| Trap | 纳入但单列一章 |
| Interactable | 共用基类讲清楚, 然后挑选一些更多的用例来详细说明 |
| Outline | 扩充为 16 页 |
| Location | BranKM/projects 下独立项目（推荐） |

### 16 页骨架

1 总览 + InteractSystem 4 章（②③④⑤）+ Puzzle 范式 4 章（⑥⑦⑧⑨）+ 族谱 5 章（⑩⑪⑫⑬⑭）+ 落地 2 章（⑮⑯）

## [2026-05-11] websearch (local-archaeology) | Research complete — 6 raw sources, 15/15 questions covered

并行启动 6 个 Explore 子代理，每个负责一块代码考古，落地到共享 raw/：

- id 55: higame-mechanism-interactsystem.md (InteractSystem C++ 9 头文件)
- id 56: higame-mechanism-interactable-base.md (基类三件套 + 14+ 用例)
- id 57: higame-mechanism-puzzle-pattern.md (Puzzle 5 标准范式)
- id 58: higame-mechanism-missionpuzzle.md (MissionPuzzle Subsystem 全链路)
- id 59: higame-mechanism-trap-gas.md (Trap 三条血脉 + GAS 三种触发)
- id 60: higame-mechanism-status-ro-database.md (Status/RO/存盘三横切)

### 完整性自检

- WHAT: ✅（每个主题首段一句话定义）
- WHY: ✅（机关分四种血脉的设计动机 / bNewStatusFW 双框架并存的过渡设计 / D4 fallback 的兜底动机 / RO ≠ UE Replication 的痛点说明 已说明）
- HOW: ✅（每个主题给出步骤 + 关键 API 名 + 真实代码片段）
- DEPTH: ✅（EInteractItemState 4 态值 / EInteractAction 7 个枚举 / E_StatusFlowRaw 6 态 / 5 个机关每个玩法 / GAS 三种触发方式 / 16 类陷阱 均显式给出）
- TRADE-OFFS: ✅（C++ vs Lua 边界 / 新旧状态机分水岭 / GAS 三种触发优劣表 / 已废弃 API 标注 / IA_* 未实现项标注）
- CROSS: ⚠️（单源项目，无外部交叉验证；但所有引用均可在 P4 工作区开源码核对，且子代理的考古结果之间互相印证）
- LINK: ✅（6 篇笔记之间互相引用，共同覆盖 15 个 key_questions）

### 6 个子代理并行成果

- **InteractSystem 考古**: 47 条 footnote，含 cpp 行号；发现 IA_SetLocoState/ActivateAbility/InactToSublevelEvent 三个枚举值 switch 中无 case；TRACE_CHANNEL 宏全模块零引用；InteractRange profile 在 DefaultEngine.ini 未定义；UInteractCharacterComponent FindInteractInfo 全注释 == 死的
- **Interactable 用例集考古**: 100 条 footnote；扫了 14+ 用例（Door / MachineDoor / SealedDoor / elevator 系列 / chest 三档 / bottle / note / Album / telephone / PumpkinGame / gravity / faithhourglass / dreamrebuild / TimedChallenges）；发现 TeleportElevator/JumpItem/MiaoQuDoor/GrottoFluvialDoor/Tombstone/WarChessboard/CombinationLock 等无 .lua 文件全在蓝图
- **Puzzle 范式考古**: 67 条 footnote；5 个机关每个细节展开；归纳 CMC 平滑动力学配置表（MaxAcc=600 等）；记录 SurveillanceBird suppress-restore 技巧；DancingSofa _ForceSaveBeforeDestroy 补丁
- **MissionPuzzle 考古**: 40 条 footnote；定位独立插件 Plugins/HiMissionPuzzle/；发现文件名陷阱 mission_puzzle_subsystem.lua 实为 MissionTrackingManager_S；带外 reliable RPC Client_NotifyPuzzleEnded 解决 bAutoUnregister 中间帧被吞；FHiPuzzleSnapshot.operator== 故意排除 RemainingTime
- **Trap 考古**: 32 条 footnote；三条血脉对照（actors/common/trap/ vs SwapHead vs interactable 特殊群）；GAS 三种触发方式实测对照（AddBuffByID 配置驱动 / MakeOutgoingSpec 携带数值 / SendMessage HandleHitEvent 击飞）；FenNuBao 是最干净 GE 直送范式
- **Status/RO/存盘考古**: 35 条 footnote；定位 LogicalChains 是外挂插件 Plugins/LogicalChains/（Copyright Space Raccoon Game Studio）；扫齐 15 个 RO 文件清单；解释为什么 Multicast 重广播+Destroy 直销毁是 SaveGame 反序列化不标脏的修法

### 关键发现（无法 grep 直接得到的洞见）

1. **机关在代码里有四种血脉**：actors/Puzzle / Plugins/HiMissionPuzzle + actors/MissionPuzzle / actors/common/interactable / actors/common/trap + SwapHead 等
2. **InteractSystem 不是机关本身，是底层框架** —— 不要混淆"挂了 UInteractItemComponent"和"是个机关"
3. **D4 fallback 的"D4"含义代码注释中未解释** —— 推测是该机关的"防御等级 4 级"或"第 4 类边界条件 (Defense #4)"，是 sprint 修 bug 时的内部编号
4. **bAutoRegister 必须 true** —— 这是 Sprint 11 修 ASC null deref @ 0x190 崩溃的核心结论
5. **Multicast 重广播 + Destroy 直销毁** —— 是 SaveGame 反序列化不标脏 bug 的根本修法
6. **Save 归一化原则** —— 任何过渡态在 Save 前必须归一到稳定终态

## [2026-05-11] wiki | higame-mechanism | Synthesis Complete — 16 pages, 6 sources

页面清单（按编号顺序）：

1. 总览 — 机关全景与边界
2. InteractSystem 概念与 Trace channel
3. InteractRange — 范围检测
4. InteractItem — 状态机与 Action 类型
5. InteractManager + Widget — 管理与 UI 提示
6. Puzzle 三层目录与启动链
7. Status 状态机 + LogicalSignal
8. RO 复制对象与 Multicast 重广播
9. 存盘 / 恢复 / D4 fallback
10. MissionPuzzle Subsystem 全链路
11. Interactable 基类三件套
12. 用例集 A — 门 / 电梯 / 平台
13. 用例集 B — 容器 / 物品 / 机械谜题
14. Trap 战斗陷阱与 GAS 集成
15. Cookbook — 新机关 / 新可交互物件模板
16. 陷阱与自检清单

### 自检通过

- ✅ 每个 ## 级章节都有至少一张 Mermaid 图（图表密度达标）
- ✅ 每页开头段 2-4 句说明清楚是什么
- ✅ 所有重大主张挂代码位置（文件:行号）
- ✅ 16 页全部至少链一篇其他页
- ✅ 15 个 key_questions 都被覆盖
- ✅ 节点标签遵守 Rule 6/6b: 用圈号 ①②③ 替代 "1. xx", 标签内不用英文双引号
- ⚠️ 部分 Mermaid 节点用 `()` 与 `:` 等可能引发渲染问题的字符已用 `<br/>` 替代（保持兼容性）

### 知识地图

```
基础(1) + InteractSystem(4) + Puzzle 范式(4) + 族谱(5) + 落地(2) = 16
```

写一个新机关时推荐顺序：① → ⑥ → ⑦ → ⑨ → ⑪ → ⑮
排错或优化时：⑯ → ⑦ → ⑧ → ⑨ → ⑩

### 4 套真实代码模板（位于 ⑮ Cookbook）

- **模板 A**: 标准 Puzzle 机关（参考 GhostMechanism 三层）
- **模板 B**: 简单 F 键交互物（参考 Album）
- **模板 C**: MissionPuzzle 任务驱动（参考 path_chasing_puzzle）
- **模板 D**: 战斗陷阱（参考 SwapHead BattleTrap）

### 16 类陷阱（位于 ⑯ 陷阱与自检）

- InteractSystem 4 类
- 状态机 4 类
- 存盘 4 类
- 其他 4 类（含 seal_fragment 反模式 / 文件名陷阱）

### 14 项自检清单 + 5 类性能项

- 蓝图配置 4 项
- 必加组件 3 项
- Lua 三层 6 项
- 集成测试 4 项

性能项覆盖 Tick 检测受击、Overlap 替代 Tick、RPC 内反复 require、同步加载阻塞、Updater 全员每帧刷新。

## 项目落地

- wiki 位置：`D:\BranKM\BranKM\projects\higame-mechanism\`
  - `index.md` — 知识地图 + 关键事实 + 页面目录 + 数据来源 + 质量说明
  - `wiki/01-overview.md ... wiki/16-pitfalls.md` — 16 页内容（注：index.md 即 ① 总览，wiki/ 内文件 02-16）
  - `log.md` — 本文件
- raw 笔记：`D:\BranKM\BranKM\raw\`（与 higame-ui-script 共享）
  - higame-mechanism-interactsystem.md (#55)
  - higame-mechanism-interactable-base.md (#56)
  - higame-mechanism-puzzle-pattern.md (#57)
  - higame-mechanism-missionpuzzle.md (#58)
  - higame-mechanism-trap-gas.md (#59)
  - higame-mechanism-status-ro-database.md (#60)
- sources.yaml: 在原文件末尾追加 6 条 source（id 55-60），保持与 ui 研究统一索引

## 总结

总页数：16
总参考来源数：6（全部为本地代码考古产物）
覆盖率：15 / 15 关键问题全部覆盖
空白领域：D4 fallback 的"D4"具体含义代码注释中未解释；C++ 端 RO 基类位于 DDS 框架插件外（HiGame Source 内未找到）；IInteractLevelExecutorInterface 当前未连线
