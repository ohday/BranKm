# Area Graph: Voronoi-Based Map Segmentation with Room Detection

- **URL**: Synthesized from search results + https://springer.iq-technikum.de/article/10.1007/s11370-021-00392-5
- **Fetched**: 2026-04-18
- **Relevant to**: Q2, Q3, Q5

---

## Approach
Topological map where vertices = areas (rooms), edges = passages (doors/corridors).
Built from pruned Voronoi Graph (Topology Graph) with room detection post-processing.

## Algorithm Pipeline
1. Create Voronoi Diagram (VD) from 2D grid map
2. Filter and prune VD to obtain Topology Graph
3. Maintain region faces as areas in the Area Graph
4. Apply room detection as post-processing to prevent over-segmentation

## Room Detection & Over-Segmentation Prevention
- Uses Voronoi diagrams to locate narrow passages in free zones that coincide with protruding parts in occupied zones (indicating potential doors)
- Dynamic threshold adapts to map structure rather than fixed value
- Room detection algorithm compensates for Voronoi Graph instability in open areas

## Corridor Identification
- Analyzes shape of both free and occupied Voronoi regions
- Narrow passages in corridors identified through edge detection and clustering
- Corridor structural cues: walls, floors, boundaries
- Multi-LiDAR for occlusion-free mapping

## Code Availability
- MAORIS: https://github.com/MalcolmMielle/maoris
- Area Graph: https://github.com/STAR-Center/areaGraph
