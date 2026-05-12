## [2026-05-10] explore | Topic defined — HiGame Battle Script Architecture

- 研究方法:**本地代码考古**(沿用 higame-ui-script 的方法论,替代 km-websearch 的 web fetch)
- 数据源:
  - `E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame\Content\Script\` 全 Lua 树 (6829 个 .lua 文件)
    - `CommonScript\skill\ability\*` (主线 GA 实现, 21 个分支)
    - `CommonScript\skill\ExecCalc\*` (伤害结算)
    - `Content\Script\skill\*` (旧版 skill 子树, Affix/Knock/passiveability/SimpleSummon)
    - `CommonScript\actors\components\*` (skill_component / buff_component / calc_component / passiveskill_component / battle_state_*)
    - `ClientScript\skill\SkillDriver.lua` 与 `SkillManagerBase.lua`
    - `ServerScript\ai\BTTask\BTTask_HI_TryActiveAbility.lua` (AI 调用 GA 链)
  - `Source\HiGame\Public\HiAbilities\*` 与 `Public\Component\Hi*Component.h` C++ 头文件
    - `HiGameplayAbility.h` `HiPassiveGameplayAbility.h` `HiGameplayEffect.h` `HiAbilityTypes.h`
    - `HiAbilitySystemComponent.h` `HiAttributeSet.h` `HiAttributeComponent.h` `HiBuffComponent.h`
    - `HiTargetActorBase.h` `HiProjectileActorBase.h` `HiAbilityDataBase.h`
    - `Tasks\HiAbilityTask_PlayMontage.h` `Tasks\HiAbilityTask_PlaySequence.h`
    - `GameplayCue\HiGameplayCueNoitfy_Actor.h` `HiGameplayCueManager.h`
- key_questions: 12 个
- mode: new
- 关联仓库:`D:\BranKM\BranKM\projects\higame-ui-script\` (前序 UI 研究产物,已确立写作风格与 Mermaid 规范)

## [2026-05-10] websearch (local-archaeology) | Research complete — 12 raw insights, 12/12 questions covered

用本地代码考古替代 web fetch,全部 12 个 key_questions 都在 P4 工作区可核对的代码中找到答案。

### 12 个 key_questions

- Q1 — 战斗 Lua 目录结构与三层(Common/Server/Client)划分?
- Q2 — UHiGameplayAbility 与 GABase 的 C++/Lua 双向继承链与生命周期?
- Q3 — EffectContainerMap 与 Tag 驱动的"AnimNotify → GameplayEvent → ApplyEffect"结算流?
- Q4 — UHiAttributeSet / FHiHeroAttributeContainer / FHiHeroGameplayEffectContainer 中间层 UID 时序?
- Q5 — Buff 系统 (FHiBuffRow → BuffComponent → ApplyBuffByID → GE FastArray) 的全链路?
- Q6 — TargetActor / ProjectileActor / Sweep / Overlap 的命中检测与 Hit→Calc 转发?
- Q7 — ExecCalc_Damage 26 属性预取、ExecCalc_Base 等级压制的伤害公式?
- Q8 — Knock(被击)与 Counter 巫师时间(EWitchTimePriority)的状态机与优先级?
- Q9 — GameplayCue 的客户端表现层(Notify_Actor / Notify_Static / 预设 GC_*)?
- Q10 — 输入(SkillDriver) / AI(BTTask_HI_TryActiveAbility) / 动画(AnimNotify) 三大入口?
- Q11 — C++ vs Lua 边界、DDS 位面迁移 PreTransfer/PostTransfer 协议?
- Q12 — Affix 词缀、Roguelike 流派、怪物 AI 技能、换人/支援/Counter 协同 4 个进阶主题?

### 完整性自检

- WHAT: ✅(每个主题首段一句话定义)
- WHY: ✅(为何用 Tag 驱动结算 / 为何引入中间层 UID / 为何 GA Knock 单独继承等设计动机均说明)
- HOW: ✅(每个主题给出步骤 + 关键 API 名 + 真实代码片段)
- DEPTH: ✅(EWitchTimePriority 4 档/EBuffReplacePolicy 3 档/EHiGameplayEffectItemType 3 档/26 属性下标全列出)
- TRADE-OFFS: ✅(Lua 反射穿透优化 / 中间层 GE 的 Apply 时序 / DDS 的迁移代价)
- CROSS: ⚠️(单源项目, 无外部交叉验证, 但所有引用均可在 P4 工作区开源码核对)
- LINK: ✅(12 篇 wiki 之间互相引用, 共同覆盖 12 个 key_questions)

## [2026-05-11] wiki | higame-battle-script | Synthesis Complete — 12 pages

- 页面清单(按编号顺序):
  - 1. 总览 — 战斗脚本架构与目录拓扑
  - 2. GA 继承层次与生命周期
  - 3. EffectContainer 与 Tag 驱动结算流
  - 4. AttributeSet 与 Hero 属性中间层
  - 5. Buff 系统 — BuffID 到 GE 的链路
  - 6. TargetActor、Projectile 与命中检测
  - 7. ExecCalc 伤害计算
  - 8. Knock 与 Counter 巫师时间
  - 9. GameplayCue 表现层
  - 10. 输入、AI 与动画接入
  - 11. C++ 与 Lua 边界 + DDS 迁移
  - 12. 进阶 Cookbook 与常见陷阱

- 自检通过:
  - ✅ 每个 ## 级章节都有 Mermaid 图(图表密度达标)
  - ✅ 每页开头段 2-4 句说明清楚是什么
  - ✅ 所有重大主张挂 footnote `[^c01..c12]`
  - ✅ 12 页全部至少链一篇其他页
  - ✅ 12 个 key_questions 都被覆盖
  - ✅ 节点标签遵守 Rule 6/6b: 用圈号 ①②③ 替代 "1. xx", 标签内不用英文双引号
