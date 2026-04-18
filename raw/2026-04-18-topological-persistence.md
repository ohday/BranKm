# Topological Persistence for Indoor Space Segmentation

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q2, Q5

---

## Core Concept
Study how connected components of the sublevel set {x : f(x) ≤ t} evolve as threshold t changes. Applied to distance fields of indoor spaces.

## Algorithm: Persistence-Based Bottleneck Detection

### Step 1: Distance Field as Filter Function
- f(v) = distance transform value at voxel v (distance to nearest wall)
- Or: f(v) = -distance (inverted, so rooms are minima)

### Step 2: Sublevel Set Filtration
- Sweep threshold from max to min (or min to max for inverted)
- At high threshold: only room centers are active → many connected components
- As threshold decreases: components grow and merge through doorways

### Step 3: Persistence Diagram
- Each component has (birth, death) pair
  - Birth = threshold when component first appears (= distance at room center)
  - Death = threshold when component merges with another (= bottleneck width)
- Long-lived features (|birth - death| large) = robust rooms
- Short-lived features = noise or minor sub-spaces

### Step 4: Doorway Identification
- Merge events where death threshold is small (= narrow passage) → doorways
- Merge events where death threshold is large → open connections
- The threshold at merge = half the doorway width

### Step 5: Room Segmentation
- Cut the merge tree at a chosen persistence threshold
- Remaining components = rooms
- Merge points below threshold = internal sub-divisions (ignore)
- Merge points above threshold = room boundaries (keep)

## Advantages
- **Principled**: based on algebraic topology, no ad-hoc parameters for number of rooms
- **Multi-scale**: naturally handles rooms of different sizes
- **Ranks bottlenecks**: automatically orders by significance (persistence)
- **Single parameter**: persistence threshold (interpretable as "minimum doorway width to consider")

## Implementation
- Gudhi library: efficient persistent homology computation
- For 1D functions (distance fields), linear-time algorithms exist
- For 3D scalar fields on grids: use cubical complex persistent homology

## Connection to Watershed
The merge tree from persistent homology is essentially the same information as marker-controlled watershed, but with a formal persistence-based selection of markers.

## Relevance to User's Setup
- User has 26-direction ray distances per voxel → can derive a scalar "openness" field
- Openness field = average or min of 26 ray distances → analogous to distance transform
- Apply persistent homology directly to this scalar field
- Merge tree gives room structure; bottleneck thresholds give doorway locations
