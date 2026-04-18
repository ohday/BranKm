# Recast Navigation Watershed-Based NavMesh Generation

- **URL**: Synthesized from search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q5

---

## Recast Navigation Pipeline (Industry Standard)

### Step 1: Voxelization
- Rasterize all triangle meshes into a 3D heightfield (solid spans + open spans)
- Each column of voxels stores walkable/non-walkable spans
- Voxel size = cell size parameter (typically 0.1-0.3m)

### Step 2: Heightfield Filtering
- Mark walkable voxels based on:
  - Slope: surface angle < maxSlope (typically 45°)
  - Step height: vertical gaps < maxClimb (typically 0.4m)
  - Clearance: ceiling height > agentHeight (typically 2.0m)

### Step 3: Compact Heightfield
- Compress heightfield into span-based representation
- Build connectivity between adjacent walkable spans (4/8-connected)

### Step 4: Region Building (Watershed!)
- **Distance field computation**: Each walkable voxel gets distance to nearest boundary
- **Watershed transform**: Regions grow from distance field maxima
- **Region merging**: Small regions merged with neighbors to prevent over-segmentation
- **Region parameters**: minRegionArea, mergeRegionArea

### Step 5: Contour Generation
- Trace boundaries between regions
- Simplify contours to reduce vertex count

### Step 6: Polygon Mesh
- Triangulate each region contour
- Create navigation mesh from polygon regions

## Key Relevance
Recast's Step 4 is exactly the distance-field + watershed approach applied to indoor-like environments. The same algorithm concepts apply to full 3D room segmentation:
- Distance field ← user already has ray distances
- Watershed ← can be applied directly
- Region merging ← handles over-segmentation

## Parameters (Recast defaults)
| Parameter | Default | Purpose |
|-----------|---------|---------|
| cellSize | 0.3m | Voxel XZ resolution |
| cellHeight | 0.2m | Voxel Y resolution |
| maxSlope | 45° | Walkable surface angle |
| maxClimb | 0.4m | Step height |
| agentHeight | 2.0m | Ceiling clearance |
| agentRadius | 0.6m | Agent collision radius |
| minRegionArea | 8 cells² | Minimum region size |
| mergeRegionArea | 20 cells² | Auto-merge threshold |
