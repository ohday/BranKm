# Lovable / Bolt.new / v0 — 2026 Pricing, Output, and Failure Modes

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-05-16
- **Project**: llm-webgen-tools-2026
- **Relevant to**: Q1 (visual aesthetic), Q4 (output formats), Q5 (iteration), Q6 (pricing), Q7 (failure modes)
- **Sources drawn from**:
  1. [V0 vs Bolt.new vs Lovable: AI App Builder Comparison 2025/2026 — NxCode](https://www.nxcode.io/resources/news/v0-vs-bolt-vs-lovable-ai-app-builder-comparison-2025)
  2. [Best AI App Builder 2026: Lovable vs Bolt vs v0 vs Mocha — Mocha](https://getmocha.com/blog/best-ai-app-builder-2026/)
  3. [Lovable vs Bolt.new — Yannick Dangoumba](https://yannickdangoumba.com/lovable-vs-bolt-new-which-ai-builder-is-best/)
  4. [v0 vs Lovable visual editor comparison — wbolt 2026](https://www.wbolt.com/vibe-coding-tools.html)
  5. [Using images in Lovable — Lovable Docs](https://docs.lovable.dev/tips-tricks/using-images)
  6. [No-Code Revolution: Lovable AI vs Bolt.new — LinkedIn](https://www.linkedin.com/pulse/no-code-revolution-how-platforms-like-lovable-ai-boltnew-subir-banik-ko87f)
  7. [v0 design systems docs](https://mvrx.v0.build/docs/design-systems)

---

## v0 (Vercel)

### Stack & visual style
- Generates **React + Tailwind CSS + shadcn/ui** components, optimized for Next.js [1][7]
- Default aesthetic = shadcn defaults: minimalist, flat or "New York" style, subtle shadows, neutral palettes (slate/zinc/stone/gray/neutral) [7]
- Two shadcn style presets: **Default** (flat, larger text, no shadows, spacious) vs **New York** (compact, subtle shadows, smaller radii) [7]
- Palette options built-in: slate, zinc, stone, gray, neutral, red, rose, orange, green, blue, yellow, violet [7]
- Lucide icons + dark/light mode native [7]
- Frontend code quality rated ⭐⭐⭐⭐⭐ (highest of the three) [1]
- **Supports Figma import + image/screenshot-to-code** (unique vs Bolt) [1]

### Pricing 2026 (verbatim)
| Tier | Price | What you get |
|---|---|---|
| Free | $0/mo | $5/mo credits, v0-1.5-md model, up to 200 projects |
| Premium | $20/mo | $20/mo credits |
| Team | $30/user/mo | shared credits, central billing |
| Enterprise | Custom | SSO, priority support |

- Token cost: **v0-1.5-md ~$1.50 in / $7.50 out per 1M tokens**, **v0-1.5-lg ~$7.50 in / $37.50 out per 1M tokens** [1]
- Credits reset monthly; only purchased-credit reloads carry forward [1]
- May 2025 pricing shift unlimited→metered caused community backlash [1]

### Iteration model
- Code-centric: select element, refine via NL prompt at Tailwind class level [4]
- "Conversational prompts to chain edits"; strong context for single component, weaker for cross-file [4]
- No drag-and-drop visual editor [4]
- Recent: direct Figma import with edits syncing to code [4]

### Output
- React component code; deploys to **Vercel only** [1]
- GitHub integration; manual API integration [1]
- "V0 generates beautiful layouts but those screens don't do anything on their own" — frontend only [1]

### Failure modes
- No backend, DB, auth — must integrate Clerk/Auth0/Supabase externally [1]
- Vercel lock-in [1]
- "Unpredictable Costs" — token consumption scales fast [1]
- v0 frontend animation: CSS @keyframes drop frames in complex DOM without `will-change: transform`; no built-in performance profiling [2]
- No backend logic for real-time multiplayer animation sync [2]
- G2 rating: **4.7/5** [1]

### Speed (verbatim from [1])
- UI Component: 30s
- Landing page: 5min
- Full-stack app: N/A
- Cost for simple landing page: $5–10, ~2h

---

## Bolt.new (StackBlitz)

### Stack & visual style
- Uses StackBlitz **WebContainer** technology — Node.js running in the browser [1]
- Powered by **Claude 3.5 Sonnet** (per [1] data; likely upgraded since)
- "Fast, energetic" generated designs; "hacker-like vibe" — speed > polish [3]
- Frontend ⭐⭐⭐⭐, backend ⭐⭐⭐ [1]
- "Vibe-based coding" with smart presets, native responsiveness [3]
- Templates like LaunchKit (devtools), bento grids, hero sections [3]
- **No Figma import or image-to-code** [1]
- Example: Dockit landing page built in hours, "monospace fonts and muted tones" [3]

### Pricing 2026
| Tier | Price | Tokens |
|---|---|---|
| Free | $0 | 150K/day, capped at 1M/month |
| Starter | $20/mo | 10M/mo |
| Pro | $50/mo | 26M/mo |
| Business | $100/mo | 55M/mo |
| Enterprise | $200/mo | 120M/mo |

- Monthly tokens expire; separately purchased reloads carry forward [1]

### Output
- Deploys to **Netlify** or **Vercel** [1]
- GitHub push; custom domains [1]
- No native database — relies on external Supabase setup [1]

### Failure modes (verbatim quotes)
- "I burned through 1.3 million tokens in a single day working on a standard web application" [1]
- "Spent over $1,000 on tokens just to fix code problems once my project grew beyond basic prototypes" [1]
- "Once projects exceed 15-20 components or require custom API integrations, context retention degrades noticeably" [1]
- Cloud-only — no offline mode [1]
- WebContainer has no GPU hardware acceleration controls; struggles with particle systems / GPU-intensive animations; "Prompt size limit exceeded" errors; Netlify OOM deployment failures [2]

### Speed
- UI Component: 2min, Landing page: 10min, Full-stack: 30min [1]
- SaaS MVP estimated: $50–200, ~20h [1]
- G2 rating: **4.5/5** [1]

### Growth
- $0 → $4M ARR in 30 days; $40M ARR by March 2025 [1]

---

## Lovable (formerly GPT Engineer)

### Stack & visual style
- Generates **UI + backend (Supabase) + DB schema + auth + deployment** end-to-end [1]
- Built-in **Stripe** + **Clerk** integration [1]
- Frontend ⭐⭐⭐, backend ⭐⭐⭐⭐ [1]
- Aesthetic: "polished, production-ready UI"; "beautiful UI components"; "smooth interactions"; "calm, wellness-inspired design", "muted earth tones", "bold, expressive aesthetics" all proven via prompt [3]
- Auto-refactors for coherence — better for scaling polished apps [3]
- **Supports Figma import + image upload via drag-drop into chat** [1][5]
- Multimodal file analysis (screenshots, PDFs, design files) for style extraction [5]

### Pricing 2026
| Tier | Price | What |
|---|---|---|
| Free | $0 | 5 daily credits (30/mo), public projects only, GitHub sync, one-click deploy |
| Pro | $25/mo | 100 monthly credits + 5 extra daily, rollover, private projects, custom domains, no Lovable badge |
| Business | (higher) | Pro + 100 extra credits, SSO, AI training opt-out, design templates |
| Student | $12.50/mo | 50% off Pro |

- 2025 bonus: every workspace gets **$25 Cloud + $1 AI per month** even on Free [1]
- Build mode credit cost: 0.5 credit for simple change (e.g., button color); 2+ credits for full app structure [1]
- Chat mode: 1 credit per message [1]

### Iteration model
- **Visual editor with Figma-like canvas, drag-drop, layer reordering, style panels** [4]
- **Real-time sync between visual edits and code** [4]
- **Plan Mode**: AI outlines architecture / DB schema before execution [4]
- "Select element directly from preview and reference in chat" [1]
- Encourages planning-first workflow ("map user journeys before prompting") [3]

### Output
- One-click deployment with instant hosting [1]
- **GitHub sync (NOT full export ⚠️)** [1]
- Custom domains on Pro [1]
- Code export only via GitHub sync, not direct full codebase download [1]

### Failure modes
- Output quality heavily depends on prompt quality [1]
- "I burned about 150 messages just trying to create the layout for the app without achieving my MVP" [1]
- Limited customization vs custom code [1]
- Animation: "Limited to pre-built templates (Lottie/SVG); custom physics-based animations require manual Supabase backend tweaks" [2]
- Failure modes: Supabase RLS misconfiguration, edge function CORS errors, AI-generated code lacking lifecycle management (`requestAnimationFrame`) [2]
- Workaround: force GPU accel via `transform: translateZ(0)`; pre-bake physics [2]

### Speed
- UI Component: 5min, Landing page: 3min, Full-stack: 15min, With DB: 5min [1]
- SaaS MVP cost: $25–100, ~8h (cheaper than Bolt) [1]
- G2 rating: **4.6/5** [1]

### Growth
- $20M ARR in 2 months — "fastest growth in European startup history" [1]

---

## Cross-platform animation truth (combined from [2])

| Platform | Animation defaults | Limitation |
|---|---|---|
| v0 | CSS only via Tailwind, shadcn motion utilities | No backend-driven animation sync; @keyframes drop frames without `will-change` |
| Bolt | Whatever Claude writes; mostly CSS / occasional Framer Motion | WebContainer no GPU hw accel; particle systems struggle |
| Lovable | Templated Lottie/SVG | Custom physics needs manual tweaks; missing lifecycle (no requestAnimationFrame) |

None of the three is described as a "Framer Motion / GSAP first" generator. Framer (the design tool) is the cinematic-animation specialist; v0/Bolt/Lovable are app-building generalists.

---

## Quick decision matrix

| Need | Pick |
|---|---|
| Fastest beautiful **single component** | v0 (30s, shadcn polish) |
| **Full-stack** MVP w/ auth + DB | Lovable (15min full app, $25–100 budget) |
| **Quick prototype** with full freedom | Bolt.new (Claude-powered, code download) |
| **Code ownership** (no platform lock) | Bolt > Lovable (only GitHub sync) > v0 (Vercel lock) |
| **Element-level visual edit** | Lovable (Figma-like canvas) > v0 (prompt-only) |
