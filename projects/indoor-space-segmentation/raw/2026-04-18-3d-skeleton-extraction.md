# 3D Skeleton Extraction for Indoor Buildings — Methods Survey

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q3

---

## Method 1: Isthmus-Based Parallel Thinning
- Parallel and symmetric homotopic thinning
- Detects "isthmuses" (parts locally resembling curves/surfaces)
- Preserves salient geometric features
- Produces either curvilinear or surface skeletons
- "Isthmus persistence" for noise robustness
- Source: ScienceDirect S1524070315000181

## Method 2: Subfield Thinning
- Divides voxel grid into 8 subfields activated sequentially
- Preserves topology while removing simple points
- Can generate curve/surface skeletons or topological kernels
- Source: ResearchGate 253809374

## Method 3: GINIT (BIM-based)
- Automatically generates 3D navigation networks from BIM data
- Projects 3D forms to 2D, thins image to extract paths, restores 3D routes
- Works with irregular shapes using slab/door semantics
- Source: MDPI 2220-9964/12/6/231

## Method 4: Voronoi-Based Skeleton
- Divide-and-conquer: slice 3D into 2D images
- Extract medial axes using Voronoi diagrams per slice
- Reconstruct 3D skeleton from 2D slices
- Optional interpolation for connectivity and smoothing

## Application to Corridor Detection
1. Apply thinning to 3D empty-space voxel grid
2. Result = 1-voxel-wide skeleton through all spaces
3. Skeleton in rooms: short branches meeting at a center
4. Skeleton in corridors: long linear paths
5. Classify skeleton segments by length and linearity
6. Map back to original voxels via wavefront from skeleton

## Key Insight
The skeleton of the free space naturally encodes the topology:
- Branch points = room connections / corridor junctions
- Long linear segments = corridors
- Star-shaped subgraphs = rooms
- Leaf endpoints = dead-end rooms
