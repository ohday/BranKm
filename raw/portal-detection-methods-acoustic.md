# Automatic Acoustic Portal Detection — Methods Survey for Voxel Scenes

- **Type**: Synthesis (survey for Q2, Q3 from the perspective "we have a voxel grid + SDF, and we want portals — but maybe we don't need them at all")
- **Fetched**: 2026-04-21
- **Project**: acoustic-portal-baking
- **Relevant to**: Q1, Q2, Q3, Q4
- **Sources drawn from**:
  1. [Portal rendering (Wikipedia)](https://en.wikipedia.org/wiki/Portal_rendering)
  2. [Teller & Séquin — Visibility Preprocessing (overview)](https://pixl.cs.princeton.edu/pubs/Tsingos_2001_MAI/index.php) [and search results on Teller-Séquin cells-and-portals]
  3. [Morphological opening + connected components for narrow passage detection](https://www.freepatentsonline.net/y2012/0219198.html)
  4. [3D Curve Skeleton for narrow passage detection (Biswas 2018)](https://www.sciencedirect.com/science/article/pii/S0020025518304614)
  5. [Epsilon-shapes for thin feature detection (Aichholzer et al.)](https://www.researchgate.net/publication/316505389)
  6. [Existing shared raw: chokepoint-detection.md, topological-persistence.md, ao-openness-field.md, gudhi-cubical-persistence.md, cells-portals-bsp.md]
  7. [Dolby Grid Pathfinding Diffraction (US20230188920)](https://patents.justia.com/patent/20230188920)
  8. [Raghuvanshi Dynamic Portal Occlusion](https://arxiv.org/abs/2107.11548)

---

## THE FRAMING QUESTION

The user's brief assumes we need explicit portal surfaces. **The Steam Audio evidence (see `steam-audio-pathing-source-breakdown.md`) shows explicit portals are optional** — a dense probe graph implicitly encodes all portal information through pairwise visibility.

This note covers: **if you still want explicit portals** (e.g., for dynamic closure modulation à la Raghuvanshi 2021, or for debugging visualization, or for a hybrid reverb model), what are the options on a voxel grid?

---

## Menu of methods

```
Portal candidate extraction from a voxel grid with SDF
│
├── 1. MORPHOLOGICAL OPENING differencing ────── simple, robust, diameter-parametric
├── 2. SDF LOCAL-MAX RIDGE / SADDLE ─────────── medial axis, ridge lines at openings
├── 3. MIN-CUT on voxel adjacency graph ──────── theoretically optimal, expensive
├── 4. SUBLEVEL-SET PERSISTENCE on SDF ───────── principled ranking, needs Gudhi
├── 5. REEB GRAPH of SDF ────────────────────── topology-aware, complex implementation
├── 6. 3D CURVE SKELETON pinch detection ────── medial axis + branch analysis
├── 7. AO / OPENNESS FIELD saddles ──────────── alternative scalar field
└── 8. CROSS-SECTION AREA along skeleton ────── post-process on skeleton
```

### Comparison table

| Method | Additional voxel fields needed | Accuracy on open/irregular | Cost | Implementation difficulty |
|---|---|---|---|---|
| 1. Morphological opening | none | good for rooms, poor for half-open | O(N) per structuring radius | easy |
| 2. SDF saddles | SDF + Hessian | medium (sensitive to noise) | O(N) | medium |
| 3. Min-cut | segmentation | excellent (global optimum) | O(N^2.5) | hard |
| 4. Persistence | SDF | excellent + ranked | O(N log N) | medium (Gudhi lib) |
| 5. Reeb graph | SDF | excellent topologically | O(N log N) | hard |
| 6. 3D curve skeleton | binary occupancy | good | O(N·iterations) | medium |
| 7. AO/openness saddles | new field | medium-good | O(N·#rays) | medium |
| 8. Cross-section on skeleton | skeleton + SDF | good | O(skel_length) | easy after skel |

---

## 1. Morphological Opening (simplest) [3][6]

### Idea

Opening = erosion followed by dilation:
```
Opened(A, B) = (A ⊖ B) ⊕ B
```
with a spherical structuring element of radius `r`. Narrow passages smaller than `2r` get eliminated in the erosion; the dilation restores the large rooms but cannot restore the filaments.

**Portals appear in the difference `A − Opened(A, B)`**: voxels that were in the original but absent after opening correspond to thin features, which are exactly the doorway/window regions.

### Algorithm

```python
def detect_portals_morphological(occupancy, r_min, r_max):
    """
    occupancy[i,j,k] = 0 (free) or 1 (solid)
    free = 1 - occupancy
    """
    portals = []
    for r in range(r_min, r_max):   # sweep structuring radius
        SE = ball(r)
        opened = closing(free, SE)   # eroding free, then dilating restores large cavities
        # Narrow passages vanish from `opened` but exist in `free`
        narrow = free & ~opened
        components = connected_components(narrow)
        for comp in components:
            # Each comp should lie on the interface between two large cavities
            aperture_radius = r
            portal = extract_portal_surface(comp, SE)
            portals.append((portal, aperture_radius))
    return portals
```

### Strengths
- Single parameter: structuring element radius (corresponds to minimum door width of interest)
- Fast with scipy.ndimage.binary_opening (or equivalent 3D implementations)
- Naturally rejects noise < r
- Handles non-Manhattan geometry without modification

### Weaknesses
- Needs multi-scale sweep (one radius misses tall/wide openings)
- Doesn't produce portal normal/plane directly — needs post-processing
- Fails for **large semi-open apertures** (a 5m-wide open mezzanine edge isn't "narrow" compared to the mezzanine itself)

### Recommended parameters for acoustic portals
- `r_min ≈ half of smallest acoustically interesting aperture` — typically 0.3 m (small vent) to 0.5 m (crawl space)
- `r_max ≈ half of largest aperture` — 1.5 m (double door)
- At 5 cm voxels: `r_min = 6`, `r_max = 30` voxels

---

## 2. SDF Ridge/Saddle Analysis [6]

### Idea

The signed distance field `ϕ` reaches **local maxima at medial axis** points (inscribed sphere centers). Along the medial axis, the value of `ϕ` is the local "radius of free space". An acoustic portal corresponds to a **local minimum of `ϕ` along the medial axis path** between two rooms — i.e., a **saddle point of `ϕ`** in 3D.

### Algorithm (voxel-based)

```python
def detect_sdf_saddles(sdf, occupancy):
    """Classify each voxel by Hessian eigenvalues of the SDF."""
    # Compute Hessian at each free voxel via central differences
    H = hessian(sdf)   # 3x3 matrix per voxel
    eigvals = eigenvalues(H)   # sorted λ1 <= λ2 <= λ3

    saddle_type_1 = (eigvals[0] < 0 and eigvals[1] > 0 and eigvals[2] > 0)  # 1-saddle
    saddle_type_2 = (eigvals[0] < 0 and eigvals[1] < 0 and eigvals[2] > 0)  # 2-saddle

    # Portal candidates are 1-saddles along high-curvature directions
    # (narrow passage = one negative eigval perpendicular to passage, two positive across)
    candidates = saddle_type_1 & (|eigvals[0]| > |eigvals[1]| * 2)

    # Portal normal = eigenvector of the negative eigenvalue
    for v in candidates:
        normal = eigvec_of_min(H[v])
        ...
```

### Strengths
- Gives portal **position + normal** in one pass
- Physically meaningful: saddle = the "pinch point" between two cavities
- Can threshold by saddle strength (|λ|)

### Weaknesses
- **Noise sensitive** — discrete SDF has staircase artifacts that produce spurious saddles; needs smoothing (Gaussian convolve SDF first)
- Saddle classification via Hessian is known to be fragile at pixel grids; topology-robust alternatives use persistent homology (below)
- Requires gradient + Hessian = significant per-voxel compute

---

## 3. Min-Cut on Voxel Adjacency Graph [6]

### Idea

Treat free voxels as graph nodes, 6-connectivity edges with cost = 1 (or inverse-SDF-weighted). For every pair of well-separated rooms, the **minimum s-t cut** is exactly the narrowest interface between them — by definition, a portal.

### Algorithm

```python
def detect_portals_mincut(occupancy, rooms):
    """rooms is a list of room seeds identified separately."""
    portals = []
    for (room_a, room_b) in pairs(rooms):
        # Source = all voxels in room_a, Sink = all voxels in room_b
        cut_edges = max_flow_min_cut(voxel_graph, room_a, room_b)
        # cut_edges forms a surface between rooms — the portal surface
        portals.append(cut_edges_to_polygon(cut_edges))
    return portals
```

### Strengths
- **Globally optimal** — provably narrowest cut
- Portal is exactly the minimum aperture surface
- Naturally handles multiple portals between same room pair (multiple min cuts)

### Weaknesses
- **Requires room labels first** — ill-posed as the user's pivot explicitly rejects segmentation
- Practical min-cut at voxel resolution is expensive: O(N^2.5) with Boykov-Kolmogorov
- Portals are voxel-grid-aligned surfaces; need post-processing for clean polygons

**Verdict: contradicts the pivot. Don't use.**

---

## 4. Sublevel-Set Topological Persistence on SDF [6]

### Idea

(See `topological-persistence.md`, `gudhi-cubical-persistence.md` in shared raw for full treatment.)

Sweep a threshold `t` over the SDF from large to small. At `t = +∞`, the sublevel set `{ϕ ≤ t}` has 0 components. As `t` drops, rooms appear as new components. When `t` drops to a doorway width, **two components merge into one** — this merge event is a "death" in persistent homology.

**Each merge event corresponds to exactly one portal.** The merge value `t_merge` equals the portal's aperture radius. The persistence lifetime `t_birth_of_smaller_room − t_merge` measures how significant the portal is.

### Algorithm

```python
import gudhi as gd

def detect_portals_persistence(sdf):
    cc = gd.CubicalComplex(
        dimensions=sdf.shape,
        top_dimensional_cells=-sdf.flatten()  # negate for sublevel set
    )
    persistence = cc.persistence()
    # persistence: list of (dimension, (birth, death)) pairs
    # Dimension 0 pairs are component-merge events = portals
    portals = []
    for dim, (birth, death) in persistence:
        if dim == 0 and death < float('inf'):
            aperture_radius = -death   # negated back
            significance = death - birth
            # Merge tree gives the voxels where the merge occurred
            merge_voxel = cc.get_merge_voxel(death)
            portals.append({
                'position': voxel_to_world(merge_voxel),
                'radius': aperture_radius,
                'significance': significance,
            })
    # Rank by significance, threshold
    portals.sort(key=lambda p: -p['significance'])
    return portals
```

### Strengths
- **Principled ranking** by persistence — can set a single threshold parameter (minimum portal significance) and get non-subjective filtering
- Handles multi-scale automatically
- Theoretically grounded (persistent homology)
- Gudhi library makes implementation relatively short

### Weaknesses
- Still doesn't produce portal normal/polygon directly — persistence gives a **voxel where merge happens**, not an oriented polygon
- Computation is O(N log N) but with large constants
- Merge voxel location is approximate — the actual portal surface passes near this voxel
- Post-processing needed to extract aperture polygon (e.g., cross-section of the merge region perpendicular to the gradient)

### Recommended workflow
1. Run Gudhi CubicalComplex on voxelized SDF
2. Filter by significance threshold (only keep portals with persistence > N × voxel_size)
3. For each merge voxel, compute local SDF gradient → portal normal
4. Slice the voxel region with the perpendicular plane → portal aperture polygon

---

## 5. Reeb Graph of SDF

### Idea

The **Reeb graph** of a scalar field tracks how **level sets** evolve as the threshold varies. Build it by sweeping the SDF: at each level, record connected components of the level set; track their merges/splits.

**Portal = edge in the Reeb graph where two components merge** (equivalent to persistence merges but with the full tracking graph, not just the ranked pairs).

### Use for portals

Each Reeb graph edge that ends in a merge corresponds to a portal. The portal's **aperture polygon** is the shared level set component just before merging.

### Strengths
- Gives **full level-set topology**, not just ranking — you know the rooms AND the portals between them
- Can select any level for aperture polygon extraction

### Weaknesses
- Complex implementation (practical 3D Reeb graph code is rare outside research)
- Reverts to having "rooms" as a byproduct — slightly against the pivot

---

## 6. 3D Curve Skeleton Pinch Detection [4]

### Idea

Extract a 1D **curve skeleton** of the free space (medial axis, thinned to curves). Portals appear as **branch points** or **narrow sections** along the skeleton.

### Algorithm sketch

```python
def detect_portals_skeleton(occupancy, sdf):
    # Parallel thinning of occupancy volume preserving topology
    skel = parallel_thinning_3d(1 - occupancy)
    # At each skeleton point, record the SDF value = local inscribed-sphere radius
    skel_points = [(v, sdf[v]) for v in skel]

    # Find local minima of sdf along skeleton path → portal candidates
    portals = []
    for branch in extract_branches(skel_points):
        for i, (v, r) in enumerate(branch):
            if is_local_min(r, branch, i, window=5):   # minimum along a 5-voxel window
                normal = skeleton_tangent_at(v)
                aperture_polygon = slice_free_space(v, normal, occupancy)
                portals.append((v, normal, aperture_polygon))
    return portals
```

### Strengths
- Gives portal + normal directly (skeleton tangent = portal normal approximation)
- Detailed aperture polygon via free-space cross-section
- Handles arbitrary geometry

### Weaknesses
- Curve skeleton extraction is nontrivial in 3D (iterative thinning, cavity detection)
- Branch-point detection heuristics introduce tuning knobs

---

## 7. AO / Openness Field Saddles

(See `ao-openness-field.md` in shared raw for scalar-field construction.)

Same idea as (2) SDF saddles, but using the **openness field** O (sum of ray distances in N directions) instead of SDF. O emphasizes **directional anisotropy** — narrow slots have low O in the across-slot direction but high O along the slot, giving stronger contrast for detection than isotropic SDF.

### Strengths
- Better discrimination of elongated corridors vs. portals
- Cross-section indicator directly available from opposite-direction ray pairs

### Weaknesses
- Requires ray casting in N directions per voxel — expensive precompute

---

## 8. Cross-Section Area Along Medial Axis

Hybrid of (6) and heuristic sizing.

```python
def detect_portals_cross_section(skeleton, occupancy, A_threshold):
    for v in skeleton:
        normal = tangent_at(v)
        slice = cross_section_at(v, normal, occupancy)
        area = count_free_voxels(slice) * voxel_size²
        if area < A_threshold:
            portals.append((v, normal, slice))
    return portals
```

Simplest classic approach. Works surprisingly well if skeleton quality is good.

---

## The acoustic wrinkle — wavelength-dependent portal threshold

Unlike physical "narrow passage" detection (medical, manufacturing), **acoustic** portals must be selected relative to the sound wavelengths of interest. A 10 cm gap is:
- Opaque to 1 kHz (λ ≈ 34 cm) — acts like sound barrier
- Partially transmissive at 250 Hz (λ ≈ 1.4 m) — diffracts strongly

### Selection rule (from Fresnel zone theory)

A gap smaller than roughly **λ/4** diffracts sound essentially completely (Huygens: the gap acts as a point source). A gap larger than roughly **4λ** is approximately a pinhole with geometric shadow. Between these, diffraction is partial.

For a game with audible bandwidth 20 Hz - 20 kHz:
- λ_min ≈ 1.7 cm (20 kHz)
- λ_max ≈ 17 m (20 Hz)

Practical perceptual audio uses 125 Hz - 4 kHz for most content:
- λ_min ≈ 8.5 cm
- λ_max ≈ 2.7 m

**Portals of interest thus range from ~5 cm apertures to ~3 m apertures**. This is a 60× scale range — motivates multi-scale methods (morphological opening across a radius sweep, or persistence with lifetime ranking).

### Schroeder frequency

For a room with volume V, the **Schroeder frequency**:
```
f_s ≈ 2000 · sqrt(T60 / V)    Hz
```
is the frequency above which modal density is high enough that statistical acoustics applies (geometric methods work well). Below f_s, modal resonance dominates (geometric methods fail — need wave-based).

Typical living room: V = 60 m³, T60 = 0.5 s → f_s ≈ 180 Hz. Steam-Audio-class methods lose accuracy below this.

---

## Recommendation for the voxel-based clone

**Primary path: follow Steam Audio's lead and SKIP explicit portal detection.** Dense probes + visibility graph implicitly encode every portal you could detect explicitly.

**Secondary path (for hybrid features): use morphological opening + connected components at 2-3 scales**. Cheapest, robust, handles irregular geometry. Use the portal locations only for:
- Dynamic closure modulation (doors that open/close at runtime à la Raghuvanshi 2021)
- Debug visualization
- Reverb region association (each enclosed "pocket" between portals gets its own reverb preset)

**Tertiary path (research / papers): persistence on SDF** for principled significance ranking. More work, but gives rankable portals and integrates with Gudhi library.

**Do NOT use**: min-cut (requires segmentation), Reeb graph (complex implementation, complete overkill for game audio).
