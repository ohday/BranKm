# CGAL SDF-Based Mesh Segmentation — Complete Technical Details

- **URL**: https://doc.cgal.org/5.6/Surface_mesh_segmentation/index.html
- **Fetched**: 2026-04-18
- **Relevant to**: Q1, Q2

---

## Shape Diameter Function (SDF) Computation

### Core Concept
SDF measures local object diameter per facet. Pose-invariant.

### Ray Casting
1. Construct cone at facet centroid, axis = inward-normal
2. Sample 25 rays within cone (default), cone angle = 2π/3 radians (default)
3. Truncate each ray to segment between apex and first mesh intersection
4. Outlier removal on ray lengths, then average remaining

### Post-Processing
1. **Gap filling**: Facets with no SDF get neighbor average, then global min
2. **Bilateral smoothing**: window w = ⌊√(|F|/2000)⌋ + 1, spatial σ_s = w/2, range σ_r adaptive per-facet
3. **Linear normalization** to [0,1]

## Soft Clustering (GMM)
- Fit k Gaussian distributions to SDF value distribution
- Init: k-means++, multiple random restarts, best fit → EM refinement
- Output: probability matrix P(f|x_p) for each facet f and cluster x_p
- Default k=4. No direct relationship between k and final segment count.

## Hard Clustering (Graph-Cut)
Energy: E(x̄) = Σ e₁(f,x_f) + λ · Σ e₂(x_f,x_g)

- **Unary**: e₁(f,x_f) = -log(max(P(f|x_f), ε₁))
- **Pairwise**: Based on dihedral angle θ(f,g). Convex angles weighted w=0.08, concave angles w=1. Concave boundaries are natural segment boundaries.
- **λ**: smoothness parameter [0,1], default 0.3

Alpha-expansion graph cut algorithm. Segments = connected components of same cluster.

## Parameters
| Parameter | Default |
|-----------|---------|
| Rays per facet | 25 |
| Cone angle | 2π/3 |
| Number of clusters k | 4 |
| Smoothing λ | 0.3 |

## Performance (CGAL 4.4, i7 3.2GHz)
SDF computation: 7828 facets → 2.3s, 88928 facets → 46.1s
Segmentation: 88928 facets, k=5 → 1.26s

## Indoor Application Notes
- Mesh must be closed/2-manifold for reliable SDF
- Alternative scalar fields (height, wall-distance) can replace SDF using same graph-cut framework
- segmentation_from_sdf_values() accepts any [0,1]-normalized scalar field
