---
title: HiGame NubisCloud 知识库 — iWiki / km 发布计划
status: planning
updated: 2026-05-11
---

# HiGame NubisCloud 知识库 — 发布计划

> 本计划描述如何把已就绪的 12 页 wiki(`D:\BranKM\BranKM\projects\higame-nubis-cloud\wiki\`)发布到公司内网 iWiki 与 KM。当前状态:**计划已就绪,待用户决策后执行**。

## 现状清点

```
D:\BranKM\BranKM\projects\higame-nubis-cloud\
├── .research-brief.yaml    21 KB    研究计划
├── index.md                14 KB    总览导航 (含 8 项颠覆性发现 + 12 页地图)
├── log.md                   6 KB    完整过程日志
├── lint-report.md           7 KB    质量自检报告 (5 严重 + 4 轻微问题已全部修复)
└── wiki\                  ~390 KB   12 页源码级 wiki / 5590 行
```

可用 MCP:
- ✅ `mcp__iWiki__*` — 公司 iWiki(空间 / 文档 / 评论 / 标签 / 搜索 / 多维表)
- ✅ `mcp__km__*` — KM 知识平台(K 吧 / 文集 / 文章 / 用户 / 活动)

发布者管理空间:
- iWiki: `branqian(branqian)` (个人空间, space_id=4010378263)
- KM: 暂未查询(需要用 `mcp__km__list-groups` 找 HiGame 项目对应 K 吧)

## 关键问题先解决

发布到内网知识库不只是"上传 12 个 .md"。需要先想清楚以下几点:

### Q1: 目标平台选 iWiki 还是 KM?

| 维度 | iWiki | KM |
|------|-------|-----|
| 协作模式 | 文档 + 评论 + 划词批注 | K 吧 + 文章 + 文集 |
| 适合内容 | 长文档 / 知识库 / 流程文档 | 技术文章 / 经验分享 / 团队知识 |
| 跨页引用 | 强 (文档树) | 文集组织 |
| 富媒体 | Markdown / DOC / 多维表 / 文件夹 | Markdown / 文章 |
| 团队可见 | 按空间权限 | 按 K 吧成员 |

**推荐:iWiki** — 12 页源码级 wiki + 9 篇 raw 笔记 + index/log/lint 合计 24 篇 markdown,**文档树结构最契合**。

### Q2: 一个 root 文档树,还是分两层?

**推荐结构**:
```
HiGame NubisCloud 知识库 (folder, root)
├── 总览 (来自 index.md, 作为入口文档)
├── wiki/ (folder)
│   ├── 1. 总览
│   ├── 2. 引擎端架构
│   ├── ... (12 页)
│   └── 12. 调试 / 性能 / 平台 / 陷阱
├── raw/ (folder, 仅团队可见)
│   ├── higame-nubis-engine-arch
│   ├── ... (9 篇)
│   └── higame-nubis-debug-and-platforms
└── 元数据/ (folder, 可选)
    ├── 研究计划 (.research-brief.yaml 转 markdown)
    ├── 过程日志 (log.md)
    └── 质量自检 (lint-report.md)
```

理由:
- **wiki/ 是给读者的最终成品** → 主推
- **raw/ 是知识来源,留作可追溯** → 副推 (避免读者迷路在 raw 里)
- **元数据/ 是过程产物** → 可选 (有助于其他团队复用本研究方法)

### Q3: 跨文档链接怎么处理?

wiki 内 254 条 `.md` 链接是相对路径(如 `2.%20...md`)。iWiki 文档体系用的是**文档 ID** 引用(`https://iwiki.woa.com/p/<doc_id>`)。

**方案**:
1. 先创建 12 个 wiki 文档,记录 doc_id
2. 再用脚本把 .md 内的相对路径替换为 iWiki 文档 URL
3. 每个 wiki 页面顶部加"返回总览 [→ iWiki 总览页](http://...)"导航

发布顺序必须是:**先批量创建 stub → 拿到 doc_id → 再批量更新内容**(否则 round 1 创建时找不到对方 doc_id)。

### Q4: 发布前要清洗哪些内容?

#### 🔴 必须清洗
- **绝对路径泄露**: 文件中可能含 `D:\BranKM\BranKM\raw\...` 与 `E:\HiProject\PerforceDev\UnrealEngine\...` 这类本机路径,需替换为相对路径或公司 P4 路径
- **内部 raw 引用**: footnote 指向 raw/*.md,在 iWiki 上要么解析为 iWiki 内的 raw 文档,要么剔除 (raw 不上传)
- **CL611225 等 P4 changelist 号**: ✅ 保留(团队内部 changelist 在公司 P4 可查,有用)
- **路径里的反斜杠**: Windows 风格 `\` → Unix `/`(iWiki 渲染更友好)

#### 🟡 可选清洗
- 把 `[推测]` 标注替换为 ⚠️ 图标或保留
- Mermaid diagram 在 iWiki 渲染兼容性测试 (公司 iWiki 是否支持?)

### Q5: 谁写,谁审?

发布前流程建议:
1. **发布到个人空间** (`branqian` space_id=4010378263) 作为预览版
2. **发给 HiGame 渲染组负责人 review** (用 `mcp__iWiki__addComment` 收意见)
3. **review 通过后,迁移到 HiGame 项目空间** (需查询 / 申请权限)
4. **正式发布,加上 `nubis-cloud / volumetric-cloud / rendering / ai-dev-guide` 标签** (`mcp__iWiki__addDocumentTags`)
5. **公告**: 在 KM 发一篇文章简介本知识库,引流到 iWiki

## 执行计划 (3 阶段)

### Phase 3.1: 预览版到个人空间 (branqian)

每步附 MCP 调用示意。

#### Step 1: 创建 root 文件夹
```
mcp__iWiki__createDocument({
  spaceid: 4010378263,
  parentid: 0,  // root
  title: "HiGame NubisCloud 知识库",
  contenttype: "FOLDER"
})
→ 返回 root_doc_id
```

#### Step 2: 创建 wiki 子文件夹
```
mcp__iWiki__createDocument({
  spaceid: 4010378263,
  parentid: root_doc_id,
  title: "wiki",
  contenttype: "FOLDER"
})
→ 返回 wiki_folder_id
```

#### Step 3: 批量创建 12 页 stub (内容为占位)
对 12 个 wiki 文件:
```
mcp__iWiki__createDocument({
  spaceid: 4010378263,
  parentid: wiki_folder_id,
  title: "1. 总览 — 4 处分散位置与跨模块 API",
  body: "<占位>",
  contenttype: "MD"
})
→ 记录 doc_id_1, doc_id_2, ... doc_id_12
```

#### Step 4: 创建总览入口 (基于 index.md)
```
mcp__iWiki__createDocument({
  spaceid: 4010378263,
  parentid: root_doc_id,
  title: "HiGame NubisCloud — 知识库总览",
  body: <index.md 内容,链接已替换为 iWiki URL>,
  contenttype: "MD"
})
```

#### Step 5: 批量更新 12 页内容
对每个 wiki 文件:
1. 读 .md 内容
2. 替换:
   - `D:\BranKM\BranKM\raw\` → 删除或替换为 iWiki raw 文档 URL
   - `E:\HiProject\PerforceDev\UnrealEngine\` → 保留为 P4 相对路径(如 `Engine/Source/...`)
   - `[第 N 页](N.%20...md)` → `[第 N 页](https://iwiki.woa.com/p/<doc_id_N>)`
3. `mcp__iWiki__saveDocument({ docid: doc_id_N, title, body })`

#### Step 6: 加标签
```
mcp__iWiki__addDocumentTags({
  docids: [root_doc_id, doc_id_1, ..., doc_id_12],
  labels: ["NubisCloud", "VolumetricCloud", "Rendering", "HiGame", "ai-dev-guide", "ue5"]
})
```

#### Step 7: 创建 raw 子文件夹与 9 篇笔记 (可选,看你判断)

同上,但 root 标题可加"⚠️ 内部研究底稿,请勿直接引用"。

### Phase 3.2: Review 与修订

1. 把 root 文档 URL 发给 HiGame 渲染组负责人
2. 用 `mcp__iWiki__getComments({ docid })` 周期性收集评论
3. 用 `mcp__iWiki__getInlineComment({ docid })` 收划词批注
4. 修订后用 `mcp__iWiki__saveDocument` 更新

### Phase 3.3: 正式发布到 HiGame 项目空间

需要先查询 HiGame 在 iWiki 的项目空间:
```
mcp__iWiki__getSpaceInfoByName({ spaceName: "HiGame" })
→ 拿到 spaceid_higame 与权限
```

如果有权限:
- 用 `mcp__iWiki__copyDocument` 把整个 root 树复制过去
- 或者用 `mcp__iWiki__moveDocument` 直接迁移

如果无权限:
- 联系 HiGame 项目管理员申请编辑权限
- 或者保留在个人空间,只把链接发给渲染组

### Phase 3.4: KM 引流文章 (可选)

```
（KM MCP 没有"发文章"工具,只能 list/search,所以这一步需要手动在 KM 网页发文）
```

文章建议标题:**"HiGame NubisCloud 体积云体系 — 12 页源码级知识库"**

文章内容:
- 100 字背景:HiGame 的 NubisCloud 是什么
- 200 字研究方法:本地代码考古 + Phase 0/1/2 三阶段
- 300 字关键发现:8 项颠覆 (Sparse Voxel 空壳 / HWRT 未接通 / Plugin 不触 RDG 等)
- 链接到 iWiki 总览页
- 标签:`Rendering / VolumetricCloud / UE5 / NubisCloud / HiGame`

## 自动化 vs 手动

| 步骤 | 自动化可行性 | 备注 |
|------|-------------|------|
| Phase 3.1 Step 1-7 | ✅ 全自动 (我可一次跑完) | 需要你 OK 才开始 |
| Phase 3.2 review | ❌ 手动 | 评论 / 修订需人工判断 |
| Phase 3.3 迁移 | 🟡 半自动 | 拿到权限后可全自动 |
| Phase 3.4 KM 文章 | ❌ 手动 | KM MCP 不支持创建文章 |

## 风险与注意

1. **个人空间 quota**: 一次创建 ~25 个文档可能触发 iWiki 个人空间限制 — 待执行时观察
2. **链接替换正确性**: doc_id 替换有错时整批文档链接乱套 — 建议先创建 1 页跑通再批量
3. **Mermaid 渲染兼容性**: 公司 iWiki 对 Mermaid 支持程度未知,可能要降级为 ASCII / SVG 图
4. **绝对路径泄露**: 上传前必须 grep `D:\` `E:\` 全部替换 — 否则可能泄露开发机信息
5. **CLAUDE.md 私密内容**: 我们 wiki 引用的 raw 笔记可能含 P4 路径与 changelist,确认这些可以对内分享

## 决策点

我没有自动开始执行(发布到知识库是大动作,影响他人)。需要你先决定:

- **(A) 现在就发布预览版到个人空间 branqian**:我会按 Phase 3.1 全自动执行(20-30 分钟),完成后给你 root 文档 URL
- **(B) 先把内容清洗(grep `D:\` `E:\` 改路径)再说**:我先派一个 agent 做路径清洗,然后再决定发不发布
- **(C) 不发布,只留计划**:本计划文件存档,等你手动执行
- **(D) 不发布到个人空间,直接申请 HiGame 项目空间权限**:我帮你查 HiGame 项目空间存在性 + 权限情况

也可以组合,例如 B → A。
