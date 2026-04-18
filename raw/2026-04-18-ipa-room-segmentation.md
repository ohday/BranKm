# IPA Room Segmentation Algorithms — Complete Technical Extraction

- **URL**: https://blog.csdn.net/jucat/article/details/138755341
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q2, Q3

---

## 1. Morphological Segmentation

**Algorithm Steps:**
1. **Input:** Occupancy grid map M₁ with two cell types — white (accessible) and black (inaccessible/obstacles).
2. **Iterative erosion:** Perform morphological erosion on accessible regions, eroding one pixel at a time in concentric rings.
3. **Region evaluation after each erosion step:** After each single-pixel erosion pass, examine the resulting connected components. If a separated region's area falls within user-defined upper and lower bounds, label that region as room rᵢ. Copy the original map to M₂. Mark each identified room rᵢ with a distinct color in M₂, and simultaneously mark that room's area as inaccessible in M₁.
4. **Repeat erosion-separation cycle** to extract successive rooms into M₂.
5. **Expansion/flood-fill:** Expand each labeled room region outward until all accessible pixels are assigned.

**Parameters:** Number of erosion iterations, room area min/max thresholds.

## 2. Distance Transform-Based Segmentation

**Algorithm Steps:**
1. **Distance transform computation:** Each accessible pixel stores its Euclidean distance to the nearest boundary pixel.
2. **Local maxima identification:** Distance transform local maxima are located at spatial centers of rooms. In narrow corridors/doorways, local maxima are significantly smaller.
3. **Optimal threshold search (watershed):** Sweep distance threshold t from high to low. At each threshold, extract a set C of disconnected spatial regions. As t decreases, |C| increases. When t equals the local maximum distance at a doorway, two spaces merge and |C| decreases. Seek t* that maximizes |C|.
4. **Label and expand:** Assign labels to discovered discrete spaces, then flood-fill/expand until all accessible pixels are labeled.

Optimal t*=65 yielded maximum of 10 rooms in test case.

## 3. Voronoi Graph-Based Segmentation

**Algorithm Steps:**
1. Compute Generalized Voronoi Diagram (GVD), then prune skeleton by collapsing leaf edges.
2. Identify critical points: Voronoi points with **exactly two** equidistant nearest obstacle pixels. Store in set P.
3. Draw critical lines connecting each critical point to its two nearest obstacle points. Critical lines + walls form enclosed regions. Filter nested enclosures. Compute critical angle. Remove corner angles <90°. Large angles at doorways → retain.
4. Merge Voronoi cells into rooms using five heuristic rules:
   - Rule 1: Cell area < 12.5m², one neighbor, 75% boundary doesn't touch walls → merge
   - Rule 2: Cell area < 2m², neighbor has ≥20% wall boundary → merge
   - Rule 3: Neighbor has ≤2 neighbors AND current cell has ≥50% wall perimeter → merge
   - Rule 4: ≥20% boundary shared with large cell AND ≥10% of large cell's boundary → merge
   - Rule 5: >40% boundary touching neighbor nₘ → merge

## 4. Feature/Semantic-Based Segmentation
Uses 33 geometric features from simulated 360° laser scans + AdaBoost classifier. Requires training data.
