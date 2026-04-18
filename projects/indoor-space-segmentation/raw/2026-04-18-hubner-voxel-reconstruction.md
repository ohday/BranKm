# Hübner et al. — Voxel-Based Indoor Reconstruction and Room Partitioning

- **URL**: https://ar5iv.labs.arxiv.org/html/2002.07689
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q2, Q5

---

## Voxel Grid
- 3D array of byte values, uniform 5cm voxel size
- Initial states: Empty, Normal Up (±45°), Normal Down (±45°), Normal Horizontal
- Multi-class/multi-room: voxel can belong to multiple rooms with different labels
- Semantic labels: Ceiling, Floor, Wall, Wall Opening, Empty Interior, Interior Object

## Room Detection Pipeline

### Ceiling Detection
1. Pre-sweep: Normal Down voxel above Normal Horizontal → reclassify to Horizontal (ensures rooms separated by walls produce distinct ceiling segments)
2. Region growing: Connected components of Normal Down voxels using 3D-26-neighbourhood
3. Size filter: Discard segments < 0.5 m² horizontal coverage

### Ceiling Refinement (Hole Closing)
1. Project ceiling onto 2D horizontal pixel grid with height values
2. Detect holes between non-empty pixels along 4 main directions (2D-8-neighbourhood)
3. Segment holes using 2D-4-neighbourhood
4. Assign height via 2D ray tracing with linear interpolation

### Floor Detection
1. From each ceiling voxel, trace downward through voxel grid
2. Normal Up voxel before grid bottom → floor candidate
3. Region growing with 2.5D-8-neighbourhood, max height difference 18cm (stair step)
4. Discard < 0.5m²; select lowest height segment as floor

### Voxel Classification Sweep
Top-to-bottom: Below ceiling → same room ID. If ceiling has no Wall label → Empty Interior/Interior Object. If ceiling has Wall label → Wall/Wall Opening.

## Wall Opening (Door) Detection
- Detected implicitly: empty voxels below wall-labeled ceiling = Wall Opening
- Refinement: if any voxel in a perpendicular-to-wall stack is solid Wall, whole stack → Wall
- Furniture occlusion check at 70cm eliminates false openings
- Limitation: occlusions from tables not above wall are not handled

## Key Parameters
| Parameter | Value |
|-----------|-------|
| Voxel size | 5 cm |
| Normal threshold | ±45° |
| Min ceiling/floor area | 0.5 m² |
| Floor height diff | 18 cm |
| Hole closing occupancy | ≥75% |
| Wall thickening search | 15 cm |
| Occlusion check distance | 70 cm |

## Evaluation
~92% correct for Office dataset, ~85% for Attic dataset.
