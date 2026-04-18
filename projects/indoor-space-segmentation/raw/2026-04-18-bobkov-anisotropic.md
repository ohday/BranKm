# Bobkov et al. — Room Segmentation via Anisotropic Potential Fields

- **URL**: https://ieeexplore.ieee.org/document/8019484
- **Fetched**: 2026-04-18
- **Relevant to**: Q5

---

## Core Method
Room segmentation in 3D point clouds using anisotropic potential fields + concavity-aware graph partitioning.

## Algorithm Steps
1. **Anisotropic Potential Field**: Computed for the 3D environment, encoding room boundaries as potential barriers
2. **Concavity Detection and Removal**: Novel robust criterion detects concave regions; removes them to simplify partitioning
3. **Graph Construction**: Point cloud → graph (nodes = spatial elements, edges = relationships)
4. **Graph Partitioning**: After concave region removal, partition into locally convex connected subgraphs
5. **Relabeling**: Removed concave regions relabeled using recovery procedure

## Key Properties
- Noise-resistant: handles high noise, varying point density, registration artifacts
- Non-Manhattan: works with curved walls
- Unsupervised: no training data needed
- No scanner pose information required
- Works with multi-view and single-view data

## Evaluation
Outperforms state-of-the-art on non-Manhattan-world datasets.
