## [2026-05-11] explore | Topic defined — HiGame NPC Subsystem Architecture

- 研究方法:**本地代码考古**(替代 km-websearch 的 web fetch),与 `higame-ui-script` (2026-05-10) 同源方法论
- 数据源:
  - `E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame\Content\Script\npc\` 全目录(~50 文件)
  - `Content\Script\kittens\{active_object,state_flow,node_handle,event_driven}\` 全目录(~40 文件)
  - `Content\Script\actors\common\{NPC.lua,BaseNPC.lua,components\npc\,changehead\}`(~15 文件)
  - `Content\Script\ai\{BTTask,BTDecorator,BTService,BTCommon}\` 全目录(~10 文件)
  - `Source\HiGame\Public\{HiAI,BT,Npc}\*.h` + `Public\Component\HiNpc*.h, HiNPC*.h`(~15 头)
  - `Source\HiGame\Private\{HiAI,BT,Npc}\*.cpp`(~10 实现)
- key_questions: 16 个(Q1-Q16,见 index.md)
- mode: new
- openspec change: `Projects/HiGame/openspec/changes/author-higame-npc-script-wiki/`
- 计划产出: 1 index + 1 log + 15 raw + 16 wiki = 33 markdown 文件

## [2026-05-11] websearch (local-archaeology) | Research complete — 15 raw sources, 16/16 questions covered

用本地代码考古替代 web fetch,落地到共享 raw/:
- npc-01 — 总览/拓扑/启动链(NPC.lua, BaseNPC.lua, npc_active_object.lua, npc_const.lua, NpcSettings.h)
- npc-02 — Kittens ActiveObject + Mail(11 个 active_object/ 文件)
- npc-03 — Kittens StateFlow(13 个 state_flow/ 文件含 task 子目录)
- npc-04 — Kittens NodeHandle + EventFlow + EventSystem(node_handle/ + event_driven/ 全部)
- npc-05 — 13 server 状态 + 10 client 状态 + npc_state_config + state_mail_handlers
- npc-06 — 35+ NodeComponent 实现(server/client/common 三栏对照)
- npc-07 — 30 EF 节点(28 Action + 2 Switch)+ 4 类 Mixin context
- npc-08 — EFConditionRegistry + 2 个内置 EFCond
- npc-09 — SmartObject + WayPoint Free/Claimed/Occupied 三态
- npc-10 — 5 件套 CustomTask(BehaviorTree/ChaseGame/Escort/Klutz/Tracking)+ ChangeHead/Move 子目录
- npc-11 — 9 个 Actor 级 BP 组件 + ChangeHead Attractor;解决 Open Question Q2
- npc-12 — HiAIController + DetourCrowd + PathFollowing + AvoidanceManager(C++ 5 文件)
- npc-13 — 8 个 C++ NPC Component(MovementComponent / SklMeshChangeComponent / DialogueCameraControllerComp / NpcAnimInstance 等)
- npc-14 — BehaviorTree(BossPunk + DonNavigation + Boss decorator + BTCompositeNode);标注 Content/Script/ai/ 根目录为空,真实代码在 ServerScript/ai/
- npc-15 — npc_const.lua 30+ 常量表全枚举,作为反查索引

### 完整性自检
- WHAT: ✅(每个主题首段一句话定义)
- WHY: ✅(13 状态机的设计动机/Direct vs Environment 优先级动机/Significance 三级分级动机已说明)
- HOW: ✅(每个主题给出步骤 + 关键 API 名 + 真实代码片段)
- DEPTH: ✅(NodeCompBindingKey 17 键全表/Enum_Mail_Type 60+ 类型按 7 簇分类/EF 节点 28+2 全表/Significance Distance Table verbatim 均显式给出)
- TRADE-OFFS: ✅(剧情 NPC ActiveObject 路径 vs 战斗 NPC BehaviorTree 路径分野;Lua NodeComponent vs C++ Actor Component 平行系统)
- CROSS: ⚠️(单源项目, 无外部交叉验证, 但所有引用均可在 P4 工作区开源码核对)
- LINK: ✅(15 篇笔记之间互相引用, 共同覆盖 16 个 key_questions)

### 解决的 Open Questions
- Q1(condition_agent / puppet 空目录):在 wiki/1 标注"规划占位",未为它们分页 ✅
- Q2(Actor 级 BP 组件 vs Lua NodeComponent 关系):npc-11 §5 给出对照表,确认是平行的两套系统,部分通过 add_component 桥接 ✅
- Q3(术语表):嵌入 wiki/1 §术语速记表,与 UI wiki 风格一致 ✅
- 额外发现:Content/Script/ai/BTTask 根目录为空,真实 BT Lua 代码在 ServerScript/ai/ 镜像位置(npc-14 §9 已记录)

## [2026-05-11] wiki | higame-npc-script | Synthesis Complete — 16 pages, 15 sources

- 页面清单(按编号顺序):
  - 1. 总览 — NPC 全栈拓扑与启动链(215 行)
  - 2. Kittens — ActiveObject 与 Mail(282 行)
  - 3. Kittens — StateFlow(310 行)
  - 4. Kittens — NodeHandle 与 NodeComponent(339 行)
  - 5. NpcActiveObject 与 13 状态机(499 行 — 最长页)
  - 6. Mail 类型与 Handler 路由(361 行)
  - 7. Node 组件矩阵(382 行)
  - 8. EventFlow — 28 个 Action 节点(343 行)
  - 9. EventFlow Condition 与 Switch(220 行)
  - 10. SmartObject 与 WayPoint(309 行)
  - 11. CustomTask 五件套(366 行)
  - 12. ChangeHead 与 Performance 表演栈(266 行)
  - 13. Significance 与性能分级(295 行)
  - 14. HiAIController + 寻路与规避(335 行)
  - 15. BehaviorTree 与战斗 NPC(333 行)
  - 16. Cookbook + 陷阱 + 自检清单(450 行 — cookbook 性质)

- 总计行数: ~8936 行(wiki ~5005 + raw ~3447 + index 160 + log)

- 自检通过(16/16 wiki):
  - ✅ 每页 `##` 二级章节都有 mermaid / 代码 / 表格(图表密度达标)
  - ✅ 每页 mermaid 数 ≥ 3(范围 3-8),共 ~78 个 mermaid 图
  - ✅ 每页 footnote 数 ≥ 2(范围 2-31),全部解析到 `raw/` 下真实笔记
  - ✅ 每页 cross-link 数 ≥ 3(范围 3-29),互链密度高
  - ✅ 16 个 key_questions 全部被覆盖矩阵显式标注
  - ✅ mermaid 节点标签内零英文双引号(subgraph 标题、code 块内、中文叙述引号不计)
  - ✅ mermaid 内部枚举使用 ①②③(总览图、4 层架构图、Switch 类型对比等)
  - ✅ 所有 API 名 / 字段名 / 路径名 verbatim 来自源代码

- 文件清单:
  - 1 index.md(含 16Q × 16P 覆盖矩阵 + mermaid 知识地图)
  - 1 log.md(三段式过程日志:Topic defined / Research complete / Synthesis Complete)
  - 15 raw/ 笔记(npc-01 ~ npc-15)
  - 16 wiki/ 主页面(1 ~ 16,5 簇分组:基础底座 4 / NPC 核心 5 / 扩展能力 4 / 战斗 AI 2 / 落地 1)

- 验收通过:全部满足 spec `higame-npc-knowledge-base` 7 个 Requirement 的 20+ Scenario。零 P4 工作区改动。

## [2026-05-11] websearch (local-archaeology) | Research complete — 15 raw sources, 16/16 questions covered

用本地代码考古替代 web fetch,落地到共享 raw/:
- npc-01 — 总览/拓扑/启动链(NPC.lua, BaseNPC.lua, npc_active_object.lua, npc_const.lua, NpcSettings.h)
- npc-02 — Kittens ActiveObject + Mail(11 个 active_object/ 文件)
- npc-03 — Kittens StateFlow(13 个 state_flow/ 文件含 task 子目录)
- npc-04 — Kittens NodeHandle + EventFlow + EventSystem(node_handle/ + event_driven/ 全部)
- npc-05 — 13 server 状态 + 10 client 状态 + npc_state_config + state_mail_handlers
- npc-06 — 35+ NodeComponent 实现(server/client/common 三栏对照)
- npc-07 — 30 EF 节点(28 Action + 2 Switch)+ 4 类 Mixin context
- npc-08 — EFConditionRegistry + 2 个内置 EFCond
- npc-09 — SmartObject + WayPoint Free/Claimed/Occupied 三态
- npc-10 — 5 件套 CustomTask(BehaviorTree/ChaseGame/Escort/Klutz/Tracking)+ ChangeHead/Move 子目录
- npc-11 — 9 个 Actor 级 BP 组件 + ChangeHead Attractor;解决 Open Question Q2
- npc-12 — HiAIController + DetourCrowd + PathFollowing + AvoidanceManager(C++ 5 文件)
- npc-13 — 8 个 C++ NPC Component(MovementComponent / SklMeshChangeComponent / DialogueCameraControllerComp / NpcAnimInstance 等)
- npc-14 — BehaviorTree(BossPunk + DonNavigation + Boss decorator + BTCompositeNode);标注 Content/Script/ai/ 根目录为空,真实代码在 ServerScript/ai/
- npc-15 — npc_const.lua 30+ 常量表全枚举,作为反查索引

### 完整性自检
- WHAT: ✅(每个主题首段一句话定义)
- WHY: ✅(13 状态机的设计动机/Direct vs Environment 优先级动机/Significance 三级分级动机已说明)
- HOW: ✅(每个主题给出步骤 + 关键 API 名 + 真实代码片段)
- DEPTH: ✅(NodeCompBindingKey 17 键全表/Enum_Mail_Type 60+ 类型按 7 簇分类/EF 节点 28+2 全表/Significance Distance Table verbatim 均显式给出)
- TRADE-OFFS: ✅(剧情 NPC ActiveObject 路径 vs 战斗 NPC BehaviorTree 路径分野;Lua NodeComponent vs C++ Actor Component 平行系统)
- CROSS: ⚠️(单源项目, 无外部交叉验证, 但所有引用均可在 P4 工作区开源码核对)
- LINK: ✅(15 篇笔记之间互相引用, 共同覆盖 16 个 key_questions)

### 解决的 Open Questions
- Q1(condition_agent / puppet 空目录):在 wiki/1 标注"规划占位",未为它们分页 ✅
- Q2(Actor 级 BP 组件 vs Lua NodeComponent 关系):npc-11 §5 给出对照表,确认是平行的两套系统,部分通过 add_component 桥接 ✅
- Q3(术语表):嵌入 wiki/1 §术语速记表,与 UI wiki 风格一致 ✅
- 额外发现:Content/Script/ai/BTTask 根目录为空,真实 BT Lua 代码在 ServerScript/ai/ 镜像位置(npc-14 §9 已记录)

## [2026-05-11] wiki | higame-npc-script | Synthesis Complete — 16 pages, 15 sources

- 页面清单(按编号顺序):
  - 1. 总览 — NPC 全栈拓扑与启动链(215 行)
  - 2. Kittens — ActiveObject 与 Mail(282 行)
  - 3. Kittens — StateFlow(310 行)
  - 4. Kittens — NodeHandle 与 NodeComponent(339 行)
  - 5. NpcActiveObject 与 13 状态机(499 行 — 最长页)
  - 6. Mail 类型与 Handler 路由(361 行)
  - 7. Node 组件矩阵(382 行)
  - 8. EventFlow — 28 个 Action 节点(343 行)
  - 9. EventFlow Condition 与 Switch(220 行)
  - 10. SmartObject 与 WayPoint(309 行)
  - 11. CustomTask 五件套(366 行)
  - 12. ChangeHead 与 Performance 表演栈(266 行)
  - 13. Significance 与性能分级(295 行)
  - 14. HiAIController + 寻路与规避(335 行)
  - 15. BehaviorTree 与战斗 NPC(333 行)
  - 16. Cookbook + 陷阱 + 自检清单(450 行 — cookbook 性质)

- 总计行数: ~8936 行(wiki ~5005 + raw ~3447 + index 160 + log)

- 自检通过(16/16 wiki):
  - ✅ 每页 `##` 二级章节都有 mermaid / 代码 / 表格(图表密度达标)
  - ✅ 每页 mermaid 数 ≥ 3(范围 3-8),共 ~78 个 mermaid 图
  - ✅ 每页 footnote 数 ≥ 2(范围 2-31),全部解析到 `raw/` 下真实笔记
  - ✅ 每页 cross-link 数 ≥ 3(范围 3-29),互链密度高
  - ✅ 16 个 key_questions 全部被覆盖矩阵显式标注
  - ✅ mermaid 节点标签内零英文双引号(subgraph 标题、code 块内、中文叙述引号不计)
  - ✅ mermaid 内部枚举使用 ①②③(总览图、4 层架构图、Switch 类型对比等)
  - ✅ 所有 API 名 / 字段名 / 路径名 verbatim 来自源代码

- 文件清单:
  - 1 index.md(含 16Q × 16P 覆盖矩阵 + mermaid 知识地图)
  - 1 log.md(三段式过程日志:Topic defined / Research complete / Synthesis Complete)
  - 15 raw/ 笔记(npc-01 ~ npc-15)
  - 16 wiki/ 主页面(1 ~ 16,5 簇分组:基础底座 4 / NPC 核心 5 / 扩展能力 4 / 战斗 AI 2 / 落地 1)

- 验收通过:全部满足 spec `higame-npc-knowledge-base` 7 个 Requirement 的 20+ Scenario。零 P4 工作区改动。
