## [2026-04-18] explore | Topic defined

- Scope: Deterministic geometric algorithms for indoor space segmentation
- Input: Voxel grid with 26-dir ray distances + collision types + material info
- Output: SubZone partition + door/opening locations + zone type classification
- 5 key questions defined covering: distance field segmentation, chokepoint detection,
  corridor identification, spectral clustering, and academic survey
- Depth: Algorithmic detail sufficient for implementation
- Excluded: ML methods, door/window distinction, UE integration, real-time constraints

## [2026-04-18] websearch | Deterministic Indoor Space Segmentation — Research Complete

### Coverage

| # | Key Question | Status | Sources | Rounds |
|---|-------------|--------|---------|--------|
| 1 | Distance-field-based spatial segmentation (SDF, Watershed, MAT) | ✅ complete | [1,2,3,5,6,7,13,19] | 3 |
| 2 | Chokepoint/bottleneck detection in 3D voxel grids | ✅ complete | [1,2,5,6,7,8,9,15,17,18] | 3 |
| 3 | Corridor detection and classification | ✅ complete | [1,5,10,14,15] | 2 |
| 4 | Spectral clustering for indoor space partitioning | ✅ complete | [4,11,12] | 2 |
| 5 | Academic state-of-the-art survey (2018-2026) | ✅ complete | [2,7,8,15,16,17,18,19] | 3 |

### Completeness Detail

**Q1 — Distance-field segmentation:**
- ✅ WHAT: 5+ methods to convert ray distances to scalar fields for segmentation
- ✅ WHY: Each method's trade-offs clearly documented
- ✅ HOW: Step-by-step algorithms for morphological, watershed, sphere packing, SDF approaches
- ✅ DEPTH: Specific parameters (voxel sizes, thresholds, merging rules)
- ✅ LINK: Direct mapping from user's 26-direction rays to each method's input

**Q2 — Chokepoint detection:**
- ✅ WHAT: 4 distinct methods (min-cut, cross-section, gradient saddle, persistence)
- ✅ WHY: Each serves different precision/robustness trade-off
- ✅ HOW: Detailed algorithms with thresholds (door ~1.9m², double ~3.8m²)
- ✅ DEPTH: Topological persistence provides principled ranking
- ✅ TRADE: Persistence most elegant but requires Gudhi; morphological simplest

**Q3 — Corridor detection:**
- ✅ WHAT: 4 methods (aspect ratio, multi-scale erosion, graph degree, skeleton)
- ✅ HOW: Specific thresholds (aspect >3:1, compactness <0.02, degree ≥3)
- ✅ DEPTH: T-junction handling via skeleton branch points
- ⚠️ CROSS: Limited to 2 sources for skeleton-based approach

**Q4 — Spectral clustering:**
- ✅ WHAT: Full pipeline from affinity matrix to k-means on eigenvectors
- ✅ HOW: 5 affinity options, Nyström algorithm steps, eigengap heuristic
- ✅ DEPTH: O-notation complexity, adaptive rank, landmark selection
- ✅ TRADE: Sparse Laplacian with 26-connectivity efficient; Nyström for extreme scale

**Q5 — Academic survey:**
- ✅ Methods catalogued: Hübner voxel (2020), Sphere packing (2021), VOX2BIM+ (2023), Area Graph (2022), Bobkov anisotropic (2017), Recast (industry), BSP cells-portals, topological persistence
- ✅ Trend: shift from 2D to true 3D; voxel-based dominance; deterministic still competitive
- ✅ TRADE: each method's strengths/weaknesses mapped

### Sources collected: 19 files in raw/
### Total search rounds: 13
### Gaps: None critical. Q3 skeleton-based corridor detection has limited source diversity but method is well-established in morphology literature.

## [2026-04-18] wiki | Deterministic Indoor Space Segmentation — Synthesis Complete

- Pages created: 10
  1. index.md — Knowledge map and entry point
  2. wiki/ray-to-scalar-fields.md — 5 scalar fields from 26-dir rays
  3. wiki/morphological-segmentation.md — Erosion/dilation pipeline (2D & 3D)
  4. wiki/watershed-segmentation.md — Distance field + marker-controlled flooding
  5. wiki/sphere-packing.md — Maximal inscribed spheres + graph topology
  6. wiki/topological-persistence.md — Persistent homology approach
  7. wiki/spectral-clustering.md — Graph Laplacian + Nyström scaling
  8. wiki/chokepoint-detection.md — 4 methods for doorway/opening detection
  9. wiki/corridor-classification.md — Identifying and typing corridors
  10. wiki/method-comparison.md — Decision matrix + failure modes
  11. wiki/academic-survey.md — Literature map 2017–2026
- Sources used: 19 out of 19
- Knowledge map: Pipeline architecture (input → derived fields → 5 method families → output)
- Every key question answered by 1-3 dedicated pages
- Diagram density: 3-5 Mermaid diagrams per page average
