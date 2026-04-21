## [2026-04-21] explore | Topic defined

Pivoted from "segmentation → portal" to acoustics-first "direct portal
detection + probe graph + UTD diffraction". 9 key questions locked.
5 existing raw sources flagged as reusable.

## [2026-04-21] explore | Brief revised

User clarified:
- Portal auto-detection is THE core problem
- Diffraction must be a dedicated focus (not buried in edge weights)
- Acoustic model: Scheme A (geometric + UTD). Project Acoustics = contrast only
- Steam Audio depth: SOURCE-CODE LEVEL (able to modify)
- Input: voxel grid already baked by the user with customizable fields

Key questions expanded to 11 (8 must + 1 should + 2 stretch),
reorganized into 3 groups: portal-detect (Q2-Q6), diffraction (Q7-Q9),
industrial (Q10-Q11).

## [2026-04-21] explore | Steam Audio repo pinned

User pinned the Steam Audio repo URL:
https://github.com/ValveSoftware/steam-audio/tree/master
Brief Q10 updated with the repo URL and concrete file/module targets
under `core/src/core/` to guide km-websearch directly into source.

## [2026-04-21] websearch | Research complete — 6 new raw sources

Fetched source-code-level details from Steam Audio's open-source SDK
(18 files under core/src/core/) plus supporting literature on UTD,
Tsingos 2001 beam tracing, Project Acoustics/ARD, Raghuvanshi 2021
dynamic portal occlusion, and morphological/persistence-based portal
detection.

### New raw files
- 20. steam-audio-pathing-source-breakdown.md     [Q1,Q5-7,Q9-10]
- 21. steam-audio-probe-placement.md              [Q5,Q6]
- 22. utd-diffraction-steam-audio-vs-tsingos.md   [Q7,Q8,Q9]
- 23. project-acoustics-wave-based-contrast.md    [Q7,Q11]
- 24. portal-detection-methods-acoustic.md        [Q1-Q4]
- 25. runtime-acoustic-path-query-architecture.md [Q6,Q9]

### Completeness per key question

Q1  Portal theory + why skip segmentation          ✅ complete
    WHAT + WHY + HOW + DEPTH + TRADE + CROSS + LINK
    - Source-code evidence that Steam Audio encodes "portal" implicitly
      in probe visibility graph - NO explicit portal concept
    - Visual portal baseline from existing raw #18 (cells-portals-bsp)
    - Three-source cross-validation (Steam Audio, Tsingos, Raghuvanshi)

Q2  Portal candidate extraction from voxel/SDF     ✅ complete
    WHAT + WHY + HOW + DEPTH
    - 8 methods compared (morphological, SDF saddle, min-cut,
      persistence, Reeb, skeleton, AO saddles, cross-section)
    - Leverages existing raw #8, #9, #13, #17 with acoustic reinterpretation

Q3  Saliency filtering                             ✅ complete
    WHAT + WHY + HOW + DEPTH
    - Persistence lifetime ranking from shared raw #8
    - Multi-scale morphological opening sweep strategy
    - Wavelength-relative threshold rules (Fresnel zone)

Q4  Portal parameterization                        ✅ complete
    WHAT + WHY + HOW
    - Documented as part of portal-detection survey
    - Per-portal: plane, polygon, centroid, normal, edge points, thickness
    - Steam Audio's counter-evidence: no parameters needed if going graph-only

Q5  Probe placement                                ✅ complete
    WHAT + WHY + HOW + DEPTH + TRADE
    - Full Steam Audio algorithm with pseudocode
    - Octree mode is stubbed-out in OSS release (verified)
    - Trade-offs vs adaptive/Poisson-disk discussed

Q6  Graph connection                               ✅ complete
    WHAT + WHY + HOW + DEPTH + TRADE
    - Full visibility test implementation (single/volumetric modes)
    - Edge cost = Euclidean distance, symmetrization details
    - Asymmetric-range for multi-story buildings

Q7  UTD theory                                     ✅ complete
    WHAT + WHY + HOW + DEPTH + TRADE
    - Keller cone + UTD derivation at 1-page depth
    - Full coefficient formula with 4 cot terms + Fresnel transition
    - Frequency dependence |D|~1/sqrt(f)
    - BTM alternative noted as too expensive for games

Q8  Offline diffraction bake                       ✅ complete
    WHAT + WHY + HOW + DEPTH
    - Steam Audio stores NO per-edge data - only SoundPath deviation
    - Storage analysis for 1000 probes
    - Alternative: per-edge UTD table (Tsingos style) with storage scaling

Q9  Runtime diffraction                            ✅ complete
    WHAT + WHY + HOW + DEPTH + TRADE
    - Full runtime pipeline with Python-style pseudocode
    - Path lookup O(1), A* fallback for dynamic
    - Funnel algorithm NOT used - discussed why
    - Multi-path aggregation as incoherent energy sum

Q10 Steam Audio source-code dissection             ✅ complete
    WHAT + WHY + HOW + DEPTH + CROSS
    - 18 files dissected to function level
    - Every key class/struct documented
    - File table for porting reference
    - Cross-validated against guide.html documentation

Q11 Project Acoustics contrast                     ✅ complete (should-level)
    WHAT + WHY + HOW + TRADE
    - ARD algorithm + parametric encoding covered
    - Trade-off table vs Steam Audio
    - Full-paper extraction blocked by PDF binary issue but abstract +
      secondary sources give enough for contrast depth

### Unresolved / stretch items
- Q12 (freq-banded transmission): not deeply covered; standard impedance
  BC treatment is well-known but not project-critical
- Q13 (dynamic geometry): covered via Raghuvanshi 2021 citation only;
  full mechanistic detail blocked by PDF binary issue

### Total source count: 6 new + 5 reused = 11 sources for this project
### Total pages in shared raw: 25 (was 19)

Ready for km-wiki synthesis.

## [2026-04-21] wiki | Acoustic Portal Baking — Synthesis Complete

- Pages created: 12
- Sources used: 11 (6 new + 5 reused)
- Total Mermaid diagrams: 80+
- Key insight threaded across all pages: Steam Audio uses NO explicit
  portals — probe visibility graph implicitly encodes them, and
  diffraction is approximated by feeding path-bend-angle into UTD with
  dummy wedge params (n=2, L=0.05m).

### Page list
1. 核心洞察：声学不需要显式 Portal
2. 从体素到探针图：完整流水线
3. 探针自动布置
4. 可见性图构建
5. 烘焙阶段：Dijkstra 全对最短路
6. SoundPath 存储结构
7. UTD 绕射理论
8. Steam Audio 的偏折角-UTD 近似
9. 运行时查询与 DSP
10. 显式 Portal 检测方法
11. Project Acoustics 波动式对比
12. 方法对比与原型建议

### Coverage
All 9 must-priority key questions + 1 should + 2 stretch addressed.
Stretch items Q12 (freq-banded transmission) and Q13 (dynamic geometry)
are lightly covered; noted as known gaps in index.md.

## [2026-04-21] wiki | Acoustic Portal Baking — Synthesis Complete

- Pages created: 12
- Sources used: 11 (6 new + 5 reused)
- Total Mermaid diagrams: 80+
- Key insight threaded across all pages: Steam Audio uses NO explicit
  portals — probe visibility graph implicitly encodes them, and
  diffraction is approximated by feeding path-bend-angle into UTD with
  dummy wedge params (n=2, L=0.05m).

### Page list
1. 核心洞察：声学不需要显式 Portal
2. 从体素到探针图：完整流水线
3. 探针自动布置
4. 可见性图构建
5. 烘焙阶段：Dijkstra 全对最短路
6. SoundPath 存储结构
7. UTD 绕射理论
8. Steam Audio 的偏折角-UTD 近似
9. 运行时查询与 DSP
10. 显式 Portal 检测方法
11. Project Acoustics 波动式对比
12. 方法对比与原型建议

### Coverage
All 9 must-priority key questions + 1 should + 2 stretch addressed.
Stretch items Q12 (freq-banded transmission) and Q13 (dynamic geometry)
are lightly covered; noted as known gaps in index.md.
