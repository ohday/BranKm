# Academic Survey: Chokepoint/Bottleneck Detection in 3D Voxel Grids

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q2

---

## Method 1: Graph-Based Min-Cut

### Construction
1. Each free voxel = graph node
2. Edges connect 6/26-adjacent free voxels
3. Edge weights encode passage width (e.g., min distance transform value along edge, or harmonic mean of endpoint DT values)

### Min-Cut Detection
- Min-cut between two region seeds identifies the narrowest passage
- Chokepoint = the set of edges in the min-cut
- Max-Flow/Min-Cut theorem: capacity of min-cut = max flow between source and sink

### Practical Approach
1. Over-segment space (watershed, connected components after erosion)
2. Build region adjacency graph (RAG)
3. For each pair of adjacent regions, find the min-cut cross-section
4. Cross-section area < door threshold → mark as doorway
5. Cross-section area > corridor threshold → mark as corridor entrance

## Method 2: Cross-Section Area Analysis

### Algorithm
1. Identify boundary voxels between two adjacent regions
2. Fit a plane to boundary voxels (or use the principal direction of the boundary)
3. Project boundary voxels onto this plane
4. Compute the convex hull area of the projection
5. Compare to thresholds: < door_area → doorway, > corridor_area → open connection

### Parameter Guidance
- Standard single door: ~0.9m × 2.1m = ~1.9 m²
- Double door: ~1.8m × 2.1m = ~3.8 m²
- Corridor entrance: typically > 4 m²
- These can be expressed in voxel counts based on voxel resolution

## Method 3: Distance Field Gradient Analysis

### Concept
- At doorways, the distance field forms a "saddle point" — low value surrounded by higher values in perpendicular directions
- The gradient of the distance field changes direction abruptly at chokepoints

### Detection
1. Compute gradient of 3D distance field
2. Find voxels where:
   - DT value is a local minimum along one direction (doorway width direction)
   - DT value is NOT a minimum along perpendicular directions (room extends both sides)
3. These saddle points = chokepoint candidates
4. Cluster nearby saddle points into single doorway markers

### Advantage
- Works directly on the existing ray-distance data
- No need for separate segmentation first
- Can detect doorways even in unsegmented spaces

## Method 4: Topological Persistence

### Concept
From persistent homology: study how topological features (connected components) appear and disappear as a threshold changes.

### Application to Indoor Spaces
1. Sweep distance threshold from max to 0
2. At high threshold: only room centers survive → many components
3. As threshold decreases: components merge through doorways
4. The threshold at which two components merge = the "bottleneck width"
5. Pairs (birth, death) form a persistence diagram
6. Long-lived features = robust rooms; short-lived = noise
7. Merge events at small cross-section thresholds = doorways

### Advantage
Theoretically clean; automatically identifies all bottlenecks and ranks them by significance.
