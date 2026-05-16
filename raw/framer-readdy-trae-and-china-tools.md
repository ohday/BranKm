# Framer / Readdy / Trae SOLO / 国产 AI 网页生成 — 关键事实

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-05-16
- **Project**: llm-webgen-tools-2026
- **Relevant to**: Q1 (aesthetic), Q2 (animation), Q3 (color/font), Q4 (output), Q6 (pricing), Q8 (China access)
- **Sources drawn from**:
  1. [Framer Guide: Master Modern Web Design in 2026 — Embark Studio](https://embark-studio.com/blog/framer-guide-master-modern-web-design-in-2026)
  2. [Framer AI Review 2026 — Scottmax](https://scottmax.com/?p=69913)
  3. [Framer 2026 Pricing — OnePageLove](https://onepagelove.com/framer-pricing)
  4. [Framer Pricing May 2026 — Hamsterstack](https://hamsterstack.com/pricing/framer/)
  5. [Why Teams Are Leaving Framer for Headless — MigrateWebsite](https://www.migratewebsite.ai/blog/pros-cons-framer-migrate-off-framer)
  6. [Unframer GitHub](https://github.com/fav-devs/unframer)
  7. [Readdy AI Review — Kingy AI](https://kingy.ai/ai/in-depth-review-of-readdy-ai-revolutionizing-website-building-with-conversational-ai/)
  8. [Readdy AI 网易实测](https://m.163.com/news/article/K3EOHJ5R0556BKW5.html)
  9. [Readdy 知乎实测 zhuanlan](https://zhuanlan.zhihu.com/p/1962227437065699346)
  10. [Readdy AI Review Pricing — findmyaitool](https://findmyaitool.io/tool/readdy-ai)
  11. [TRAE SOLO 中国版正式发布 — 掘金](https://article.juejin.cn/post/7577583653728763942)
  12. [一张草图变网页 字节 TRAE SOLO 实测 — 腾讯](https://news.qq.com/rain/a/20251114A03YNE00)
  13. [Trae SOLO 深度评测 — 掘金](https://article.juejin.cn/post/7633783248478945316)
  14. [Trae 官网](https://www.trae.com.cn/)
  15. [国内3款AI生成UI工具深度体验 — 掘金](https://juejin.cn/post/7485669029206884367)
  16. [DeepSite V2 体验 — 掘金](https://juejin.cn/post/7518298983116029967)
  17. [Subframe AI Review — 攻壳智能体](https://gongke.net/tools/subframe)
  18. [2026 AIGC 设计工具排行榜 — 墨刀](https://edge.modao.cc/ad/blog/8-AIGC-design-tools.html)
  19. [Galileo AI Review — Toutiao](https://www.toutiao.com/article/7628867766266659343/)
  20. [Magic Patterns Review — findmyaitool](https://findmyaitool.io/tool/magic-patterns)

---

## Framer (AI 视觉天花板, 但 walled garden)

### 美学 DNA
- 2026 主打 "cinematic / immersive / high-end" 视觉 [1][2]
- 标志性视觉元素：Apple Liquid Glass 风格、frosted cards、metal textures、Dithered Waves 组件 (鼠标响应着色器波纹)、暗色为主 [1]
- AI 模块：
  - **Wireframer AI** — 由 prompt 生成响应式 layout，自带 scroll-triggered fades 和 parallax (e.g., "cyberpunk portfolio with neon scroll effects") [1]
  - **Workshop AI** — 自动 hover 3D 旋转、kinetic typography，无需写代码 [1]
  - **Vectors 2.0** — 原生 SVG 动画，替代 After Effects 的部分场景 [1]
- AI 自动适配移动端 performance-safe 模式，省 ~35% 手动响应式调试 [1]

### 价格 (2026.5)
| Plan | 月费 | Domain | Pages | 月访问量 |
|---|---|---|---|---|
| Hobby (free) | $0 | ❌ framer.site 子域 + watermark | 2-3 | 1,000 |
| Mini | $5 (年付) / $10 (月付) | ✅ 1 个 | 1-2 / 1 站 | 2,000 |
| Basic | $10 (年付) / $20 (月付) | ✅ 含 .com 年付 | 150 | 10,000 |
| Pro | $30 | ✅ 多个 | 无限 | 100,000 |

[3][4]

### 致命限制：导出
- **不能导出干净的 React 代码**：输出是 "Div Soup" — absolute positioning + 大量 wrapper [5]
- **必须托管在 Framer 平台**：停付费 → 站点消失。无法部署到 AWS / Vercel / Netlify [5]
- HTML/CSS 导出缺语义标签 (`<section>` 等)、用绝对定位代替 Flexbox [5]
- 无 headless 架构、无 programmatic SEO、无 CMS 深度 [5]
- 第三方 workaround: **Unframer** 把 Framer 组件导成 React 文件，但需要 React 19、动画 / strict mode 受限 [6]

### 翻车场景
- 不导代码 → 不适合需要 code ownership 的人
- 不适合大型 CMS 内容站、复杂动态站
- 中国访问需代理

---

## Readdy.ai (蓝湖团队 · 视觉精美 · 出海产品)

### 出身 & 美学
- **蓝湖原班人马** 出海 4 个月 ARR ~$5M [8]
- Product Hunt 榜首 [搜狐参考]
- 实测中文国风营销页表现突出: "字体、色彩、留白和整体氛围上高度贴合国风审美"，水墨色调、传统版式、书法字体 [9]
- 苹果风、营销页、Landing 页特别擅长

### 输出 & 集成
- 主输出 **HTML/CSS/JS** (default 推荐)；说明也提到 **React/Vue 代码导出** [9][10]
- **Figma 文件导出** ✅ (Starter/Pro 计划) [10]
- 自动 **响应式适配** [9]
- 不限于 SPA，可继续生成子页面保持风格一致 [9]
- 支持自定义域名 hosting，DNS 配置引导 [7]
- 支持 GitHub Copilot 集成 [10]

### 编辑模型
- **Point-to-point editing**: 点任意元素直接改 (image, button, text) — 类 v0/Lovable 但更细粒度 [7]
- 局部 prompt: 选区改配色 (红渐变 → 靛青)、替换图文、加纹样 [9]
- **每次编辑保存为新版本**，可对比、回滚 [7]
- 支持上传参考图复刻 (复刻抖音国风活动页测试中按钮、纹理接近原设计) [9]

### 动画能力
- 实测动画偏 CSS hover 效果 (按钮 glow、scale)、暗色切换 [7]
- **未提及 Framer Motion / GSAP / Lottie**
- 复杂动效 (水墨晕染、卷轴展开) **需手写代码** [9]

### 价格 2026
| Tier | 价格 | Credits | 项目数 |
|---|---|---|---|
| Free | $0 | 500 / 月 | 2 active；约 10 页生成 (50 credits/page) |
| Starter | ~$20–25/月 | 5,000 / 月 | 10 |
| Pro | ~$40/月 | 11,000 / 月 | 无限；优先支持 |
| Enterprise | 定制 | — | — |

- **每次新生成 50 credits, 每次编辑 10 credits** [10]
- 折扣组购可低至 $3.99/月 [10]

### 翻车 / 局限
- 国风 / 传统素材依赖外部上传 (内置素材库偏通用) [9]
- 古风 prompt (如"飞白效果") 经常得多次调整 [9]
- 后端逻辑需手动开发 [9]
- SEO 控制弱 [7]
- 抽象 / nuanced prompt 仍然吃力 [7]
- 国内能用，但官网在国外，访问通常需代理

---

## Trae SOLO (字节 · 国内首选 · 完全免费)

### 定位
- 字节 AI 原生 IDE, **SOLO 模式** 端到端：需求分析 → 代码生成 → 调试 → 部署 [11][13]
- 国内版个人**永久免费**，企业版 ¥69 元/人/月 [11]
- 国际版 Pro 首月 $3，后续 $10/月 [11]
- 国内节点毫秒级响应，无需翻墙 [11]

### 模型 (国产)
- 内置 **豆包 1.5-pro** + **DeepSeek R1/V3** [11][13]
- 支持模型间切换；针对中文语境 + 国内技术栈 (Spring Boot/Vue/鸿蒙/Taro/Ant Design Pro) 优化 [11]
- **中文注释准确率 98.7%** [11] (字节官方数据，需打折)

### 网页生成能力
- 自然语言 → HTML/CSS/JS [12][13]
- 支持上传 **Figma / Sketch 设计稿** 转代码 [11]
- 支持上传 **PDF / Word** 提取内容生成网页 [13]
- 截图标注修改 ("看图写代码") [12]
- 实时预览，自然语言迭代修改 [12]

### 实测分数
- 简单网页 (简历、倒计时、纪念页): **8.5/10**, 4-10 分钟 [13]
- 复杂交互 (电商前端): 6.5/10, 后端逻辑常用 mock data [13]
- 90%+ 设计稿元素准确转码；UI 错位修复成功率约 70% [13]
- 500+ 行项目架构可能混乱 [13]

### 翻车
- **2026.4 独立端**暂不支持代码编辑，需回退 IDE 版 [13]
- 复杂全栈生产项目仍需人工补 [13]
- 像素级 UI 精度场景吃力 [13]

---

## DeepSite V2 (HuggingFace 上的 DeepSeek 加持版, 完全免费)

- 主页 `deepsite.hf.co/projects/new`，零配置 [16]
- 模型: **DeepSeek R1-0528** [16]
- "左指令右预览" 双栏，**Diff Patching** 增量更新代码 (~3s) [16]
- 自动引入 TailwindCSS + Font Awesome [16]
- 8 秒生成响应式咖啡店官网；12 秒 Three.js 旋转地球 [16]
- HTML 文件直接导出，可部署到 Nginx [16]
- **完全免费、开源、无订阅** [16]
- 评分 4.5/5 (实测口径)

### 局限
- 复杂物理引擎/游戏逻辑仍需人工优化 [16]
- 首次生成 prompt 偏差较大，需多次迭代

---

## 国内 AI 设计稿/网页一体化工具

| 工具 | 出身 | 核心优势 |
|---|---|---|
| **MasterGo (莫高)** | 字节系 | AI 文生设计稿 + MCP 服务转 HTML/Vue/React；500人协同 [15][18] |
| **摹客 (Mockplus)** | 老牌 | "小摹 AI" 多页设计稿；离线全流程 (国内唯一)；国风素材库 [15] |
| **即时设计 (JsDesign)** | 国产 Figma 替代 | 一句话生成 UI；标注/切图自动；CSS/React 导出 [15] |
| **Pixso** | 万兴 | 鸿蒙生态适配；变量化设计系统 [18] |
| **Motiff** | 猿辅导 | AI 复制/布局；组件库管理强 [18] |
| **Ardot** | 腾讯 | 文生 UI + 动态布局 [18] |
| **稿定 AI** | 稿定 | 抖音/淘宝/小红书 平台规范自动适配；营销物料 [18] |

(国内 AI 设计工具更偏 Figma 模式，Readdy / Trae 才是直接出网页的)

---

## Subframe (设计 + 代码同源)

- **Code-native UI design** — 设计师和开发用同一组 React/Tailwind 组件 [17]
- 拖拽 + AI prompt 生成；导出 React/Tailwind code 或 CLI 部署 [17]
- 生成的代码号称 **pixel-perfect** [17]
- 支持自定义组件并同步到代码 [17]

### 价格
- Free: $0/editor/month, 1 项目 max 5 pages, 不限协作者, 受限 AI, 24h 历史
- Pro: **$29/editor/month** (无限 AI, 高级特性)

### 适合
- 团队 (设计-开发对齐)；初创快速交付 UI
- 个人用户：免费档够探索

---

## Galileo AI (设计稿生成器, 现在偏 mobile)

- 文本/草图 → 高保真 Figma 设计稿 [19]
- 30-60 秒出图，专长：mobile App UI、营销页 [19]
- 输出 Figma frame，含 auto-layout 图层 [19]
- **不支持代码导出，不支持多页原型** [19]
- 适合"灵感探索"，不适合"直接交付可用网页"
- (注：Galileo 已被 Google 收购整合进 Stitch / Gemini Canvas 系列；自身网站不再独立活跃)

---

## Magic Patterns

- AI 文本/截图 → React + Tailwind + shadcn 组件 [20]
- 支持导入现有品牌组件库做 design-system matching [20]
- 多人协作 canvas
- 直接导出 Figma / React code
- 起价 **$19/月** (含 free trial) [20]
- 偏 B2B / 企业 brand 一致性场景

---

## 截图/克隆类开源项目 (零成本路径)

| 项目 | Star | 输出 | 模型 |
|---|---|---|---|
| **abi/screenshot-to-code** | 33.8k | HTML/Tailwind, React, Vue, Bootstrap | GPT-4 Vision + DALL-E 3 |
| **JCodesMore/ai-website-cloner-template** | 3.6k | Next.js 16 + Tailwind v4 + TS | Claude Code agent |
| **ailinuxok/ScreenCoder** | — | HTML/CSS modular | 多模型 (Doubao/GPT/Gemini) |
| **skumar54uncc/aiwebcloner** | — | Tailwind + Playwright (动态页) | 通用 |

- 最大 ROI: 自带 OpenAI key, 本地跑 Screenshot-to-Code, 一键克隆 + 修改

---

## 关键 trade-off 速记

| 维度 | 视觉强 → 弱 |
|---|---|
| 电影感 / 动画 / 营销页 | Framer > Readdy > v0 > Lovable > Bolt |
| 代码 ownership | screenshot-to-code (本地) / Bolt > v0 > Lovable >> Framer (无) |
| 中国直连可用 | Trae SOLO / DeepSite (HF 经常波动) > 其他都需代理 |
| 免费可用度 | Trae (永久免费) > DeepSite > v0 ($5/mo credits) > Lovable (5/天) > Readdy (500 credits) > Framer (有 watermark) |
| 一句话出网站 | Lovable > Readdy > Trae > Bolt > v0 (更适合组件) |
| 设计稿/参考图复刻 | v0 (Figma+image) ≈ Readdy ≈ Lovable ≈ Trae > Framer > Bolt (无) |
