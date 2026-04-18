# Spectral Bisection via Fiedler Vector for Room Partitioning

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q4

---

## Fiedler Vector Definition
The eigenvector associated with the second-smallest eigenvalue (λ₂, the algebraic connectivity) of the graph Laplacian matrix L = D - W.

## Spectral Bisection Algorithm
1. Construct adjacency graph: nodes = spatial units (voxels/cells), edges = connectivity, weights = affinity
2. Compute graph Laplacian L = D - W (or normalized L_sym = I - D⁻¹/²WD⁻¹/²)
3. Compute second eigenvector (Fiedler vector) v₂
4. Partition by sign: positive entries → group A, negative → group B
5. The cut minimizes the normalized cut (ratio of edge weight between clusters to total edge weight)

## Recursive Multi-Way Partitioning
- Apply bisection recursively to each subgraph
- Continue until desired granularity (individual rooms)
- Alternative: use k eigenvectors directly for k-way partitioning

## Normalized Cut Metric
Ensures:
- Minimal edge weight between clusters (rooms with few doorways get separated)
- Balanced cluster sizes (avoids skewed partitions)
- NP-hard to optimize exactly; spectral methods provide polynomial-time approximation

## Indoor Application
- Graph: grid/voxel adjacency, weight by openness/visibility
- Door regions serve as natural split points (low-weight edges)
- Can combine with Voronoi graphs
- Distributed computation available for large networks

## Practical Tools
- METIS: multi-level graph partitioning
- scikit-learn: spectral clustering with Laplacian eigenmaps
- Custom: compute sparse Laplacian, use ARPACK/Lanczos for top-k eigenvectors
