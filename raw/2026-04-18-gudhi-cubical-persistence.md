# Gudhi Persistent Homology for Cubical Complexes

- **URL**: Synthesized from search results + https://gudhi.inria.fr/python/3.8.0/index.html
- **Fetched**: 2026-04-18
- **Relevant to**: Q2, Q5

---

## Gudhi Cubical Complex API

```python
import gudhi
import numpy as np

# Create cubical complex from 3D numpy array
# scalar_field shape: (x_dim, y_dim, z_dim)
cubical_complex = gudhi.CubicalComplex(
    dimensions=[x_dim, y_dim, z_dim],
    top_dimensional_cells=scalar_field.flatten()
)

# Compute persistence
persistence = cubical_complex.persistence()

# Get persistence pairs (birth, death) for dimension 0
# Dimension 0 = connected components
# Long bars = robust rooms, short bars = noise
pairs_dim0 = cubical_complex.persistence_intervals_in_dimension(0)
```

## How It Works for Room Segmentation
1. Input: 3D scalar field (e.g., distance transform or openness field)
2. Lower-star filtration creates nested subcomplexes
3. Dimension 0 persistence tracks connected components:
   - Born at local maximum of inverted field (room center)
   - Die when component merges with another (through doorway)
   - Death value = bottleneck width
4. Persistence diagram shows all (birth, death) pairs
5. Filter by persistence threshold to identify robust rooms

## Complexity
- Efficient for cubical complexes on grids
- Memory: proportional to grid size
- Computation: near-linear for cubical complex persistence

## Integration Strategy for User's Use Case
1. Convert 26-direction ray data to scalar "openness" per voxel
   - Openness = average of 26 ray distances
   - Or: Openness = minimum of 26 ray distances (more conservative)
   - Or: Openness = harmonic mean (emphasizes small values)
2. Create CubicalComplex from openness field
3. Compute 0-dimensional persistence
4. Birth-death pairs give room hierarchy
5. Cut at desired persistence threshold → room segmentation
