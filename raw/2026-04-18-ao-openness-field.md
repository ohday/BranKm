# Ambient Occlusion as Spatial Analysis Field for Voxel Segmentation

- **URL**: Synthesized from search results including ScienceDirect articles
- **Fetched**: 2026-04-18
- **Relevant to**: Q1

---

## Key Insight
Ambient Occlusion (AO) values computed from multi-directional ray casting are essentially an **openness metric** — exactly what the user has with 26-direction ray distances. AO has been shown to outperform distance maps for spatial segmentation tasks.

## AO as Segmentation Field
From "Ambient occlusion – A powerful algorithm to segment shell and skeletal intrapores in computed tomography data" (ScienceDirect):
- AO fields **outperform distance maps** in separating connected cavities
- AO captures complex 3D morphology that distance fields miss
- Works by computing fraction of unoccluded rays per voxel

## Connection to User's 26-Direction Ray Data
The user's 26-direction ray distances are essentially a discrete AO/openness field:
- **Openness variants:**
  1. Average of all 26 ray distances → mean openness
  2. Minimum of 26 ray distances → most constrained direction
  3. Harmonic mean → emphasizes short rays (near walls)
  4. Variance of 26 ray distances → shape indicator (low variance = room center, high variance = near wall/corner)
  5. Directional histograms → anisotropy indicator

## From Ray Distances to Scalar Fields for Segmentation

### Field 1: Mean Openness (Ō)
Ō(v) = (1/26) Σᵢ ray_distance_i(v)
- High in room centers, low near walls, very low in doorways
- Good for watershed/persistence-based segmentation

### Field 2: Minimum Openness (Ō_min)
Ō_min(v) = min_i(ray_distance_i(v))
- Approximates distance-to-nearest-wall (similar to DT)
- Most conservative estimate of local space size

### Field 3: Directional Anisotropy (A)
A(v) = max_i(ray_distance_i) / min_i(ray_distance_i)
- High in corridors (long in one direction, short perpendicular)
- Low in room centers (roughly equal all directions)
- Directly useful for corridor detection

### Field 4: Cross-Section Indicator
For each of 13 opposite direction pairs:
C_j(v) = ray_distance_j(v) + ray_distance_opposite_j(v)
- Estimates cross-section diameter in that direction
- Min over all pairs ≈ local passage width
- Doorways have very small min(C_j)

## Deep Medial Fields (DMF)
Neural approach for O(1) thickness estimation from SDF. Encodes:
- Local thickness (medial sphere radius)
- Medial axis proximity
- Relevant but ML-based (out of user's scope)
