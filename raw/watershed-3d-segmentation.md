# Watershed 3D Distance Field Room Segmentation — Synthesized Overview

- **URL**: Multiple search results synthesized
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q2

---

## Complete Pipeline

### Step 1: 3D Euclidean Distance Transform (EDT)
- Input: 3D occupancy grid (voxel: free/occupied)
- Each free voxel stores distance to nearest occupied voxel (wall)
- Higher values = farther from obstacles = room centers
- Can be optimized with wavefront propagation or VDB structures

### Step 2: Seed Generation
Two main approaches:
1. **Sphere Packing**: Fill thresholded space with maximal inscribed spheres centered at local maxima of EDT. Each sphere = candidate room seed.
2. **DBSCAN Clustering**: Group spatially proximate thresholded voxels into seed clusters.

### Step 3: Marker-Controlled 3D Watershed
- Seeds serve as markers/labels
- Inverted distance field: peaks become basins
- Watershed lines (room boundaries) form where water from different basins meets
- Marker-controlled approach **prevents over-segmentation** (key advantage)

### Step 4: Post-Processing
- **Topological graph**: Link adjacent seeds to identify over-segmented regions for merging
- **Wavefront growth**: Expand seed regions into unlabeled voxels using 26-connectivity
- **Boundary smoothing**: Morphological operations or graph-cut energy minimization

## Handling Over-Segmentation
- Use markers (derived from distance field local maxima) instead of raw watershed
- Merge nearby local maxima before seeding
- Post-process: merge regions where boundary cross-section exceeds door-sized threshold

## Marker-Controlled Watershed Details
Traditional watershed produces one region per local minimum → severe over-segmentation.
Solution: replace all local minima with predefined markers. Only marker-seeded basins grow.
Dimensionless parameter α (e.g., α = 0.16) distinguishes true contacts from "necks".

## Advantages
- Handles non-Manhattan geometries
- Efficient with VDB structures
- Marker control constrains region count
