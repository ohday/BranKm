# 3D Morphological Room Segmentation (Frías et al. 2020)

- **URL**: https://isprs-archives.copernicus.org/articles/XLIV-4-W1-2020/49/2020/
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q2, Q3

---

## Algorithm Pipeline

### Step 1: Voxelization
Convert point cloud to regular 3D voxel grid (binary: occupied/empty).

### Step 2: Indoor Empty Space Extraction
Extract indoor empty voxels using CropHull algorithm — identifies voxels within the hull but containing no points.

### Step 3: 3D Morphological Erosion
- Sweep 3D structuring element through binary voxel volume
- Voxel survives only if entire structuring element fits within empty voxels
- **Key**: Doorways/narrow openings have thin cross-sections → erosion severs connectivity
- Structuring element must exceed half the doorway width to work

### Step 4: Connected Components Clustering
3D connected components on surviving voxels. Each cluster = one room's eroded interior.

### Step 5: Individual 3D Morphological Dilation
- Dilation performed **separately per labeled cluster** (prevents re-merging!)
- Recovers room volume lost during erosion

### Step 6: Classification of Occupied Voxels
Unlabeled occupied voxels assigned to rooms by proximity to labeled empty voxels.

## Critical Design Insight
The per-cluster individual dilation is the key that prevents re-merging of adjacent rooms. Global dilation would reconnect everything.

## Parameters
| Parameter | Role |
|-----------|------|
| Voxel size | Spatial resolution |
| Structuring element size | Must exceed half minimum doorway width |
| Connectivity type | Face/edge/vertex neighbors for connected components |

## Limitation
Small rooms may be completely erased by erosion if structuring element is too large.
