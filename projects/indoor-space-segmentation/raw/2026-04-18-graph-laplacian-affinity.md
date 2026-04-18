# Graph Laplacian Affinity Matrix for Indoor Voxel Grids

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q4

---

## Affinity Matrix Construction Options

### Option 1: Distance-Based Gaussian Kernel
W_ij = exp(-‖x_i - x_j‖² / 2σ²) for adjacent voxels, 0 otherwise

Simple but doesn't encode spatial structure well — adjacent voxels in same room vs through a wall have same weight.

### Option 2: Distance Transform Weighted
W_ij = exp(-1 / (DT(i) + DT(j))) 

Low affinity when either voxel is near a wall (low DT), high when both are in room center. Naturally creates low-weight edges at doorways.

### Option 3: Visibility-Gated
W_ij = exp(-d_ij) · I(visible(i,j))

Binary visibility check: if line-of-sight between voxels is blocked by wall, affinity = 0. Expensive for large grids.

### Option 4: Ray-Distance Affinity (Custom for User's Data)
Given 26-direction ray distances per voxel:
- Openness(v) = f(ray_distances_v) — e.g., average, harmonic mean, or min
- W_ij = min(Openness(i), Openness(j)) for adjacent voxels

This uses the existing baked data directly. Doorway voxels have low openness in the across-doorway directions → low affinity.

### Option 5: Multi-Feature Affinity
Combine multiple features:
W_ij = exp(-α·d_spatial - β/openness_min - γ·|openness_i - openness_j|)

- d_spatial: Euclidean distance
- openness_min: min openness of the two voxels
- |openness_i - openness_j|: gradient penalty

## Graph Laplacian Construction

### Unnormalized
L = D - W, where D = diag(W·1)

### Normalized (Symmetric)
L_sym = I - D⁻¹/²WD⁻¹/²
Preferred for spectral clustering — avoids bias toward high-degree nodes.

### Normalized (Random Walk)
L_rw = D⁻¹L = I - D⁻¹W
Equivalent to L_sym for clustering but easier to interpret.

## Eigendecomposition
- Compute k smallest eigenvalues/eigenvectors of L (or largest of D⁻¹W)
- Use ARPACK (Lanczos) for sparse matrices — efficient for large grids
- Only need first k eigenvectors, not full decomposition

## Automatic k Selection (Eigengap)
1. Compute eigenvalues λ₁ ≤ λ₂ ≤ ... ≤ λ_max
2. λ₁ = 0 always (trivial eigenvector = constant)
3. Number of eigenvalues near 0 = number of nearly-disconnected components
4. Largest gap max(λᵢ₊₁ - λᵢ) → choose k = i
5. For well-separated rooms (walls = strong boundaries), eigengap is clear

## Sparsity for Large Grids
For N voxels on a 3D grid:
- Each voxel has ≤ 26 neighbors
- W has ≤ 26N non-zero entries
- L is extremely sparse
- ARPACK handles millions of voxels efficiently for small k
- If k is large (many rooms), Nyström approximation may be needed
