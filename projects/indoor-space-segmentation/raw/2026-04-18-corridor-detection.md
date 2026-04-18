# Corridor Detection Methods — Synthesis

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q3

---

## Method 1: Morphological Skeleton + Aspect Ratio

### Skeletonization (Zhang-Suen or Morphological)
1. Iterative thinning of binary image until 1-pixel wide skeleton remains
2. In OpenCV: repeated erosion + opening operations, accumulate differences
3. Skeleton preserves topology while simplifying structure

### Aspect Ratio Analysis
1. Segment floor plan into distinct regions
2. Calculate minimum bounding rectangle for each region
3. Aspect ratio = longer side / shorter side
4. Regions with aspect ratio > 3:1 = potential corridors

### Combined Pipeline
Preprocessing → Region segmentation → Skeleton extraction → Aspect ratio filtering → Geometric regularity checks (parallel boundaries)

## Method 2: Morphological Erosion-Based
From Frías et al.:
- Corridors typically have narrow cross-sections similar to doorways
- Multi-scale erosion: small structuring elements preserve corridors, large ones erase them
- Difference between erosion results at different scales identifies corridor regions

## Method 3: Distance Transform + Topology
- Corridors have LOW distance transform maxima (narrow) but LONG connected chains
- Rooms have HIGH distance transform maxima (wide) but compact shape
- Classify by: (local DT maximum) × (region compactness) ratio

## Method 4: Connected Graph Analysis
- Build topological graph from segmented regions
- Corridors typically have degree ≥ 3 (connect multiple rooms)
- Rooms typically have degree 1-2 (connect to corridor via 1-2 doors)
- Combined with geometric features for robust classification

## T-Junction and Branching Handling
- Skeleton branch points indicate T-junctions
- Segment corridor at branch points into linear segments
- Each segment gets its own corridor sub-zone
- Branch point becomes a "junction zone"

## Key Metrics for Classification
| Feature | Room | Corridor |
|---------|------|----------|
| Aspect ratio | < 2:1 | > 3:1 |
| Compactness (area/perimeter²) | High (>0.05) | Low (<0.02) |
| DT maximum | High | Low |
| Graph degree | 1-2 | ≥ 3 |
| Cross-section variance | Variable | Low (constant width) |
