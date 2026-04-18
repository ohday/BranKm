# Nyström Approximation for Scalable Spectral Clustering

- **URL**: https://ar5iv.labs.arxiv.org/html/2006.14470
- **Fetched**: 2026-04-18
- **Relevant to**: Q4

---

## Standard Spectral Clustering (Exact)
Kernel matrix K ∈ ℝⁿˣⁿ, κ(xᵢ,xⱼ) = exp(-‖xᵢ-xⱼ‖²/σ²)
1. Form K, compute M = D⁻¹/²KD⁻¹/² where D = diag(K1ₙ)
2. Compute k leading eigenvectors of M
3. Normalize rows, run K-means
Complexity: O(n²d) to form K, O(n²k) for eigendecomp

## Nyström Method

### Landmark Selection
Select m < n points uniformly without replacement. Paper notes results "hold for any landmark selection process."

### Matrix Construction
- C ∈ ℝⁿˣᵐ: κ(xᵢ, zⱼ) — all n points vs m landmarks
- W ∈ ℝᵐˣᵐ: κ(zᵢ, zⱼ) — pairwise among landmarks
- Approximation: K ≈ K̂ = CW†Cᵀ
- Cost: O(nmd)

### Proposed Algorithm (Algorithm 2)
1. Form C and W — O(nmd)
2. EVD of W = U_W Σ_W U_Wᵀ. Find rank l via: l = max{i : σᵢ(W)/σ₁(W) ≥ γ}
3. G = C U_{W,l} Σ_{W,l}⁻¹/² ∈ ℝⁿˣˡ — O(nml)
4. D̂ = diag(G(Gᵀ1ₙ)) — two mat-vec products, avoids forming n×n
5. G̃ = D̂⁻¹/²G
6. SVD of G̃ → k leading left singular vectors Û_M ∈ ℝⁿˣᵏ
7. Normalize rows to unit length
8. K-means on normalized rows

### Key: Adaptive Rank l
- l determined by spectrum of W, not forced to k
- γ = 10⁻² recommended default
- Prevents information loss from premature rank reduction

## Complexity Comparison
| Method | Cost | Passes over C |
|--------|------|---------------|
| Exact SC | O(n²k) | N/A |
| Approach 1 [Fowlkes] | O(nm²) | Multiple |
| Approach 2 [Chen&Cai] | O(nmk) | Single |
| **Proposed** | **O(nml)**, k≤l≤m | **Single** |

## Practical Guidelines
- m must exceed k; experiments use m=40-320 for n=8000-100000
- γ = 10⁻² gives reasonable trade-off
- On blobs (n=100000, k=3, m=200): γ ∈ {10⁻³, 10⁻²} → perfect clustering
- Proposed method outperforms alternatives especially when m is small
- Average l for γ=10⁻²: roughly 50-75% of m

## Eigengap Heuristic for Automatic k
- Compute eigenvalues λ₁ ≤ λ₂ ≤ ... of graph Laplacian
- Largest gap δᵢ = |λᵢ - λᵢ₊₁| → select k at that index
- Multiplicity of eigenvalue 0 = number of connected components
- Well-separated clusters produce k small eigenvalues then significant gap
- Can combine with silhouette score for robustness
