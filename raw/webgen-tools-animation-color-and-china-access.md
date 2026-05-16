# 补充工具 + 动画/配色系统深度细节

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-05-16
- **Project**: llm-webgen-tools-2026
- **Relevant to**: Q1 (additional tools), Q2 (animation deep dive), Q3 (color/font), Q5 (iteration), Q7 (failure modes)
- **Sources drawn from**:
  1. [Google Stitch AI Review 2026 — Banani](https://www.banani.co/blog/google-stitch-ai-review)
  2. [Google Stitch Review 2026 — Vibecoding](https://vibecoding.app/blog/google-stitch-review)
  3. [Google Stitch — ToolsCompare](https://toolscompare.ai/tool/google-stitch)
  4. [Replit Agent 3 launch — Sohu](https://www.sohu.com/a/934255967_122328931)
  5. [Replit Agent 3 — ToolifyAI](https://toolifyai.cn/article/483)
  6. [Lovable vs Replit vs Manus — Manus blog](https://contact@manus.im/zh-cn/blog/lovable-vs-replit-vs-manus)
  7. [Magic MCP component generation — TencentCloud](https://cloud.tencent.com/developer/mcp/server/10027)
  8. [TweakCN AI ShadCN theme editor — CSDN](https://blog.csdn.net/gitblog_00319/article/details/151291623)
  9. [Lovable visual edits InfoQ](https://xie.infoq.cn/article/3741c049bbbf13046f4dfe74f)
  10. [Same.new alternatives — Shipper](https://shipper.now/ru/same-new-alternatives/)
  11. [a0.dev review — Declom](https://declom.com/a0.dev)
  12. [Mocha guide — getmocha.com](https://blog.getmocha.com/how-to-build-a-website-with-ai-for-a-business-without-coding)
  13. [Tempo Labs review — vibe-coding.uk](https://vibe-coding.uk/tools/tempo-labs)
  14. [Claude Artifacts complex animation — synthesis](https://www.beautifulcode.in/blog/framer-motion-performance-issues)
  15. [BeautifulCode Framer Motion performance](https://www.beautifulcode.in/blog/framer-motion-performance-issues)
  16. [LinkedIn Aditya Valvi Claude+Blender](https://www.linkedin.com/pulse/claude-blender-aditya-valvi)
  17. [QDStaff overcomplicated animations](https://qdstaff.com/the-hidden-performance-killer-in-games/)

---

## Google Stitch (新势力, Gemini 3 Pro 加持, 2026 主推)

### 定位 [1][2][3]
- Google 的 AI UI 生成器, Gemini 3.1 Pro/Flash 模型
- **官网**: stitch.withgoogle.com
- 输入：文本 / 草图 / 截图 / 语音
- "**Voice Canvas**" 模式：实时语音改设计 ("Increase font size")
- 主要走 Material Design 路线 + 主题自定义 (色彩/字体/响应式)

### 输出 [1][2]
- Figma 直接导出（仅 Standard Mode 支持），含 auto-layout、图层组、color variables
- 代码：**HTML/CSS (Tailwind)** + **JSX**
- 注：Sketch-to-UI / Voice 这两个实验性模式暂不支持 Figma 导出

### 2026 新特性
- **Vibe Design**: 抽象风格 prompt ("cyberpunk aesthetic") 生成多个变体 [1]
- 多屏原型连接交互流 [1]
- Smarter Design Agent: 识别布局上下文给 spacing/mobile 建议 [1]

### 价格
- **完全免费**：每天 400 credits（约月 12,450）[1]
- 没有付费档（截至 2026.5）[3]

### 翻车
- 输出常需设计师再修改 [1]
- 无团队协作功能 [1]
- 复杂项目 credits 消耗不可预测 [1]
- 国内访问需代理（withgoogle.com 域）

---

## Replit Agent 3 (200分钟自治长链路)

### 关键能力 [4][5][6]
- 单次 autonomous run 上限 **200 分钟**
- 自带浏览器验证系统：模拟 click / form submit / animation 验证（"animation testing"）
- 内置 Figma 导入 + visual editor
- 用 "dynamic intelligence" 帧级优化动画渲染
- 支持 Three.js / react-three-fiber 项目导入

### 输出
- 完整 Web 应用（前端 + 后端 + DB + 部署 + 测试一条龙）[4]
- 直接发布到 Replit 子域或 custom domain [4]

### 价格 (2026 模型)
- Pro: ~$15/mo (低端)，更高档面向企业 [Replit 公开页]
- 国内访问需代理

### 适合
- 需要 AI 长时间自动 iterate 大项目（不只是组件）
- 不太适合"我只想快速出一个好看的 landing page"——overkill

---

## Same.new (一键克隆现有网站)

[10]

- 输入 URL/截图，输出 React/Vue/vanilla JS
- 主打"pixel-perfect cloning"
- 自动部署 Netlify
- **致命弱点**: 复杂项目不稳定、credits 消耗效率低、edits 容易破坏现有结构
- 适合：纯前端原型、临时复刻参考站

---

## a0.dev (移动 App 优先, 不是网页)

[11]

- 文本→原生 iOS/Android App
- QR 码即时预览到真机
- 内置支付 / 分析 / app store 一键发布
- $20/月起
- **本表中唯一非 Web 工具**，纳入参考是因为常被混淆为 web 生成器

---

## Tempo (React 增强 IDE 类)

[13]

- "60-80% 前端代码 from prompt"
- 设计系统支持 MUI / Chakra / 自定义
- 与现有 React 代码库集成
- 免费档可用
- **不是一句话出网页**，更像 Cursor 在 React 范畴的轻量化 — 与本研究 scope 边缘相关

---

## Mocha (Y Combinator 全栈 no-code)

[12]

- 自然语言 → DB / auth / 支付 / 部署一条龙
- 类 Lovable 但更"无代码"，输出代码 ownership 弱
- 适合非技术创始人验证 SaaS 想法
- 视觉调性：与 Lovable 类似，"production-ready" 但不算惊艳

---

## 21st.dev Magic MCP — 不是独立网页生成器

[7]

- 通过 MCP 协议给 IDE (Cursor/Windsurf/VSCode-Cline) 接入组件生成
- 用 `/ui` 命令 → 生成 shadcn 风格 React 组件
- 针对**已有项目里加组件**的场景，不是从零生成网站
- **本研究 scope 边缘** — 用户明确说不要 IDE 嵌入类，但作为 shadcn 生态扩展提一笔

---

## TweakCN — shadcn 主题 AI 编辑器

[8]

- 自然语言改主题：颜色 / 圆角 / 字号 / shadow
- 一键导出 CSS 变量到 shadcn 项目
- 适合：v0 生成 shadcn 后想"换皮肤"
- 免费

---

## 动画能力深度（按场景）

### 入场 / scroll-triggered 动画
| 工具 | 默认输出 | 备注 |
|---|---|---|
| **Framer** | scroll-triggered fades, parallax 默认带; AI 模块 Wireframer/Workshop 自动生成 | 视觉天花板, 但只在 framer.site 跑 |
| **v0** | shadcn 自带 Tailwind + Framer Motion utilities (有 motion-safe variants) | 需要 prompt 明确要求才会写; 默认 CSS 居多 |
| **Lovable** | Tailwind transitions + 偶尔 Lottie/SVG; 有 Plan Mode 配合 | 物理动画需手动 |
| **Bolt** | Claude 写啥就是啥; CSS / 偶尔 Framer Motion | WebContainer 限制 GPU |
| **Readdy** | CSS hover / glow / scale; **未提及任何 JS 动画库** | 复杂动效要自己写 [10 in framer-readdy note] |
| **Trae SOLO** | "粉色渐变 + 飘落爱心动画" 这类提示能输出 CSS keyframes | 复杂物理仍弱 |
| **Claude Artifacts** | SVG / HTML / CSS / JS 动画 + 数据可视化时间轴 / 思维图; Three.js / GSAP 在明确 prompt 下能写但一次过率低 | 单文件限制 |
| **Stitch** | Material 风格的 motion 默认 (transition / ripple) | Vibe Design 多变体 |

### 复杂 3D / 物理 / 粒子
- **没有**任何"一句话出 Three.js demo"做得好的工具
- DeepSite V2 在 prompt 明确时能 12 秒出 Three.js 旋转地球——是个例外亮点
- Claude Artifacts 是写 3D / GSAP 代码的最强工具，但要多轮 debug；不会自己 profile 性能 [14][16]

### Framer Motion 性能陷阱（即使工具帮你写也要警惕）[15]
- FLIP 技术在 layout 动画上 overhead 大
- gesture 动画跑 main thread (vs CSS-native)
- `AnimatePresence` 与 conditional render 的边界条件 fragile
- spring physics 一次过率低 — 需手动 tune mass/stiffness/damping

### GSAP 翻车清单（AI 生成代码常见问题）
- 动画 `top/left` 而不是 `transform` → layout thrashing
- 无脑加 `will-change` → 内存膨胀
- Timeline chain 锚点没设好 → 复杂序列错乱

### CSS @keyframes 翻车（v0 等的常见缺陷）
- 复杂 DOM 没加 `will-change: transform` → 掉帧
- 没有 `requestAnimationFrame` 生命周期管理 → 内存泄漏 [Lovable]
- workaround: `transform: translateZ(0)` 强制 GPU 合成

---

## 配色 / 字体系统现状（残酷真相）

**没有任何工具真的"应用色彩理论 + 字体配对算法"自动选**：

| 工具 | 实际机制 |
|---|---|
| **v0/shadcn 生态** | 给定 12 个调色板 (slate/zinc/stone/gray/neutral/red/rose/orange/green/blue/yellow/violet) + Lucide + 默认字体 — 主题选择题，不是生成 |
| **Lovable 2.0 Visual Edits** | 让你手动改色/字体 (推荐 mycolor.space)；rule-based 限制 ≤3 字体；serif headline + sans body 这类启发式 [9] |
| **Readdy** | prompt 描述 mood 选色；支持点选改色；**无参考图色彩复刻**（review 中无证据）|
| **Framer** | 受 designer 风格驱动，不是自动算法；2026 趋势紧跟 Pantone (Cloud Dancer 米白 + citrus / fuchsia 强调色) |
| **Stitch (Google)** | Material Design 主题系统 (色彩/字体 token) — 本质是模板填空 |
| **Trae SOLO** | 中文 prompt 理解准；色彩是 LLM "审美" 直觉，不是色彩学 |
| **TweakCN** | AI 改主题但仍是 shadcn 12 板范围内 |

### "复刻参考图配色"能力对比
| 工具 | 是否支持 |
|---|---|
| Lovable | ✅ 上传截图 / PDF, 多模态分析提取色 layout [Lovable docs] |
| v0 | ✅ Figma 导入 + image-to-code |
| Readdy | ✅ 上传参考图复刻 (实测中按钮纹理接近原设计) |
| Trae | ✅ 截图标注 |
| Stitch | ✅ image-to-UI 模式 |
| Framer | ❌ 主要靠 prompt 描述 |
| Bolt | ❌ 无 image-to-code |
| Claude Artifacts | ✅ 支持上传图片 (Claude 多模态)，但需要明确 prompt |

### 字体配对自动化
- **真自动**: 没有
- **半自动 / 启发式**: Lovable 2.0 (≤3 字体 + serif/sans 对比规则) [9]
- **手动 prompt**: 大多数其他工具
- **完全跟 design system**: shadcn 默认 (Inter / system font) — 谁用 v0 都长一样

---

## "我只想快速出好看的网页"决策树（基于 2026.5 实测）

```
Q: 想要的是 "可以看可以分享" 还是 "我要 React 代码"?

├─ 只要能看
│   ├─ 视觉天花板（动画/电影感）
│   │   └─ Framer (Hobby 免费有 watermark; Mini $5/年付)
│   ├─ 出海营销页 / 国风 / 苹果风
│   │   └─ Readdy (Free 500 credits / Pro $40)
│   ├─ 国内直连免费
│   │   └─ Trae SOLO 中国版 (个人永久免费)
│   └─ Claude Pro 用户已经有的能力
│       └─ Claude Artifacts (单文件限制, 但 React/HTML/SVG 都能跑)
│
├─ 要 React 代码 + 高质量组件
│   ├─ 一句话出组件
│   │   └─ v0 ($20/mo, shadcn 默认)
│   ├─ 全栈 + 数据库
│   │   └─ Lovable ($25/mo, 最快出 MVP)
│   └─ 想要代码完全 ownership
│       └─ Bolt.new ($20/mo, 但 token 易爆)
│
└─ 想要克隆某个网站 / 截图复刻
    ├─ 闭源好用
    │   └─ Same.new
    └─ 开源零成本
        └─ abi/screenshot-to-code (33.8k stars, 自带 GPT-4 Vision key 跑)
```

---

## 国内可访问性矩阵 (2026.5)

| 工具 | 直连 | 国内支付 | 备注 |
|---|---|---|---|
| Trae SOLO 中国版 | ✅ | ✅ | 字节，毫秒级响应；个人永久免费 |
| MasterGo / 摹客 / 即时设计 / Pixso | ✅ | ✅ | 偏 Figma 替代，能转代码 |
| DeepSite V2 (HF) | ⚠️ | — | HF 域不稳; 走 mirror 站可解 |
| Claude Artifacts | ❌ | ❌ | 需 Claude Pro + 代理 |
| ChatGPT Canvas | ❌ | ❌ | 需 ChatGPT Plus + 代理 |
| Gemini Canvas / Stitch | ❌ | ❌ | 需 Google 账号 + 代理 |
| v0 | ❌ | 有些工具中转 | Vercel 域可达但生成 API 卡 |
| Lovable | ❌ | 有些工具中转 | |
| Bolt.new | ❌ | 有些工具中转 | StackBlitz 域稳定 |
| Framer | ⚠️ | 信用卡需国际版 | 域可访问但站点编辑器卡 |
| Readdy | ⚠️ | 信用卡需国际版 | 出海产品但有华人友好支付路径 |
| Subframe / Magic Patterns / Same.new / Mocha / Tempo | ❌ | ❌ | 需代理 |

---

## 终极推荐 (基于用户的 4 个偏好)

用户偏好：
- (a) 选型 → 直接拿来用
- "好看的网页" + 动画 + 配色
- "直接使用"（不是嵌开发流）
- 与项目无关（个人兴趣）

### Top 3 个人选型 (2026.5)

#### 🥇 **Framer (Mini 档 $5 年付)**
- 视觉天花板，动画原生
- "直接拿来用"——发布即可分享
- 不导出代码？无所谓——你不打算改代码
- 国内可访问，问题在于编辑器有时卡
- 缺点：不能跑后端逻辑

#### 🥈 **Trae SOLO 中国版 (永久免费)**
- 国内直连零成本
- 中文 prompt 理解最强
- 出 HTML/CSS/JS 文件可直接部署
- 缺点：复杂动画/像素级精度仍弱

#### 🥉 **Claude Artifacts (Claude Pro 副产品)**
- 你已经有 Claude Pro 的话，零边际成本
- React / HTML / Three.js / GSAP 都能写
- 缺点：单文件限制，复杂动画需多轮迭代，国内访问要代理

### 按场景的次优推荐
- **想要电商/营销页**：Readdy
- **想要 shadcn 极简组件**：v0
- **想要全栈 MVP**：Lovable
- **想要完全免费 + 创意 demo**：DeepSite V2
- **设计稿 → 代码**：Trae SOLO 或 v0
- **截图克隆**：Same.new 或开源 screenshot-to-code
