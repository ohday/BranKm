# Steam Audio Pathing: Complete Source-Code-Level Breakdown

- **Type**: Synthesis (aggregated from multiple source files in the Steam Audio repository)
- **Fetched**: 2026-04-21
- **Project**: acoustic-portal-baking
- **Relevant to**: Q1, Q5, Q6, Q7, Q9, Q10 (especially Q10 and Q1)
- **Sources drawn from**:
  1. [path_simulator.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_simulator.h)
  2. [path_simulator.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_simulator.cpp)
  3. [path_finder.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_finder.h)
  4. [path_finder.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_finder.cpp)
  5. [path_data.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_data.h)
  6. [path_data.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_data.cpp)
  7. [path_visibility.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_visibility.h)
  8. [path_visibility.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_visibility.cpp)
  9. [probe_batch.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_batch.h)
  10. [probe_tree.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_tree.h)
  11. [probe_tree.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_tree.cpp)
  12. [probe_generator.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_generator.h)
  13. [probe_generator.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_generator.cpp)
  14. [probe_manager.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_manager.h)
  15. [deviation.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/deviation.h)
  16. [deviation.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/deviation.cpp)
  17. [api_path_effect.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/api_path_effect.cpp)
  18. [Steam Audio C API Programmer's Guide](https://valvesoftware.github.io/steam-audio/doc/capi/guide.html)

---

## BIG PICTURE — the architecture nobody expected

**Steam Audio does NOT use explicit "portals" at all.** There is no portal polygon, no aperture outline, no portal edge list. The word "portal" does not appear in `path_simulator.cpp` or any of the pathing source files.

Instead, acoustic connectivity is modeled as:

```
Probes (free-space samples) ─[ visibility graph ]─ Probes
                       │
               Dijkstra precomputes SHORTEST PATH table
                       │
               At runtime: look up path, sum DEVIATION ANGLE along nodes,
               feed angle into UTD formula → frequency-dependent EQ
```

This is the single most important insight for Q1, Q7, and Q10. The geometry-agnostic probe graph **implicitly** encodes all portal/bottleneck information — if a door is a 1m×2m opening, only probes on both sides that can see each other through it produce edges in the visibility graph. No explicit portal is needed.

[1][2][5]

---

## 1. Probe Generation — `probe_generator.cpp` [12][13]

### Public API
```cpp
enum class ProbeGenerationType { Centroid, UniformFloor, Octree };

static void ProbeGenerator::generateProbes(
    const IScene& scene,
    const Matrix4x4f& obb,        // oriented bounding box of region
    ProbeGenerationType type,
    float spacing,                 // e.g., 2.0 m
    float height,                  // e.g., 1.5 m above floor
    ProbeArray& probes);
```

### UniformFloor algorithm (the only real one used)

1. Compute grid on the **top face** of the OBB.
2. Per grid point, cast a **downward ray** through the scene via `scene.closestHit(ray, height, distanceFromFloor + height)`.
3. On each hit, place a probe **at `hit + up * height`** (typically 1.5 m above the floor — ear level).
4. **Continue the ray downward** past the hit (with a `kDownwardOffset = 0.01f` bias) to find additional floors below — this gives multi-story support.
5. Loop stops when remaining downward distance hits zero.

Grid spacing calculation:
```
numProbesX = floor(sx / spacing) + 1
numProbesZ = floor(sz / spacing) + 1
residualX = (sx - (numProbesX - 1) * spacing) / 2     // centers grid
```

Each probe gets `influence.radius = spacing`, so adjacent influence spheres just touch.

`Octree` case is declared in the enum but `generateProbes` throws `Exception(Status::Initialization)` for any type other than `Centroid` / `UniformFloor`. **Octree-based probe generation is NOT implemented in the open-source release**, despite being referenced.

### Important: probes are NOT on a medial axis, NOT Poisson-disk, NOT adaptive

They are just a **dense uniform grid on every floor** inside an artist-placed OBB. Density is purely artist-controlled via the `spacing` parameter.

---

## 2. Probe Data Structures [9][10]

### `Probe` — one field only
```cpp
struct Probe { Sphere influence; };   // center + radius
```

### `ProbeBatch` — the atomic unit for load/unload
Holds `vector<Probe>` + a `ProbeTree` (BVH) + a `map<BakedDataIdentifier, unique_ptr<IBakedData>> mData` that stores the baked pathing data for this batch.

### `ProbeNeighborhood` — runtime query result
```cpp
static const int kMaxProbesPerBatch = 8;   // max probes that can "influence" a query point
Array<ProbeBatch*> batches;
Array<int>         probeIndices;
Array<float>       weights;
// buffers for inter-probe occlusion checks:
Array<Ray>   rays;
Array<float> minDistances, maxDistances;
Array<int>   rayMapping;
Array<bool>  isOccluded;
```

### `ProbeTree` — BVH over influence spheres, not k-d tree [10][11]

The header comment calls it a "k-d tree" but implementation is a standard **top-down median-split BVH** built iteratively on a stack:

- `2 * numProbes - 1` nodes in a flat array
- Bit-packed metadata in SIMD padding: low 2 bits of `box.minCoordinates.w` encode split axis (0/1/2 for x/y/z, **3 = leaf sentinel**); upper bits store child offset / probe index
- Left/right children contiguous: left at `this[data() >> 2]`, right at `this[(data() >> 2) + 1]`
- At each build step: pick axis with **largest extent**, median-split sorted centroids, push right task to stack, continue with left task (classic iterative BVH)

`getInfluencingProbes(point, probes, maxInfluencingProbes, probeIndices)`:
1. AABB `contains(point)` test at each internal node → subtree pruned if outside
2. Near child visited first (same side of split plane as query); far child pushed to stack
3. At leaf: real sphere containment test `probes[i].influence.contains(point)`
4. Early exit when `maxInfluencingProbes` reached

**Returns "all probes whose influence sphere contains this point" — not K-nearest, not Poisson-disk-nearest.**

---

## 3. Visibility Graph — `path_visibility.cpp` [7][8]

### `ProbeVisibilityTester::areProbesVisible`

Two modes selected inside the function:

**Mode A — single ray (`numSamples <= 1 || radius <= 0`):**
```cpp
return !scene.isOccluded(fromProbeCenter, toProbeCenter);
```

**Mode B — volumetric (probes treated as spheres of given `radius`):**
```cpp
// pre-generated mSamples via Sampling::generateSphereVolumeSamples()
for each sample i in fromSphere:
    fromSample = probeCenterFrom + transformedSample_i * radius
    if scene.isOccluded(probeCenterFrom, fromSample) : continue   // sample inside geometry, skip
    for each sample j in toSphere:
        toSample = probeCenterTo + transformedSample_j * radius
        if scene.isOccluded(probeCenterTo, toSample) : continue
        if !scene.isOccluded(fromSample, toSample): numVisibleSamples++
        if numVisibleSamples / mSamples.size(0) >= threshold : return true
return false
```

Worst case `O(numSamples²)` rays per probe pair. Early exit triggers as soon as visible fraction ≥ `threshold`.

### Asymmetric visibility range (`areProbesTooFar`)
Optional horizontal-biased range check: removes the vertical component of displacement along `mDown` before comparing to `visRange`. This lets a user set a larger horizontal range than vertical (useful for multi-story buildings where floors should be acoustically coupled horizontally but less so vertically).

### `ProbeVisibilityGraph` — the heart of the system

Stored as `vector<vector<AdjacencyListEntry>> mAdjacent`, where:
```cpp
struct AdjacencyListEntry { int index; float cost; };
```

**Construction** (parallel, via `JobGraph`):
- Distribute adjacency rows across jobs.
- For each `(i, j)` with `i > j`:
  1. `areProbesTooFar(..., visRange)` → skip if too far
  2. `areProbesVisible(..., radius, threshold)` → skip if occluded
  3. Else append `AdjacencyListEntry{j, euclideanDistance(center_i, center_j)}` to `mAdjacent[i]`
- Last job to finish performs the **symmetrization pass**: for every `(i, j)` edge, add the reverse `(j, i)` edge.

**Edge cost = Euclidean distance between probe centers.** There's NO obstruction penalty, NO angle-based weight, NO surface-material cost in the edge itself.

**Serialization stores only `j < i` edges (lower triangle).** Reverse edges reconstructed at load.

**`prune(visRange)`** removes edges beyond a given range — used to keep only "important" edges after baking. `updateCosts()` refreshes edge costs if probes move (but topology remains).

---

## 4. Baking — `BakedPathData::BakedPathData(...)` [5]

```cpp
BakedPathData::BakedPathData(
    const IScene& scene,
    const ProbeBatch& probes,
    int numSamples,
    float radius,
    float threshold,
    float visRange,              // bake-time range
    float visRangeRealTime,      // shorter range used when real-time-checking edges later
    float pathRange,             // max cumulative path length stored
    bool asymmetricVisRange,
    const Vector3f& down,
    bool pruneVisGraph,
    int numThreads,
    ThreadPool& threadPool,
    std::atomic<bool>& cancel,
    ProgressCallback progressCallback,
    void* callbackUserData);
```

### Baking pipeline

1. **Build `ProbeVisibilityGraph`** — parallel across probe pairs.
2. **Run `PathFinder::findAllShortestPaths` for every probe as a start node** — O(numProbes) Dijkstra runs, embarrassingly parallel.
3. Each run produces `ProbePath` for every `(start, end)` pair.
4. **Deduplicate paths** into `Array<SoundPath> mUniqueBakedPaths` and store per-pair references in `Array<SoundPathRef, 2> mBakedPathRefs` (indexed by `[start][end]`).
5. If `pruneVisGraph` is set, call `visGraph.prune(visRangeRealTime)` to shrink the runtime graph to the smaller range used for alt-path fixups.

### `SoundPath` — minimal per-pair metadata [5][6]
```cpp
struct SoundPath {
    int16_t firstProbe;        // 2nd probe (from start side)
    int16_t lastProbe;         // 2nd-to-last probe (from end side)
    int16_t probeAfterFirst;   // 3rd probe (valid if >=2 interior nodes)
    int16_t probeBeforeLast;   // 3rd-to-last probe
    bool    direct;            // is this just a straight LOS path?
    float   distanceInternal;  // path length between firstProbe and lastProbe
    float   deviationInternal; // summed turn-angle between firstProbe and lastProbe
};
```

Crucially, **SoundPath does NOT store the full list of interior probes**. It only stores the first two and last two. This is the key compression trick — millions of pairs × hundreds of probes would blow memory, but storing 4 int16s + 2 floats per pair is cheap.

At runtime, if the full path must be walked (e.g., for validation), `reconstructProbePath()` follows the chain by repeatedly calling `bakedPathData.lookupShortestPath(start, prev, nullptr)` on sub-paths.

### Memory scaling

For N probes:
- `mBakedPathRefs` is N × N int32 array ≈ **4·N² bytes** for references (the heavy item)
- `mUniqueBakedPaths` only ~O(N·log N) to O(N²) unique paths
- Visibility graph stores ~E edges with `(int, float)` = 8 bytes each; E ≈ k·N for a sparse graph

For 1000 probes: ~4 MB refs + some MB for unique paths. Steam Audio limits a batch to a few thousand probes in practice.

---

## 5. Dijkstra & A* — `PathFinder` [3][4]

### `findAllShortestPaths` (BAKE side, Dijkstra)

Classic Dijkstra with priority queue + `pathRange` cutoff:
```cpp
mCosts[t][start] = 0;  mParents[t][start] = -1;
push(start, 0);
while (queue not empty):
    u = pop_min()
    for each (v, edgeCost) in visGraph.mAdjacent[u]:
        newCost = mCosts[t][u] + edgeCost
        if newCost > pathRange: continue             // range cutoff
        if newCost < mCosts[t][v]:
            mCosts[t][v] = newCost
            mParents[t][v] = u
            push(v, newCost)
```

Then for each probe `i`, walk parent chain back to `start` and reverse → `ProbePath`.

### `findShortestPath` (RUNTIME side, A*)

Same as Dijkstra but pushed cost includes a heuristic:
```cpp
push(v, newCost + ProbeDistance(v, end))
    // ProbeDistance = Euclidean distance between probe centers
```
The true `g(n)` is still stored in `mCosts`; the heuristic only biases queue ordering (classic A* f = g + h).

Duplicate entries for the same node may linger (std::priority_queue doesn't support decrease-key) — standard lazy-deletion Dijkstra.

### Path simplification — `simplifyPath`

Greedy smoothing that shortcuts intermediate nodes:
```
current = end
loop:
    parent = parents[current]
    grandparent = parents[parent]
    if current can see grandparent (either via graph edge OR real-time ray test):
        parents[current] = grandparent   // skip parent
        repeat (can chain multiple shortcuts)
    else:
        current = parents[current]
```

Typical use: the runtime visibility graph was pruned to `visRangeRealTime < visRange`, so baked paths may pass through nodes that are now disconnected but geometrically still LOS — simplification fixes the jaggedness.

### Thread safety
`mParents(numThreads, numProbes)`, `mCosts(numThreads, numProbes)`, `mPriorityQueue[numThreads]` — per-thread state; no locks needed.

---

## 6. Runtime Query — `PathSimulator::findPaths` [1][2]

```cpp
bool findPaths(
    const Vector3f& source,
    const Vector3f& listener,
    const IScene& scene,
    const ProbeBatch& probes,
    const ProbeNeighborhood& sourceProbes,    // probes near source
    const ProbeNeighborhood& listenerProbes,  // probes near listener
    float radius, float threshold, float visRange, int order,
    bool enableValidation, bool findAlternatePaths, bool simplifyPaths,
    bool realTimeVis,
    float* eqGains,                // OUT: per-band EQ
    float* coeffs,                 // OUT: SH coefficients
    const DistanceAttenuationModel& distanceAttenuationModel,
    const DeviationModel& deviationModel,    // UTD or custom
    ...);
```

### Algorithm

1. **Direct LOS test:** `scene.isOccluded(listener, source)` — single ray.
   - If **unoccluded** and `!forceDirectOcclusion`: emit a single direct path with `direct = true`, weight 1.0. Done. (This is the "listener can see source" fast path.)

2. **Occluded path:** must have valid source/listener probes AND baked pathing data.

3. **For each (source probe, listener probe) pair** (up to `kMaxPaths = 64`):
   - `SoundPath path = bakedPathData.lookupShortestPath(sourceProbeIndex, listenerProbeIndex, nullptr);`
   - `isPathOccluded(path, scene, probes, radius, threshold, start, end, ...)`:
     - Walk path segments backward `current → prev → …`
     - Per segment call `mVisTester.areProbesVisible(scene, probes, current, prev, ...)`
     - If any segment is occluded → path is invalidated
   - If invalidated and `findAlternatePaths`:
     - `mPathFinder.findShortestPath(scene, probes, visGraph, visTester, start, end, ...)`
     - Convert resulting `ProbePath` → `SoundPath`
     - Re-insert as an alternate path
   - Weight: `sourceProbeWeight * listenerProbes.weights[listenerProbeNeighborhoodIndex]`

4. **Accumulate SH coefficients** (`calcAmbisonicsCoeffsForPaths`):
   ```
   for each valid path i:
       virtualSource = (direct) ? source : path.toVirtualSource(probes, starts[i], ends[i])
       distance      = ||virtualSource - listenerRef||
       gain          = weights[i] * distanceAttenuationModel.evaluate(distance)
       direction     = unit(virtualSource - listener)
       SphericalHarmonics::projectSinglePointAndUpdate(direction, order, gain, coeffs)
   ```
   The **virtual source** is `listenerProbeCenter + lastProbeDir · pathTotalDistance` — places the spatialized image at the correct acoustic distance but with the direction of the **last path hop** (so sound seems to arrive from the correct doorway).

5. **Accumulate EQ** (`calcEQForPaths`) — see next section.

### `kMaxPaths = 64`
Hard cap on the number of listener-side probes contributing. This is the runtime memory budget.

---

## 7. Deviation → UTD → EQ — the diffraction model [15][16]

### `SoundPath::deviation` — sum of turn angles [6]

```cpp
// At construction from a ProbePath:
for i in [1, probePath.nodes.size()-2]:
    prevDir = unit(center[i] - center[i-1])
    nextDir = unit(center[i+1] - center[i])
    deviationInternal += angleBetween(prevDir, nextDir)

// At query time, also add the junction angles at firstProbe and lastProbe
// connecting start/end external endpoints to the stored path.
// If single interior probe: just angle(source → firstProbe → listener).
```

**This is a geometry-only quantity. No wedge angle, no source/edge direction relative to the diffracting edge — just the total "bend" of the sequence of probe centers.**

### `DeviationModel::utdDeviation(angle, band)` — the actual diffraction math [16]

Implementation follows **Tsingos, Funkhouser, Chhokra, Berta, Saunders (SIGGRAPH 2001)** — "Modeling Acoustics in Virtual Environments Using the Uniform Theory of Diffraction".

Per-band UTD computation with these fixed surrogate parameters:
```
n        = 2.0         // half-plane wedge (i.e., a straight edge — not actual wedge angle!)
α_i      = 0.0         // incident angle fixed
α_d      = α_i + π + angle
L        = 0.05 m      // Fresnel integral distance surrogate (!)
c        = speed_of_sound
f        = band center freq (Bands::kLowCutoffFrequencies[band] ... kHighCutoffFrequencies[band])
λ        = c / f
k        = 2π / λ
```

Then the standard UTD formula:
```
D0 = exp(-j·π/4) / (2·n·sqrt(2·π·k))

β1 = β2 = α_d - α_i
β3 = β4 = α_d + α_i

t_1 = cot((π + β1) / (2n))     // four cotangent terms, one per shadow boundary
t_2 = cot((π - β2) / (2n))
t_3 = cot((π + β3) / (2n))
t_4 = cot((π - β4) / (2n))

a(n, β, N) = 2·cos²(π·n·N − β/2)        // with N± chosen based on β crossings of π·n

F(x) = Fresnel transition function, piecewise:
       x < 0.8   : √(πx) · (1 - √x/(0.7√x+1.2)) · exp(j·π/4 · √(x/(x+1.4)))
       x ≥ 0.8   : (1 - 0.8/(x+1.25)²) · exp(...)

D_i     = t_i · F(k·L·a_i)               // with cotangent-singularity regularization
|D|     = |D0 · (D1 + D2 + D3 + D4)|     // returned magnitude per band
```

`DeviationModel::evaluate(angle, band)` returns this scalar. Used in `calcEQForPaths`:

```cpp
refTerm = utdDeviation(1e-8, band)        // near-zero deviation as normalizer
term    = utdDeviation(angle, band) / refTerm
eqGains[band] += weight_i * overallGain * term
```

Multi-band result is normalized separating overall gain from EQ shape via `EQEffect::normalizeGains`.

### The shocking simplification

**Steam Audio uses UTD coefficients but with dummy wedge parameters** (n=2 flat half-plane, α_i=0, L=0.05). The geometry-specific parts of UTD — actual wedge angle, incident direction relative to the edge, distance from source and listener to the edge — are all replaced by this single scalar: the **summed turn angle of the probe path**.

This is NOT physically accurate UTD. It is a **heuristic that produces the right qualitative behavior**: more bending → more high-frequency attenuation. Compare with Tsingos 2001 which actually finds real geometric edges via beam tracing and computes UTD with real wedge parameters.

---

## 8. Accuracy vs alternatives

| Aspect | Steam Audio approach | "True" UTD (Tsingos) | Wave-based (Project Acoustics) |
|---|---|---|---|
| Portal/edge detection | None — implicit in probe graph | Beam tracing, real geometric edges | None — solves wave equation |
| Diffraction geometry | Path bend angle (scalar) | Per-edge wedge, source pos, receiver pos | Direct from wave field |
| Frequency dependence | UTD magnitude per band | UTD per band with real wedge params | Natural from simulation |
| Low-freq accuracy | Poor (geometric) | Medium (geometric approx) | Good (if grid ≥ 3 pts/λ) |
| Resonance / room modes | Missing | Missing | Captured |
| Baking cost | Fast (Dijkstra on visibility graph) | Medium (beam trace) | Very slow (ARD wave solve) |
| Runtime cost | Sub-ms (table lookup + SH project) | ms (beam re-trace or cache) | Sub-ms (parameter lookup) |
| Memory | N² int32 + unique paths | Scene-sized beam tree | MB-GB grid |

---

## 9. Concrete numbers (from guide.html [18] and code)

- Typical `spacing` = **2 m**, `height` = **1.5 m** → one probe every 4 m² of floor.
- `numSamples` = 1 → single-ray visibility (fastest); higher for more robust volumetric tests.
- `radius` = probe sphere radius for volumetric visibility checks (often equals `spacing`).
- `threshold` = fraction in [0, 1]: e.g., 0.5 means ≥50% of valid sample pairs must be unoccluded to call two probes visible.
- `visRange` = bake-time max pairwise distance to attempt edge creation (e.g., 50 m).
- `visRangeRealTime` = pruned range for runtime visibility graph (smaller, e.g., 10 m).
- `pathRange` = Dijkstra cutoff (e.g., 100 m) — further probes have no baked path.
- `pathingOrder` (SH order) = 0, 1, 2, or 3.
- `kMaxPaths = 64` (runtime cap).
- `kMaxProbesPerBatch = 8` (interpolation neighborhood per query point).

---

## 10. Key takeaways for a voxel-based clone

1. **You do NOT need explicit portals.** A probe visibility graph implicitly captures every acoustic bottleneck — as long as your probes are dense enough to have several on each side of every opening.
2. **The "portal" concept is replaced by graph edges through free space.** Whether a wall blocks sound is answered by whether `scene.isOccluded(probeA, probeB)` returns true in your voxel scene.
3. **Diffraction is faked via path-bend-angle + UTD(n=2 flat wedge).** This is cheap and sounds plausible but is physically wrong for true wedge geometries. Acceptable for games.
4. **Probe generation = dense grid on every floor.** No cleverness. The user places an OBB; Steam Audio drops a grid.
5. **Baking = Dijkstra all-pairs shortest paths, O(N³) worst case but O(N² log N) with priority queue.**
6. **Storage trick**: only `firstProbe / lastProbe / probeAfterFirst / probeBeforeLast + distance + deviation` per pair. Full path reconstruction via chained lookups if needed.
7. **Runtime**: lookup path for nearest source-probe × listener-probe pair → compute virtual source → SH project. That's the whole query.

---

## 11. Files/functions for porting reference

| Concern | File | Key symbol |
|---|---|---|
| Probe placement | `probe_generator.cpp` | `ProbeGenerator::generateUniformFloorProbes` |
| Spatial index | `probe_tree.cpp` | `ProbeTree::getInfluencingProbes` |
| Visibility graph | `path_visibility.cpp` | `ProbeVisibilityTester::areProbesVisible`, `ProbeVisibilityGraph` |
| Dijkstra / A* | `path_finder.cpp` | `PathFinder::findAllShortestPaths`, `PathFinder::findShortestPath` |
| Path compression | `path_data.h/.cpp` | `SoundPath`, `BakedPathData` |
| Diffraction math | `deviation.cpp` | `DeviationModel::utdDeviation` |
| Runtime orchestrator | `path_simulator.cpp` | `PathSimulator::findPaths` |
| DSP application | `api_path_effect.cpp` | `PathEffect::apply` |
