# Sphere Packing Method for 3D Room Segmentation (Li et al. 2021)

- **URL**: https://www.mdpi.com/2220-9964/10/11/739
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q2

---

## Core Concept
Pack indoor empty space with maximal inscribed spheres. Rooms naturally become separated connected components of spheres, because doorways are too narrow for large spheres.

## Algorithm Pipeline

### Step 1: Voxelization
- Convert point cloud to volumetric representation using VDB data structure (OpenVDB)
- VDB: sparse hierarchical grid, efficient for large-scale indoor environments
- Stores occupancy (occupied/empty) per voxel

### Step 2: 3D Euclidean Distance Transform
- Compute distance from each empty voxel to nearest occupied voxel
- Local maxima of distance field → centers of maximal inscribed spheres
- These maxima correspond to room centers (largest possible spheres) and corridor centers (smaller spheres)

### Step 3: Sphere Generation
- At each distance field local maximum, generate a sphere with radius = distance value
- Apply minimum radius threshold to filter noise/tiny spaces
- Spheres naturally overlap within rooms, form sparse connections through doorways

### Step 4: Sphere Graph Construction
- Connect overlapping/adjacent spheres into a topological graph
- Nodes = spheres, edges = overlap or adjacency
- Within rooms: dense connectivity (many overlapping large spheres)
- Through doorways: thin connectivity (few small spheres)

### Step 5: Room Identification
- Connected components of spheres above a size threshold → initial room seeds
- Or: find thin connection points in the sphere graph (analogous to graph cut on RAG)
- Each connected component = one room

### Step 6: Wavefront Propagation
- From room seeds, expand to fill all empty voxels
- Priority queue based on distance to seed center
- Stop at room boundaries (where expanding fronts from different rooms meet)

## Key Advantages
- True 3D: handles multi-story, mezzanine, cross-floor spaces
- Non-Manhattan: no wall verticality assumptions
- VDB efficiency: sparse storage for large buildings
- Intuitive: sphere size naturally encodes room scale

## Doorway Detection
Doorways correspond to narrow "necks" in the sphere graph — locations where:
- Sphere radii are small (constrained by doorway width)
- Graph connectivity is low (few spheres bridge two large clusters)
- Can be detected as min-cut edges in the sphere graph

## Parameters
| Parameter | Role |
|-----------|------|
| Voxel resolution | Spatial precision |
| Min sphere radius | Filter noise, define minimum detectable space |
| Connectivity threshold | How much overlap defines adjacency |
| Graph cut threshold | Define doorway vs. open connection |
