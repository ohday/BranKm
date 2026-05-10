## [2026-05-10] explore | Topic defined — HiGame UI Script Architecture

- 研究方法:**本地代码考古**(替代 km-websearch 的 web fetch)
- 数据源:`E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame\Content\Script\ui\` 全目录
  + `Source/HiGame/Public/UI/` 与 `Source/HiGame/Public/UINavigation/` C++ 头文件
- key_questions: 12 个
- mode: new

## [2026-05-10] websearch (local-archaeology) | Research complete — 7 raw sources, 12/12 questions covered

- 用本地代码考古替代 web fetch,落地到共享 raw/:
  - id 48: higame-ui-framework-overview.md
  - id 49: higame-ui-window-lifecycle.md
  - id 50: higame-ui-mvvm.md
  - id 51: higame-ui-unlua-and-events.md
  - id 52: higame-ui-input-and-navigation.md
  - id 53: higame-ui-cpp-boundary.md
  - id 54: higame-ui-cookbook-and-pitfalls.md

### 完整性自检
- WHAT: ✅(每个主题首段一句话定义)
- WHY: ✅(8 层 UI 栈的设计动机/MVVM 的耦合解耦动机/UnLua 三层目录的 CS 分隔动机已说明)
- HOW: ✅(每个主题给出步骤 + 关键 API 名 + 真实代码片段)
- DEPTH: ✅(WindowState 枚举值/绑定 BindWay 数值/Layer 8 个枚举/Handler 链顺序均显式给出)
- TRADE-OFFS: ✅(C++ vs Lua 边界, 已废弃文件标注)
- CROSS: ⚠️(单源项目, 无外部交叉验证, 但所有引用均可在 P4 工作区开源码核对)
- LINK: ✅(7 篇笔记之间互相引用, 共同覆盖 12 个 key_questions)

## [2026-05-10] wiki | higame-ui-script | Synthesis Complete — 12 pages, 7 sources

- 页面清单(按编号顺序):
  - 1. 总览 — UI 脚本三层目录与启动链
  - 2. UIWindowBase 生命周期
  - 3. UI 栈与 Layer
  - 4. UIInfo 配置与 DataTable 注册
  - 5. 2D vs 3D 双轨
  - 6. MVVM 数据绑定
  - 7. UnLua 绑定与热更新
  - 8. UI 事件原语与 UINotifier
  - 9. 输入系统
  - 10. C++ 与 Lua 边界
  - 11. 新 UI Cookbook 与真实模板
  - 12. 常见陷阱与自检清单

- 自检通过:
  - ✅ 每个 ##  级章节都有 Mermaid 图(图表密度达标)
  - ✅ 每页开头段 2-4 句说明清楚是什么
  - ✅ 所有重大主张挂 footnote `[^48..54]`
  - ✅ 12 页全部至少链一篇其他页
  - ✅ 12 个 key_questions 都被覆盖
  - ⚠️ 节点标签遵守 Rule 6/6b: 用圈号 ①②③ 替代 "1. xx", 标签内不用英文双引号

## [2026-05-10] wiki | higame-ui-script | Synthesis Complete — 12 pages, 7 sources

- 页面清单(按编号顺序):
  - 1. 总览 — UI 脚本三层目录与启动链
  - 2. UIWindowBase 生命周期
  - 3. UI 栈与 Layer
  - 4. UIInfo 配置与 DataTable 注册
  - 5. 2D vs 3D 双轨
  - 6. MVVM 数据绑定
  - 7. UnLua 绑定与热更新
  - 8. UI 事件原语与 UINotifier
  - 9. 输入系统
  - 10. C++ 与 Lua 边界
  - 11. 新 UI Cookbook 与真实模板
  - 12. 常见陷阱与自检清单

- 自检通过:
  - ✅ 每个 ##  级章节都有 Mermaid 图(图表密度达标)
  - ✅ 每页开头段 2-4 句说明清楚是什么
  - ✅ 所有重大主张挂 footnote `[^48..54]`
  - ✅ 12 页全部至少链一篇其他页
  - ✅ 12 个 key_questions 都被覆盖
  - ⚠️ 节点标签遵守 Rule 6/6b: 用圈号 ①②③ 替代 "1. xx", 标签内不用英文双引号
