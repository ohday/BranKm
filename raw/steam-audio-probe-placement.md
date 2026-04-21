# Steam Audio Probe Placement and Visibility-Graph Construction

- **Type**: Synthesis (aggregated from Steam Audio source code and documentation)
- **Fetched**: 2026-04-21
- **Project**: acoustic-portal-baking
- **Relevant to**: Q5, Q6
- **Sources drawn from**:
  1. [probe_generator.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_generator.h)
  2. [probe_generator.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_generator.cpp)
  3. [probe_tree.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_tree.h)
  4. [probe_tree.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/probe_tree.cpp)
  5. [path_visibility.h](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_visibility.h)
  6. [path_visibility.cpp](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_visibility.cpp)
  7. [Steam Audio C API Guide](https://valvesoftware.github.io/steam-audio/doc/capi/guide.html)

---

## Probe Placement Strategy — It's Basically a Grid

Contrary to intuition (which would suggest Poisson-disk, medial axis, or SDF-adaptive sampling), Steam Audio uses a **uniform 2D grid on each detected floor**.

### Algorithm (`generateUniformFloorProbes`) [2]

```python
# Pseudocode from the C++ source
def generate_uniform_floor_probes(scene, obb, spacing, height):
    # OBB defines an oriented box in world space
    sx = length(obb.column0)   # X extent
    sz = length(obb.column2)   # Z extent
    down = normalize(obb @ Vec4(0, -1, 0, 0))

    # How many probes along each horizontal axis
    nx = floor(sx / spacing) + 1
    nz = floor(sz / spacing) + 1

    # Residual offset centers the grid inside the OBB
    res_x = (sx - (nx - 1) * spacing) / 2
    res_z = (sz - (nz - 1) * spacing) / 2

    probes = []
    for i in range(nx):
        for j in range(nz):
            # Normalized OBB coordinate in [-0.5, 0.5]
            u = -0.5 + (i * spacing + res_x) / sx
            v = -0.5 + (j * spacing + res_z) / sz
            w = +0.5   # start at top face of OBB
            origin = obb @ Vec4(u, w, v, 1.0)

            # Cast rays downward from this column, finding EVERY floor below
            distance_from_floor = obb_vertical_extent
            while distance_from_floor > 0:
                hit = scene.closest_hit(Ray(origin, down),
                                        min_t=height,
                                        max_t=distance_from_floor + height)
                if not hit: break
                # Place probe at hit point + height (ear level)
                probe_pos = origin + down * (hit.distance - height)
                probes.append(Probe(influence=Sphere(probe_pos, radius=spacing)))
                # Continue searching below for multi-story
                origin = origin + down * (hit.distance + 0.01)   # kDownwardOffset
                distance_from_floor -= (hit.distance + 0.01)
    return probes
```

### What this means

- **Resolution is whatever `spacing` the artist set.** Typical: 2 m.
- **Influence radius = spacing.** Adjacent influence spheres just touch (don't overlap much).
- **Every floor is sampled.** Stairs, balconies, mezzanines, basement — anywhere a downward ray hits solid geometry gets a floor of probes.
- **Completely ignores local geometry complexity.** No denser probes near doorways. No sparser probes in big open spaces.

### The `Octree` generation mode is a lie

The `ProbeGenerationType` enum declares `Centroid`, `UniformFloor`, and `Octree`, but `ProbeGenerator::generateProbes` throws `Exception(Status::Initialization)` for anything other than the first two. **Adaptive octree probe placement is NOT implemented in the open-source release.**

### Consequence for acoustic quality

The density must be fine enough that **every doorway has at least 1-2 probes on each side** visible through the aperture. If the door is 0.9 m wide and spacing is 2 m, you might get unlucky and not have a probe that sees through the opening. Raising density is the universal fix; no algorithmic cleverness.

---

## Spatial Index for Probe Queries — `ProbeTree` [3][4]

### Structure

The header comment calls it a k-d tree, but the implementation is a **top-down median-split BVH over probe influence AABBs**. The tree is flat-array-packed:

```cpp
struct ProbeTreeNode {
    Box box;              // AABB covering all leaves in this subtree
    // Bit-packed metadata hidden in SIMD vector padding:
    //   box.minCoordinates.w (4th float's int reinterpretation):
    //     bits [0..1]  = split axis (0=X, 1=Y, 2=Z, 3=LEAF sentinel)
    //     bits [2..31] = child offset (for internal) OR probe index (for leaf)
    //   box.maxCoordinates.w = split coordinate
};

class ProbeTree {
    Array<ProbeTreeNode> mNodes;   // 2*N-1 nodes for N probes
};
```

Left child at `this[data() >> 2]`, right at `this[(data() >> 2) + 1]` (contiguous).

### Construction

Iterative top-down (stack of tasks), median split:

```python
def build(probes):
    # Each task: (nodeIndex, leafRange, leftChildIndex)
    stack = [(0, [0..N-1], 1)]

    while stack:
        task = stack.pop()

        if len(task.range) == 1:
            # LEAF
            node = mNodes[task.nodeIndex]
            node.box = AABB of probes[task.range[0]].influence
            node.splitAxis = 3   # leaf sentinel
            node.probeIndex = task.range[0]
        else:
            # INTERNAL
            node.box = union of all probe AABBs in task.range

            # Choose split axis = axis of node.box with largest extent
            axis = node.box.extents().indexOfMaxComponent()

            # Sort range by centroid on chosen axis, split at median
            sort(task.range, key=lambda i: probes[i].influence.center[axis])
            mid = len(task.range) // 2
            node.splitCoordinate = probes[task.range[mid]].center[axis]
            node.splitAxis = axis
            node.childOffset = task.leftChildIndex

            # Push children tasks
            stack.push(RIGHT task with range[mid:])
            stack.push(LEFT task with range[:mid])   # processed next
```

### Query: `getInfluencingProbes(point, maxInfluencingProbes)` [4]

**Not a nearest-neighbor query. Returns all probes whose influence sphere contains the point.**

```python
def getInfluencingProbes(point, probes, maxInfluencingProbes):
    probeIndices = [-1] * maxInfluencingProbes
    count = 0
    stack = [root]

    while stack and count < maxInfluencingProbes:
        node = stack.pop()

        if not node.box.contains(point):   # AABB prune
            continue

        if node.isLeaf():
            # Sphere containment test (real spheres, not AABBs)
            if probes[node.probeIndex].influence.contains(point):
                probeIndices[count] = node.probeIndex
                count += 1
        else:
            # Visit near-child first (same side of split plane as point)
            near, far = (left, right)
            if point[node.splitAxis] > node.splitCoordinate:
                near, far = (right, left)
            stack.push(far)
            stack.push(near)

    return probeIndices[:count]
```

`kProbeLookupStackSize` bounds the traversal stack to keep it on the thread stack.

### Weight computation

`ProbeTree` returns only the set of influencing probe indices. **`ProbeNeighborhood::calcWeights(point)`** (declared in `probe_batch.h`) computes the per-probe interpolation weights based on position within each influence sphere. The specific interpolation formula is not in the tree code itself.

Typical approach for sphere influence: weight ∝ `max(0, 1 − ||point − center|| / radius)`, normalized across contributing probes. (This is common practice; the exact Steam Audio formula lives in `probe_batch.cpp` which we couldn't directly fetch.)

---

## Visibility Graph Construction — `ProbeVisibilityGraph` [5][6]

### Data structure

```cpp
struct AdjacencyListEntry { int index; float cost; };

class ProbeVisibilityGraph {
    vector<vector<AdjacencyListEntry>> mAdjacent;   // adjacency list
    std::atomic<int> mNumJobsRemaining;
};
```

`mAdjacent[i]` = list of `(j, euclideanDistance)` pairs — all probes `j` that are mutually visible from `i`.

### Construction algorithm (parallel)

```python
def build_visibility_graph(scene, probes, vis_tester, radius, threshold, visRange, num_threads):
    # Divide rows of mAdjacent across jobs
    jobs = split_rows(0..N, num_threads)

    parallel_for job in jobs:
        for i in job.rows:
            for j in range(0, i):   # upper-triangle only (i > j)
                if vis_tester.areProbesTooFar(probes, i, j, visRange):
                    continue
                if not vis_tester.areProbesVisible(scene, probes, i, j, radius, threshold):
                    continue
                dist = length(probes[i].center - probes[j].center)
                mAdjacent[i].push_back({j, dist})

    # LAST job to finish (tracked via mNumJobsRemaining atomic):
    symmetrize()   # add reverse edges j->i for every i->j
```

### Visibility test — two modes [6]

**Point-to-point mode** (`numSamples <= 1` or `radius <= 0`):
```cpp
return !scene.isOccluded(probes[i].center, probes[j].center);
```
Single ray cast between probe centers.

**Volumetric mode** (`numSamples > 1` and `radius > 0`):
Probes are modeled as spheres of given `radius`. Pre-generate `numSamples` random points inside a unit sphere (via `Sampling::generateSphereVolumeSamples`), then:

```python
def areProbesVisible_volumetric(scene, i, j, radius, threshold):
    from_center = probes[i].center
    to_center = probes[j].center
    samples = mSamples   # pre-generated unit-sphere volume samples

    num_visible = 0
    for a in samples:
        from_sample = from_center + a * radius
        # Skip if sample point is inside geometry (ray from center to sample)
        if scene.isOccluded(from_center, from_sample): continue

        for b in samples:
            to_sample = to_center + b * radius
            if scene.isOccluded(to_center, to_sample): continue

            if not scene.isOccluded(from_sample, to_sample):
                num_visible += 1

            # Early exit if enough visible samples found
            if num_visible / len(samples) >= threshold:
                return True

    return False
```

Worst case O(numSamples²) rays per pair; early exit helps.

**`threshold`** is the minimum fraction of visible sample pairs. Note: it divides by `len(samples)`, not `len(samples)²`, so an achievable threshold is roughly `1.0 / numSamples` per surviving outer sample — effectively a lenient check. This is a deliberate design choice to call probes "mostly visible" if any reasonable fraction of rays get through.

### Asymmetric range

For multi-story buildings, `asymmetricVisRange=true` + a `down` vector lets `areProbesTooFar` strip the vertical component of displacement along `down`:
```cpp
disp = probes[i].center - probes[j].center
if asymmetric: disp -= dot(disp, down) * down    // remove vertical
return length(disp) > visRange
```
Horizontal neighbors pass at any vertical separation; vertical neighbors still limited.

### Serialization

Only `j < i` edges written (lower triangle), reverse reconstructed on load. Memory per edge: 8 bytes (int + float).

### Post-bake operations

- `prune(visRange)` removes edges beyond a range. Typically called with `visRangeRealTime < visRange` after baking, so the baked visibility graph stays dense but the runtime-loaded version is sparse.
- `updateCosts(probeBatch)` refreshes edge costs if probes move (topology preserved).
- `hasEdge(from, to)` — linear scan of `mAdjacent[from]` (not optimized for point query — assumes edges are usually queried in batch).

---

## Practical configuration [7]

| Parameter | Typical value | Purpose |
|---|---|---|
| `spacing` | 2.0 m | Probe density along X/Z |
| `height` | 1.5 m | Probe height above floor (ear level) |
| `radius` | = spacing | Probe sphere radius |
| `numSamples` | 1 (fast) or 16-64 (robust) | Visibility rays per pair |
| `threshold` | 0.1 – 0.5 | Visible-fraction threshold |
| `visRange` | 50 m (bake) | Max pairwise distance for edge creation |
| `visRangeRealTime` | 10 m | Runtime graph range after pruning |
| `pathRange` | 100 m | Dijkstra cutoff (max path length stored) |
| `asymmetricVisRange` | true for multi-story | Horizontal > vertical range |

For a 50 m × 50 m × 3 m (single-story) building with 2 m spacing:
- ~625 probes
- Visibility graph pairs: ~625² / 2 ≈ 195k pairs
- At ~1 ms/pair for 64-sample volumetric test: ~200 seconds serial bake
- Storage: path refs ≈ 4 × 625² ≈ 1.5 MB; graph edges sparse ≈ 100 KB
