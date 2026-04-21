# UTD and Deviation-Based Diffraction in Steam Audio

- **Type**: Synthesis (Steam Audio source + Tsingos 2001 references + UTD literature)
- **Fetched**: 2026-04-21
- **Project**: acoustic-portal-baking
- **Relevant to**: Q7, Q8, Q9
- **Sources drawn from**:
  1. [deviation.h (Steam Audio)](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/deviation.h)
  2. [deviation.cpp (Steam Audio)](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/deviation.cpp) — contains explicit reference `http://www-sop.inria.fr/reves/Nicolas.Tsingos/publis/sig2001.pdf`
  3. [path_data.cpp (SoundPath::deviation)](https://raw.githubusercontent.com/ValveSoftware/steam-audio/master/core/src/core/path_data.cpp)
  4. [Tsingos, Funkhouser, Ngan, Carlbom (SIGGRAPH 2001) — Modeling Acoustics in Virtual Environments Using the Uniform Theory of Diffraction](https://pixl.cs.princeton.edu/pubs/Tsingos_2001_MAI/index.php)
  5. [Funkhouser et al. — A Beam Tracing Method for Interactive Architectural Acoustics (JASA 2004)](https://europepmc.org/article/MED/15000186)
  6. [Fast UTD-Based Method for Multiple Acoustic Diffraction over Obstacles (MDPI Symmetry)](https://www.mdpi.com/2073-8994/12/4/654/xml)
  7. [UTD / beam tracing survey references](https://www.researchgate.net/publication/2397824)

---

## 1. UTD Theory in ~1 page

### The physics problem

When a sound ray hits an **edge** (a 2D ridge on a 3D surface, e.g., a doorway frame), geometrical acoustics (GA) predicts a hard shadow behind the edge. Real sound diffracts around the edge — the lower the frequency, the more strongly. GA alone yields physically wrong silence.

The **Geometrical Theory of Diffraction (GTD)** (Keller 1962) extended GA by treating each illuminated edge as a **secondary source**, emitting diffracted rays. Given an incident ray hitting an edge at angle `β₀` relative to the edge, the diffracted rays form the **Keller cone** — a cone of half-angle `β₀` around the edge, each ray on that cone being a valid diffracted direction.

```
    incident ray
         \
          \  angle β₀
           \|
  edge ─────●──────
           /|\    Keller cone: all rays
          / | \   diffracted at same β₀
         /  |  \  relative to edge
```

GTD's original diffraction coefficient `D(φ,φ')` goes **infinite** at shadow/reflection boundaries (the "transition regions"), making it unusable near those directions. **UTD** (Kouyoumjian & Pathak 1974) fixes this by multiplying `D` with a **Fresnel transition function** `F(x)` that smoothly bridges the singular limits.

### UTD diffraction coefficient for a wedge

For a wedge of exterior angle `nπ` with incident angle `φ'` and diffracted angle `φ`, the scalar UTD diffraction coefficient is:

```
           -exp(-jπ/4)
  D = ──────────────────── · [ F_term_incident_shadow
        2n·√(2πk)·sin(β₀)         + F_term_reflection_shadow
                                   + F_term_opposite_incident
                                   + F_term_opposite_reflection ]
```

Each F_term has the form:
```
F_term = cot( (π ± (φ ± φ')) / (2n) ) · F_fresnel( k·L·a(β) )
```

where:
- `k = 2π/λ` is the wavenumber
- `L` = "distance parameter" combining source-to-edge and edge-to-receiver distances (spherical incidence: L = s·s' / (s+s'))
- `a(β)` = argument of the Fresnel transition function, encoding angular proximity to a shadow boundary
- `F(x)` = complex Fresnel transition function: smoothly goes from 1 (far from boundary) to 0 (at boundary)
- The four cotangents correspond to the four shadow/reflection boundaries of the wedge

### Fresnel transition function approximation [2]

Steam Audio's implementation uses a standard piecewise approximation (valid for games):
```
        ┌ √(πx)·(1 − √x/(0.7√x + 1.2)) · exp(jπ/4 · √(x/(x+1.4)))    if x < 0.8
F(x) = ┤
        └ (1 − 0.8/(x+1.25)²) · exp(...)                             if x ≥ 0.8
```

### Frequency dependence

`k = 2πf/c`. Higher frequency → larger k → `F(kLa)` saturates closer to 1 (less diffraction effect) → diffracted field weaker. Lower frequency → smaller k → larger diffraction. This matches intuition: you hear the subwoofer through the closed door, not the cymbal.

`D ∝ 1/√k ∝ 1/√f` in magnitude far from transition regions → low-frequency favoritism.

### Limitations of UTD

- **Infinite edge assumption** — the formula comes from a 2D (half-plane) canonical problem, extended to 3D via Keller cone. For finite edges, you need to multiply by a finite-edge correction (e.g., Pierce's shadow function or line integral over the edge).
- **High-frequency asymptotic** — UTD breaks down when wavelength is comparable to or larger than the edge length (low-frequency, small openings).
- **Only first-order diffraction per edge**. Multi-edge diffraction handled by chaining UTD coefficients — gives approximate results but can be inaccurate for tight corners.
- **Wedge-shape required** — for curved surfaces, need caustic correction factors (not in Steam Audio).

### BTM (Biot-Tolstoy-Medwin) alternative

BTM uses a time-domain Huygens-style integral over the edge. Exact for infinite wedges of arbitrary angle; inherently handles finite edges via integration bounds. More expensive than UTD but physically correct at low frequencies. Steam Audio does **not** use BTM.

---

## 2. Tsingos 2001 — the "real" UTD pipeline [4]

Tsingos et al. published the seminal paper combining beam tracing with UTD for real-time virtual environments. This is the algorithmic template for "doing UTD correctly":

### Pipeline

1. **Precompute a beam tree from the source**:
   - Trace reflection beams through BSP of scene polygons
   - Track **illuminated edges** (edges where the beam hits a wedge) and spawn diffracted sub-beams from them
   - Each diffracted sub-beam = Keller cone for that (edge, incidence) combination
   - Recurse up to max order (e.g., 3 reflections + 2 diffractions)

2. **Prune the beam tree**:
   - Priority-driven: keep paths with highest estimated energy
   - Discard paths where attenuation drops below audibility threshold

3. **Query at runtime (receiver position)**:
   - Walk beam tree, find beams containing receiver
   - For each such beam, reconstruct the ray path (source → edge₁ → edge₂ … → receiver)
   - Compute per-edge UTD coefficient with **actual wedge geometry, wedge angle n, real L parameters, real β₀**
   - Accumulate per-band magnitude → impulse response

4. **Shadow-region approximation**:
   - Only compute diffraction when direct path is **occluded**
   - Saves enormous compute in LOS cases

### Concrete data structures (from 2001 paper and Funkhouser 2004 [5])

- **Beam tree**: tree of convex beams, each with (source polygon, reflections so far, diffractions so far, transmission so far)
- **Edge list per polygon**: pre-extracted silhouette edges of architectural geometry
- **Visibility cells**: precomputed cells from BSP; each cell stores its beam tree
- **UTD coefficient table**: evaluated per-path at runtime, 4 bands typical

### Scale

- Architectural scenes ~10-1000 polygons
- Beam tree traversal O(N·k^d) where d = max depth (~5)
- Runtime query ~10ms on 2001 hardware

---

## 3. Steam Audio's radical simplification — "Fake UTD" [1][2][3]

### What Steam Audio does instead

Steam Audio's `DeviationModel::utdDeviation(angle, band)` uses the UTD formula **with these hard-coded surrogate parameters**:

```cpp
const float n = 2.0f;       // half-plane wedge (flat edge)
const float alpha_i = 0.0f; // incident angle fixed at zero
const float L = 0.05f;      // fixed distance parameter, 5cm
float alpha_d = alpha_i + PI + angle;   // "diffracted angle" = π + path-bend-angle
```

Then run the standard UTD coefficient with these inputs and return `|D|` per band.

### Where does `angle` come from?

Not from real geometric edge analysis. Instead, `SoundPath::deviation` computes [3]:

```python
def deviation(source, probes, probePath, listener):
    """Sum of turn angles at each probe along the path."""
    total = 0
    for i in range(1, len(probePath.nodes) - 1):
        prev_dir = unit(probes[path[i]].center - probes[path[i-1]].center)
        next_dir = unit(probes[path[i+1]].center - probes[path[i]].center)
        total += angle_between(prev_dir, next_dir)
    # Add junction angles at endpoints
    total += angle_between(source_dir, first_interior_dir)
    total += angle_between(last_interior_dir, listener_dir)
    return total
```

Path turns 90° sharply at one probe? `angle ≈ π/2`. Path is nearly straight? `angle ≈ 0`.

### Why this works (kind of)

Tsingos UTD expects `angle ≈ π` at the actual shadow boundary behind an edge. Steam Audio reinterprets "path deviation" as a stand-in for that boundary displacement. Larger deviation → deeper into acoustic shadow → more high-frequency attenuation.

Fixed `L = 0.05` m eliminates the source-to-edge/edge-to-receiver distance dependence. Fixed `n = 2.0` (flat half-plane) ignores real wedge angles.

Result: **the EQ shape per band is a function of one scalar — the path bend angle**. It can be tabulated in advance (256 discrete angles × N bands = a tiny LUT) and looked up per source.

### Normalization [2]

`calcEQForPaths` normalizes by the UTD value at `angle=1e-8` (near-straight):
```cpp
refTerm = utdDeviation(1e-8, band)
eqGains[band] += weight * overallGain * (utdDeviation(angle, band) / refTerm)
```

So the EQ represents **relative** attenuation vs. a straight path. At `angle = 0`, all bands equal 1.0 (no attenuation). At larger angles, high-band terms drop faster than low-band → low-pass shape.

### What's lost vs. real UTD

| Feature | Real UTD (Tsingos) | Steam Audio |
|---|---|---|
| Per-edge wedge angle | real | hardcoded n=2 |
| Source-edge distance | real | hardcoded L=0.05 |
| Edge-receiver distance | real | hardcoded L=0.05 |
| Multiple edges in chain | multiplied coefficients | summed angles → single coefficient |
| Edge finiteness correction | needed separately | irrelevant (no edges) |
| Frequency-dependent direction | Keller cone per edge | single average direction |
| Compute cost per path | ms | μs |

### Why Valve chose this trade-off

1. **No edge extraction step** in baking → trivially works on voxel scenes, BSP scenes, or any mesh.
2. **Probe graph is a natural data structure** for open-world scene streaming — probes move around with the level.
3. **Perceptually acceptable**: For players with headphones in a FPS, the difference between "sound through doorway correctly bent by 31°" and "sound arriving with 31° of low-pass EQ" is subtle. Valve validated this during Half-Life: Alyx development.
4. **Sub-ms runtime cost** fits well within a game audio thread budget.

---

## 4. Offline diffraction bake — what actually gets stored

Steam Audio stores **no per-edge data**. Only:
- Visibility graph edges (probe↔probe with Euclidean distance cost)
- Per-pair `SoundPath` with compressed representation (4 int16 probe indices + 2 floats: `distanceInternal`, `deviationInternal`)

At runtime, the angle used for UTD is read from the `SoundPath` or reconstructed on the fly from the probe chain.

**Storage for 1000 probes**:
- Visibility graph: ~sparse O(N·k) = ~100 KB for k~20 edges/probe
- `mBakedPathRefs` (N×N `SoundPathRef`): 1000² × 4 bytes = 4 MB
- `mUniqueBakedPaths` (deduplicated paths): ~1-10 MB depending on topology

Total: **5-20 MB for a 1000-probe scene**. No explicit portal/edge data anywhere.

---

## 5. Runtime diffraction query

```python
def findPaths(source, listener, scene, bakedPathData, probes):
    # 1. Try direct LOS first
    if not scene.isOccluded(source, listener):
        emit_direct_path(source, listener)
        return

    # 2. Find source probes and listener probes (influence-sphere lookup)
    source_neighborhood = probeTree.getInfluencingProbes(source)
    listener_neighborhood = probeTree.getInfluencingProbes(listener)

    # 3. For each pair, look up baked path
    paths = []
    for sp in source_neighborhood:
        for lp in listener_neighborhood:
            sp_weight = sp.weight
            lp_weight = lp.weight
            sound_path = bakedPathData.lookupShortestPath(sp.index, lp.index)

            # 4. Validate (optional) — re-check segment visibility
            if enableValidation and isPathOccluded(sound_path, scene):
                if findAlternatePaths:
                    sound_path = pathFinder.findShortestPath(scene, sp.index, lp.index)
                else:
                    continue

            paths.append((sound_path, sp_weight * lp_weight))

    # 5. Accumulate SH and EQ across all paths
    for path, weight in paths:
        distance = path.distance(probes, source, listener)
        deviation = path.deviation(probes, source, listener)
        virtual_src = path.toVirtualSource(probes, source, listener)
        dir = unit(virtual_src - listener)

        # SH projection (spatial direction)
        gain = weight * distanceAttenuationModel.evaluate(distance)
        SH.projectAndAccumulate(dir, order, gain, coeffs)

        # Per-band EQ (frequency-dependent diffraction attenuation)
        for band in range(numBands):
            eq[band] += weight * overallGain * deviationModel.evaluate(deviation, band) / refTerm[band]

    # 6. Apply DSP: mono input → convolve with SH basis → EQ → binaural
```

**Total runtime cost**: O(|source_nbd| · |listener_nbd|) = O(k²) with `kMaxProbesPerBatch = 8`. Typical k~4, so 16 pair lookups. Each lookup is an array index and a few float ops. Total < 100 μs per source.

---

## 6. Key takeaways for a voxel clone

If your goal is Steam-Audio-style pathing on voxels, you DON'T need:
- Edge extraction from voxels
- Per-portal storage
- Beam tracing
- 3D funnel algorithms through edge chains
- Portal aperture polygons

You DO need:
- Probe placement (floor-grid or any reasonable sampling)
- Voxel-based visibility test (ray marching through voxel grid)
- Visibility graph build (Dijkstra-shaped pair enumeration)
- Dijkstra all-pairs shortest paths
- UTD LUT parameterized by path-bend-angle
- Runtime path lookup + SH/EQ DSP

This is ~2000 lines of well-structured C++ (basically what Steam Audio's `path_*.cpp` + `probe_*.cpp` amount to).

If your goal is **physically-accurate UTD** (e.g., for architectural modeling), follow Tsingos 2001 — you'll need beam tracing over a BSP, silhouette-edge extraction per polygon, real per-edge UTD evaluation. Much more code, much slower bake. Not usually worth it for games.
