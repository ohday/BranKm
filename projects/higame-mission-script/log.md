## [2026-05-11] explore | Topic defined — HiGame Mission System

- 研究方法:**本地代码考古**(替代 km-websearch 的 web fetch)
- 数据源:
  - `Plugins/HiMission/Source/{HiMission,HiMissionEditor}/{Public,Private}/...`(全套头文件)
  - `Plugins/HiFlowGraph/Source/{HiFlow,HiAIFlowEditor}/...`
  - `Plugins/FlowGraph/Source/{Flow,FlowEditor,FlowSearch}/...`(社区版)
  - `Content/Script/mission/...`(全套 Lua 业务层)
- key_questions: 13 个
- mode: new
- 参考前作:`D:\BranKM\BranKM\projects\higame-ui-script`(UI 范例)— 套用 index/log/wiki 三件套结构

## [2026-05-11] research (local-archaeology) | Coverage complete

- 18 份核心头文件 / Lua 入口已读全
- FlowGraph 社区版 API 已确认(`UFlowAsset`/`UFlowNode`/`FFlowPin`)
- HiAIFlowImporter 工具链入口已确认(`UHiAIFlowImporter::ImportJsonToAsset`)
- 既有官方文档已确认:`Plugins/HiMission/Docs/external-message-system.md`(本 wiki 不重复)

### 完整性自检
- WHAT: ✅(每个主题首段一句话定义)
- WHY: ✅(双执行模式动机/State 替代 WorkAction 动机/JSON 驱动动机/UnLua UPARAM 限制动机均说明)
- HOW: ✅(每个主题给出步骤 + 关键 API 名 + 真实代码片段或时序图)
- DEPTH: ✅(7 EHiFlowStateStatus 值 / 6 MultiClientStrategy / 5 TransitionTrigger / 4 CompletionRequirement / 3 SyncPolicy 全显式给出)
- TRADE-OFFS: ✅(C++ vs Lua 边界、CProtobuf 限制、UnLua UPARAM 回写问题、DEPRECATED 老 API)
- CROSS: ⚠️(单源项目,无外部交叉验证,但所有引用均可在 P4 工作区开源码核对;UI wiki 第 6/8 章互链)
- LINK: ✅(13 篇笔记互相引用,共同覆盖 13 个 key_questions)

## [2026-05-11] wiki | higame-mission-script | Synthesis Complete — 13 pages, 18 sources

- 页面清单(按编号顺序):
  - 1. 总览 — 4 插件 + Lua 子树
  - 2. Flow 三件套 — 社区 FlowGraph 速览
  - 3. HiMissionFlowAsset 解剖
  - 4. 节点四件套生命周期
  - 5. Mission 层级与子图
  - 6. State 机制 — StateTree 风格
  - 7. FlowInstance 运行时
  - 8. 网络同步策略矩阵
  - 9. 持久化与回档
  - 10. TaskBridge 与 Lua Task 三模式
  - 11. Lua 节点工厂与命名约定
  - 12. TrackingScope ProgressScope 与 UI
  - 13. Cookbook — 加一个新任务

- 自检通过:
  - ✅ 每页 ## 级章节都至少一张 Mermaid 图(图表密度达标)
  - ✅ 每页开头段 2-4 句说明清楚是什么
  - ✅ 所有重大主张挂 footnote `[^X-Y]`,精确到 `:行号`
  - ✅ 13 页全部至少链一篇其他页
  - ✅ 13 个 key_questions 都被覆盖
  - ✅ 节点标签遵守 km-wiki Rule 6/6b: 用圈号 ①②③ 替代 "1. xx",标签内不用英文双引号
  - ✅ DEPRECATED 标注:对老 5.0 API、`FHiMissionInstance`/`AvatarData`/`ProgressInstance`/`NodeInstance`/`ProgressRecord`/`AvatarSyncData` 老结构、`mission_node_ maskinput.lua` 反面案例均显式标出

### 与 UI wiki 的互链
- 第 12 章 → UI wiki 第 6 章 MVVM 数据绑定
- 第 12 章 → UI wiki 第 8 章 UI 事件原语与 UINotifier
