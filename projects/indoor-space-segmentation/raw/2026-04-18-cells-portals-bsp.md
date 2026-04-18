# Cells-and-Portals via BSP Trees for Indoor Scenes

- **URL**: Synthesized from multiple search results
- **Fetched**: 2026-04-18
- **Relevant to**: Q2, Q5

---

## BSP Tree Cell-Portal Pipeline

### Cell Generation
1. Select splitting planes (typically wall-aligned) to recursively partition space
2. Each plane divides space into front/back child nodes
3. Leaf nodes = convex cells (rooms or room segments)
4. Indoor environments benefit from architectural constraints — walls form natural splitting planes

### Portal Generation
1. Initial portals created on splitting planes that separate cells
2. Portals clipped against other splitting planes in BSP tree
3. Final portals = actual openings between cells after all clipping
4. Redundant portals (fully occluded by geometry) eliminated

### Portal-Based Visibility Culling
1. Locate camera's cell via BSP tree traversal
2. Mark starting cell as visible
3. For each portal in current cell:
   - Check if portal is within view frustum
   - Clip view frustum against portal boundaries
   - Recursively process connected cell through portal with clipped frustum
4. Only geometry in visible cells rendered

## Advantages for Indoor
- Buildings naturally consist of convex spaces connected by well-defined openings
- Strong consistent occlusion from walls
- Predictable portal connectivity

## Challenges
- BSP balance with arbitrary layouts
- Complex architectural features may need manual refinement
- Dynamic elements (doors) need "active portal" handling

## Historical Context
Pioneered by Doom/Quake. Remains relevant for precise visibility control.
