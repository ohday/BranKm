# Runtime Acoustic Path Query — Architecture Synthesis

- **Type**: Synthesis (Steam Audio + NavMesh funnel literature + acoustic A*)
- **Fetched**: 2026-04-21
- **Project**: acoustic-portal-baking
- **Relevant to**: Q6, Q9
- **Sources drawn from**:
  1. [Steam Audio path_simulator.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_simulator.cpp)
  2. [Steam Audio path_finder.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_finder.cpp)
  3. [Steam Audio probe_tree.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_tree.cpp)
  4. [Computing Arbitrary Clearance for Navigation Meshes (Oliva et al.)](https://www.researchgate.net/profile/Ramon_Oliva/publication/257235867)
  5. [NavMesh funnel algorithm survey](https://en.wikipedia.org/wiki/Pathfinding)
  6. [Raghuvanshi 2021 portal-search](https://arxiv.org/abs/2107.11548)

---

## Problem statement

Given:
- A source at continuous 3D position `s`
- A listener at continuous 3D position `l`
- A baked acoustic graph (probes + visibility edges + per-pair shortest paths)

Return in < 1 ms:
- Acoustic distance `d` (how far the sound "traveled" around corners)
- Perceptual direction of arrival `v` (which doorway the sound came through)
- Attenuation profile (per-band gain, reflecting diffraction)

This must work for many sources simultaneously (e.g., 32 sources × 60 Hz = 1920 queries/sec).

---

## Architecture overview

```
┌──────────────┐     ┌──────────────┐
│ source pos s │     │ listener pos │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
  ┌────────────────────────────┐
  │ probeTree.getInfluencing() │   O(log N) BVH query each
  │   → source_neighborhood    │   returns up to kMaxProbesPerBatch=8
  │   → listener_neighborhood  │   probes per point
  └──────┬────────────┬────────┘
         │            │
         ▼            ▼
  ┌──────────────────────────────┐
  │ scene.isOccluded(s, l)?      │   Direct LOS check — single ray
  │   if NOT occluded: fast path │   (no portal needed, just play direct)
  └──────────────┬───────────────┘
                 │ occluded
                 ▼
  ┌───────────────────────────────────────┐
  │ For each (src_probe, lis_probe) pair: │    O(8 × 8) = 64 lookups
  │   soundPath = baked.lookupPath(i, j)  │    O(1) array access per
  │   if enableValidation:                │
  │     if isPathOccluded(soundPath):     │    optional re-check
  │       soundPath = findShortestPath()  │    A* on runtime vis-graph
  │   paths.append((soundPath, weight))   │
  └──────────────┬────────────────────────┘
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │ For each valid path:                │
  │   virtualSrc = path.toVirtualSource │
  │   distance = |virtualSrc - listener|│
  │   direction = unit(virtualSrc - l)  │
  │   SH.project(direction, gain, coeffs)│
  │   EQ += deviationModel(deviation)   │
  └──────────────┬──────────────────────┘
                 │
                 ▼
           mono input → SH convolution
                      → EQ filtering
                      → binaural/speaker decode
```

---

## 1. Point-to-probe lookup (`probeTree`)

Steam Audio uses a BVH over probe influence spheres (see `steam-audio-probe-placement.md`). O(log N) worst-case; typically < 1 μs for N ~ 10000.

Returns up to `kMaxProbesPerBatch = 8` probes **whose influence spheres contain the query point**. Each contributing probe gets a weight computed separately (typically inverse-distance-weighted within the sphere).

### Alternative: Poisson-disk neighborhood

If your probe graph uses Poisson-disk-distributed probes (instead of Steam Audio's floor grid), a fixed-K nearest-neighbor query is usually preferred. Kd-tree search is also O(log N).

---

## 2. Direct LOS early-out

**Critical for performance.** Most sources in most frames have LOS to the listener. A single raycast `scene.isOccluded(s, l)` handles this case and skips the entire graph machinery.

On voxel grids, this is a **3D-DDA ray march** through the voxel grid (Amanatides & Woo 1987) — O(scene_size / voxel_size) rays per check. At 5cm voxels over 50 m: ~1000 voxel steps, each an array lookup. Typically < 10 μs.

---

## 3. Path lookup — the O(1) miracle

The baked `mBakedPathRefs[start][end]` array stores a `SoundPathRef` (int32) per probe pair. Lookup:
```cpp
SoundPathRef ref = mBakedPathRefs(start, end);
if (ref.index == 0) return invalid;   // no path
SoundPath path = mUniqueBakedPaths[ref.index];
```

**Two array accesses** → get:
- 4 int16 probe indices (first/last and their neighbors)
- 2 floats (distance, deviation)

No graph traversal. No Dijkstra. The path is a **materialized view** of the baking-time shortest path.

### Why this is cheap storage

For N = 1000 probes:
- `mBakedPathRefs`: 1000×1000 int32 = 4 MB
- `mUniqueBakedPaths`: typically N·log(N) unique paths × 12 bytes ≈ 120 KB

Total: ~4 MB per probe batch. Doesn't scale to 10⁵ probes (would be 40 GB) — but for game levels ~5000 probes is usually enough.

### Alternative: Contraction Hierarchies

For very large graphs (N > 10⁴), Contraction Hierarchies (Geisberger et al. 2008) give O(log N) queries with O(N log²N) precomputation and O(N log N) storage. Overkill for most acoustic scenes.

---

## 4. Alternate path on validation failure — A*

If path validation (re-checking segment visibility against current scene state) fails, Steam Audio falls back to **A* on the runtime visibility graph**:

```cpp
ProbePath findShortestPath(scene, probes, visGraph, visTester, start, end):
    reset cost and parent arrays
    queue.push(start, 0)
    while queue not empty:
        u = queue.pop_min()
        if u == end: break
        for each (v, edgeCost) in visGraph.mAdjacent[u]:
            newCost = cost[u] + edgeCost
            if newCost < cost[v]:
                cost[v] = newCost
                parent[v] = u
                h = euclideanDistance(probes[v].center, probes[end].center)
                queue.push(v, newCost + h)    // f = g + h
    return reconstructPath(parent, start, end)
```

Classic A* with Euclidean heuristic (admissible for Euclidean edge costs).

### Path simplification (post-A*)

The runtime graph may be pruned to a smaller `visRangeRealTime` than the bake-time range, so baked paths can become jagged. `simplifyPath` greedily shortcuts:

```cpp
current = end
while current != start:
    parent = parents[current]
    grandparent = parents[parent]
    if visibility(current, grandparent):   // direct ray OR graph edge
        parents[current] = grandparent     // skip parent
    current = parents[current]
```

Multiple shortcuts per step possible; classic "funnel smoothing" for graphs.

---

## 5. 3D funnel algorithm — does Steam Audio use it?

**No.** The funnel algorithm (Chazelle 1984, Lee & Preparata; adapted for NavMesh by Mikko Mononen's Recast) is used for smoothing polyline paths across a sequence of polygon portals by pulling the path taut against each portal's endpoints.

Steam Audio does NOT use it because:
1. There are no explicit portal polygons — just probe centers
2. The "straight path" through a sequence of probe centers is already implicit in `path.toVirtualSource()` which places a virtual source at distance = path length, direction = last hop
3. Spatial accuracy at the millimeter level is not required for audio (you can't localize a source to < 10° for most azimuths)

### When you DO need the funnel

If your system uses explicit portal polygons (from morphological opening, persistence, etc.) and wants geometric-accurate diffracted path length (not just summed edge lengths), then:

```python
def funnel_3d(source, portals, listener):
    """
    Given a source, a sequence of portal polygons (aperture outlines),
    and a listener, find the shortest polyline that passes through
    each portal's interior.
    """
    # 3D generalization of NavMesh 2D funnel
    # Start with apex at source; left/right edges = sides of first portal
    # Walk through each portal tightening the funnel against polygon edges
    # When the new polygon's left edge crosses the old right edge, commit a
    # corner at the old right edge and reset the apex to that corner.
    # Output: polyline through corners
    ...
```

True 3D funnel is more complex than 2D (more degrees of freedom for tightening). For acoustic purposes, a simpler approach works: each portal contributes `+= straight_line_length(hop through portal center)` — approximate but audibly close.

### UTD path query with explicit edges

If you want **real UTD** (not Steam Audio's deviation surrogate), the path query becomes:

```python
def find_utd_path(source, listener, portal_edges):
    """Find sequence of diffracting edges from source to listener."""
    # 1. Find candidate portals between source-room and listener-room (graph search)
    # 2. For each sequence, compute UTD-accurate path:
    #    - Diffracted ray must lie on Keller cone at each edge
    #    - Solve system for ray geometry given edge tangents
    # 3. Return sequence + per-edge (β₀, β, β', L)
    ...
```

This is Tsingos 2001's beam-tracing approach. Orders of magnitude more complex than Steam Audio's lookup.

---

## 6. Multi-path aggregation

At typical game scenes, `findPaths` may find up to 64 distinct `(source_probe → listener_probe)` paths. They are combined as **incoherent sums**:

```python
eq_gains = [0] * numBands
sh_coeffs = [0] * (order+1)**2
for path, weight in paths:
    gain_i = weight * distance_attenuation(path.distance)
    direction_i = unit(virtualSrc_i - listener)
    # SH projection (ambisonic encoding of this path's direction)
    SH.accumulate(direction_i, order, gain_i, sh_coeffs)
    # Per-band EQ contribution
    for band in range(numBands):
        eq_gains[band] += weight * overall_gain * utd_factor(path.deviation, band) / ref_factor
```

This is energy-summed (not amplitude-summed), so paths add constructively in power. Equivalent to incoherent superposition of Gaussian reflections/diffractions — standard practice in perceptual geometric acoustics.

---

## 7. Memory layout for cache-friendly queries

Steam Audio's data organization is optimized for linear memory access during rendering:

- `mBakedPathRefs(start, end)` is laid out as `Array<SoundPathRef, 2>` — row-major, so iterating `end` for fixed `start` is sequential access
- `mUniqueBakedPaths` is a flat array; indexing is a single load
- `mAdjacent` (visibility graph) is `vector<vector<AdjacencyListEntry>>` — slight cache miss risk at inner-vector boundaries, but inner vectors usually small (<50 entries)

### Recommended for a voxel clone

```c
struct SoundPath {
    int16_t first_probe, last_probe;
    int16_t probe_after_first, probe_before_last;
    float distance_internal, deviation_internal;
};   // 16 bytes — fits 4 per cache line

struct BakedPathData {
    uint32_t num_probes;
    // Row-major: ref(src, lst) = refs[src * num_probes + lst]
    int32_t* refs;          // num_probes * num_probes * 4 bytes
    SoundPath* unique_paths;
    uint32_t num_unique_paths;
    // Visibility graph as CSR (compressed sparse row) for cache locality:
    uint32_t* adjacency_offsets;   // num_probes + 1 entries
    uint32_t* adjacency_indices;
    float* adjacency_costs;        // costs parallel to indices
};
```

CSR graph representation gives better cache behavior than vector-of-vectors for Dijkstra/A* inner loops.

---

## 8. Concrete latency budget

For a typical game audio tick (60 Hz, 32 sources concurrent):

| Phase | Per-source cost | 32 sources |
|---|---|---|
| probeTree (src + lis) | ~1 μs | 32 μs |
| scene.isOccluded (direct LOS) | ~10 μs | 320 μs |
| Path lookup (if occluded, 64 pairs) | ~2 μs | 64 μs |
| Validation (optional, 16 pairs x 10us) | ~160 μs | 5 ms |
| A* fallback (rare, 1-2 sources/tick) | ~50 μs | 100 μs |
| SH + EQ accumulation | ~5 μs | 160 μs |
| **Total (no validation)** | **~20 μs** | **~0.6 ms** |
| **Total (with validation)** | **~180 μs** | **~6 ms** |

Game audio thread budget: usually ~5-10 ms/tick. Validation-off fits easily; validation-on is tight — disable validation or cache results.

---

## 9. Dynamic geometry — open/close doors [6]

Raghuvanshi 2021's Dynamic Portal Occlusion handles runtime portal-state changes without re-baking:

1. **At bake time**: identify portal positions heuristically
2. **Per frame**: for each (source, listener) pair, use the precomputed path direction to "guess" which portals were traversed
3. **Apply closure state**: multiply the per-band gain by `(1 - closure)` per crossed portal

For Steam-Audio-style systems, an equivalent:

```python
# In findPaths, after computing each SoundPath:
for portal_i in identify_portals_on_path(path):
    if portal_i.closed_fraction > 0:
        # Extra high-frequency attenuation per closed portal
        for band in range(numBands):
            eq_gains[band] *= (1 - portal_i.closed_fraction)**band_penalty[band]
```

Requires knowing **which portals each path crosses** — this is where explicit portal detection (the hybrid approach) pays off, even if the core graph is implicit.

---

## 10. Summary recommendations

For a voxel-based implementation:

1. **Probe placement**: dense grid on free voxels at SDF > threshold (say, SDF > 0.5 m). Steam Audio's floor-grid approach adapts trivially.
2. **Visibility test**: 3D-DDA ray march on voxel grid. Single ray per pair for cheap bakes; 16-sample volumetric for robust bakes near tight geometry.
3. **Visibility graph**: CSR format for cache efficiency. Parallel build over pair range.
4. **All-pairs shortest paths**: Dijkstra per source, parallel across sources.
5. **Runtime path lookup**: row-major ref table → unique-path table.
6. **Runtime visibility re-check**: optional, for dynamic geometry; use A* on pruned runtime graph.
7. **UTD surrogate**: copy Steam Audio's `deviation.cpp` verbatim (~200 lines).
8. **SH + EQ DSP**: convolution + multi-band filter. Standard audio pipeline.
9. **Dynamic portals (optional)**: layer Raghuvanshi-style closure modulation on top using morphological-opening-detected portal locations.
