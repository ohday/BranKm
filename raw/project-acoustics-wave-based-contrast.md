# Project Acoustics (Microsoft Triton) — Wave-Based Bake as Contrast

- **Type**: Synthesis
- **Fetched**: 2026-04-21
- **Project**: acoustic-portal-baking
- **Relevant to**: Q11 (contrast), Q7 (what geometric+UTD misses)
- **Sources drawn from**:
  1. [Project Triton at Microsoft Research](https://www.microsoft.com/en-us/research/project/project-triton/)
  2. [Raghuvanshi & Snyder — Adaptive Rectangular Decomposition (ARD)](https://www.microsoft.com/en-us/research/publication/adaptive-rectangular-decomposition-spectral-domain-decomposition-approach-fast-wave-solution-complex-scenes/)
  3. [Raghuvanshi — Dynamic Portal Occlusion for Precomputed Interactive Sound Propagation (arXiv:2107.11548)](https://arxiv.org/abs/2107.11548)
  4. [Mehra, Raghuvanshi, Savioja, Lin, Manocha — GPU time-domain solver (2012)](https://doi.org/10.1016/j.apacoust.2011.05.012)
  5. [Microsoft ProjectAcoustics GitHub](https://github.com/microsoft/ProjectAcoustics)
  6. [Web searches on Raghuvanshi parametric encoding literature](https://www.microsoft.com/en-us/research/publication/parametric-wave-field-coding-for-precomputed-sound-propagation/)

---

## What Project Acoustics is, in one line

**A wave-equation solver (ARD) runs offline over a voxelized scene, producing per-(source, listener) impulse responses. These impulse responses are reduced to a handful of perceptual parameters per probe pair, compressed into an ACE file, and looked up at runtime to drive a standard DSP chain.**

It is the diametric opposite of Steam Audio's approach: replace clever data structures with brute-force physics, then compress aggressively.

---

## 1. Offline wave simulation — ARD [2][4]

### The wave equation being solved

Second-order scalar wave equation in 3D:
```
∂²p/∂t² = c² · ∇²p + f(x,t)
```
with boundary conditions on scene geometry (absorption, transmission per material).

### Why not FDTD

FDTD (Finite-Difference Time-Domain) uses a uniform grid with ≥10 points per wavelength to keep numerical dispersion low. For a max frequency of 500 Hz, that's grid spacing ≤ 343/500/10 ≈ 7 cm. A 100m×100m×10m building = 10⁵ m³ at 7³ cm³ cells = **~300 million cells**. Times hundreds of thousands of time steps. Prohibitive.

### ARD partition-and-solve

Raghuvanshi & Snyder's **Adaptive Rectangular Decomposition** decomposes the scene into **rectangular (cuboidal) sub-volumes** [2]. Inside each cuboid:

- With **spatially-constant speed of sound**, the wave equation has an **analytic spectral solution** via a 3D Discrete Cosine Transform (DCT).
- Basis functions are eigenmodes of the cuboid: `cos(k_x x) cos(k_y y) cos(k_z z)` for integer wavenumbers.
- Time-stepping each mode's amplitude is a trivial exact update — no numerical dispersion within the cuboid.

Adjacent cuboids are coupled at their shared faces by **finite-difference interface operators** (typically 6th-order stencils). Only these interfaces carry numerical error.

### Performance gain

The analytic interior permits grid spacing as coarse as **3 points per wavelength** (λ/3) — vs. FDTD's λ/10.

For 500 Hz target:
- FDTD spacing ≈ 7 cm → full grid ~300M cells
- ARD spacing ≈ 23 cm → full grid ~8M cells (37× savings just from grid)
- Plus ARD's DCT time-step is exact → can use larger time steps than FDTD CFL

Published speedups: **10-100× over FDTD** for equivalent accuracy, per Raghuvanshi & Snyder JASA 2012.

MPARD (Morales 2017) scales ARD to 16k cores, enabling 10 kHz simulations.

### What gets simulated

For each source location:
- Run wave simulation for duration ≥ T60 (e.g., 2 seconds of real time for a typical room)
- Record pressure at all probe locations (listeners) throughout simulation
- Output: **impulse response (IR) per (source, listener) pair**

For N probes as sources and M probes as listeners: N × M impulse responses.

---

## 2. Probe placement

Project Acoustics docs (including the plugin) describe probes as an **adaptive placement** strategy, not a uniform grid. The specifics (as of 2022.1 version [5]):

- User marks **navigable space** in the game level (area where listeners/sources can be)
- Automatic voxelization at some fixed resolution (reportedly ~25 cm for typical scenes)
- Probes placed on a grid but pruned to only the navigable region
- In the 2022.1 "voxel-free interpolator" update, Project Acoustics moved away from voxel-based runtime queries — voxels are kept only for debug visualization. The runtime interpolator now queries probes directly.

The key difference from Steam Audio: Project Acoustics probes are **pairwise source-listener combinations** in the bake (O(N²) simulations), whereas Steam Audio probes are just graph nodes (O(N²) Dijkstra runs over a built graph).

---

## 3. Parametric encoding — the compression secret [6]

### The problem

Raw impulse responses from wave simulation: N² pairs × (sampling rate × duration × 4 bytes) = tens of gigabytes. Not shippable.

### Solution: extract perceptual parameters per pair

The **Parametric Wave Field Coding** paper (Raghuvanshi & Snyder, TOG 2014) identifies a compact parameter set that captures what humans actually perceive:

| Parameter | Meaning |
|---|---|
| **Onset delay** | Time of first arrival (direct path time, possibly diffracted) |
| **Loudness / initial energy** | Peak amplitude of IR — captures occlusion/attenuation |
| **Arrival direction** | 3D vector of dominant initial wavefront direction (not LOS direction!) |
| **Reverb decay time (T60)** | Exponential decay constant fit to late IR |
| **Reverb direction loudness** | Directional distribution of reverb energy |
| **Direct-to-reverberant ratio** | Ratio captures room character |

The 2018 **Parametric Directional Coding** extension adds spherical-harmonic encoding of direction per energy-decay phase (early/late) — "9D" fields (3 for source pos, 3 for listener pos, plus SH order 3 for direction).

### The magic

Each parameter varies **smoothly** across source and listener positions (because diffraction is a smooth phenomenon at the perceptual parameter level, even when the raw IR is chaotic). This permits:

- **Image-style compression** of the parameter fields (they look like low-frequency images over the 6D space)
- Storage reduction to ~100 MB for a complete game level (cited by Microsoft Research)
- Runtime decode in ~100 μs per source via trilinear interpolation over the parameter grid

### What this captures that geometric+UTD misses

- **Room modes / resonance**: A source in a small room gets correct low-frequency boost at modal frequencies (not captured by ray/beam methods at all)
- **Smooth diffraction around non-edge features**: curved walls, cluttered furniture — wave sim sees all of it
- **Correct arrival direction through doors**: listener in corridor behind an open door correctly hears source through the door opening (arrival direction lights up the door direction, not the wall direction)
- **Correct late-reverb tail**: per-room decay time naturally emerges from wave sim
- **Portal scattering**: low-freq sound "bends" into shadow zones as wave theory predicts

---

## 4. Dynamic portal extension [3]

Raghuvanshi 2021 "Dynamic Portal Occlusion" adds **runtime-modulated closure** of doors/windows to precomputed fields:

- During offline bake, simulation assumes all portals **fully open**
- Portals are identified in pre-processing (location + aperture polygon) — but this **does** require some portal annotation (automated heuristically, not user-placed)
- At runtime, each portal has a `closure ∈ [0, 1]` state (0 = open, 1 = closed)
- The precomputed parameters (loudness, direction) are **modulated** based on which portals the diffracted shortest path crosses, and their closure states

### Portal-search algorithm

"Novel portal-search method that leverages **precomputed propagation delay and direction data** to find portals intervening the diffracted shortest path." Key idea: instead of searching geometric space for portals, **reconstruct which portals the sound traversed** by analyzing the precomputed arrival time and direction against candidate portal positions.

Scales **linearly with portal count** per frame — O(P) cost at runtime.

Integrated with UE4 and Wwise via drop-in.

### Inference

This paper acknowledges that even wave-based methods benefit from explicit portals **for dynamic state modulation**. Static wave bake + dynamic portal closure = better than either alone.

---

## 5. Trade-off matrix: Steam Audio vs Project Acoustics

| Aspect | Steam Audio | Project Acoustics |
|---|---|---|
| Physics model | Geometric + UTD surrogate | Full wave equation |
| Portal concept | None (implicit in probe graph) | Automatic detection for dynamic modulation |
| Probe placement | Uniform floor grid, artist OBB | Navigable-space-adaptive |
| Probe pair count | O(N²) Dijkstra | O(N²) wave simulations |
| Bake time per scene | seconds-minutes | hours-days (1 source at a time on cluster) |
| Output per pair | 4 int16 + 2 float | 5-10 float perceptual params |
| Output file size | ~5-20 MB for N~1000 | ~100 MB for full game level |
| Runtime cost | <100 μs per source | ~100 μs per source |
| Low-freq accuracy | Poor | Excellent |
| Room modes | Not captured | Captured |
| Dynamic geometry | Alt-path re-finding | Dynamic portal closure |
| Open source | Yes (Apache 2.0) | Yes (MIT, ProjectAcoustics GitHub) |
| Bake hardware | Any CPU | Azure cloud or local GPU cluster |
| Platform | cross-platform C API | Unity/Unreal plugins, Azure integration |

---

## 6. What you'd borrow into a voxel-based clone

**From Steam Audio**:
- Entire runtime architecture (probe graph, SH + EQ output, interpolation)
- UTD-surrogate deviation model (it's ~200 lines of C++)
- Visibility graph + Dijkstra bake
- DSP path effect application

**From Project Acoustics**:
- The *concept* of parametric encoding: if you can afford wave sim, reduce to ~6 floats per pair and interpolate
- Per-portal dynamic closure modulation (the 2021 paper)
- Navigable-region adaptive probe placement (better than uniform grid for non-rectangular spaces)

**NOT worth borrowing**:
- Full ARD implementation (years of work, only viable with cluster compute)
- Frequency-domain wave theory — for games, perceptual plausibility > physical accuracy

---

## 7. Why Project Acoustics is the "upper bound"

Any geometric method (Steam Audio, Tsingos beam tracing, etc.) fundamentally cannot reproduce:

- **Modal resonance** in small rooms (a guitar in a bathroom sounds different from a guitar in a hall — wave-based captures; geometric does not)
- **Low-frequency tunneling** (subwoofer energy penetrating a door gap physically follows wave equations not geometric paths)
- **Diffraction into shadow zones at low freq** (Keller UTD approximates but fails for λ >> aperture)

But wave sim's cost (cluster hours) and data pipeline (baking server, ACE file compression) make it impractical for small studios or procedurally-generated environments.

**For voxel-based procedural buildings (user's case), Steam-Audio-class geometric + UTD is the right answer.** Project Acoustics' architectural decisions are interesting as long-term aspirations, not as Day-1 implementations.
