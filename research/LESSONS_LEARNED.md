# Lessons learned & limitations — what the measurements taught

An honest, measurement-driven self-critique of the vectorial-hash scheme against
its own empirical results. Companion to [`BENCHMARKS.md`](BENCHMARKS.md) (culling),
[`UPDATE_STRATEGIES.md`](UPDATE_STRATEGIES.md) (relocation) and
[`STORAGE_AND_SCALE.md`](STORAGE_AND_SCALE.md) (scale/persistence): those report
*what won*; this reports *where the original thesis did not hold up*, so the
conclusions stay attributable to measurement rather than intuition. Every claim
is backed by a benchmark in the implementation repo (`vectorial-hash-kit`).

## 1. The adaptive binary split was rarely the top structure

The scheme is built on a binary-split tree (split the longest axis in two,
subdividing only where density demands). The 3D decision-map sweep did **not**
confirm it as the best structure in the general case:

- On **cull**, the 8-way **octree beats the binary tree by 14–19%** — fewer levels
  to descend (~5 vs ~15 pointer levels). *Descent-depth constant factors mattered
  more than the elegance of finer adaptivity.*
- On **uniform data**, the pointer-free **Morton (Z-order) grid beats both trees**
  (14.5× vs 11.4× binary vs 12.6× octree over brute force), with the cheapest build.
- On **maintenance** (per-frame relocation), the grid's flat re-bucket beat the
  trees in *every* config — until the fix in §2.

The binary tree keeps a genuine niche — **non-uniform / stacked** data (adaptivity
pays; it retakes the lead there) and it is the most **memory-lean** of the trees —
but it is a niche, not the default. **Takeaway:** present octree/Morton as the
first-line choices; frame the binary tree as the memory-frugal, non-uniform option.

## 2. The biggest speedup is a known technique; the contribution is the measurement

The largest maintenance win (**5–11×**) came from the arena layout plus an **O(1)
stable-handle relocation** (`ItemRef` / "keep-index": update items in place instead
of rebuilding). These are **established techniques, not novel ones** — arena /
index-based storage is idiomatic in systems code (slot maps, ECS storage, compiler
arenas), stable handle→slot tables are standard in engine resource systems, and
"update the broad-phase in place, don't rebuild" is textbook.

The defensible contribution is therefore **not the pattern** but the **finding**
that an `O(item_limit)` predicate scan inside `update` was the single dominant cost
sinking the trees against a flat grid, and that a stable handle removes it and
*flips the optimal structure* (15/16 configs). The novel elements of this work are
the **template-driven culling scheme** and the **measurement methodology** — not
the storage layout.

## 3. Templates and the 1×1×1 raster are conditional, not general

The precomputed-template culling and the 1×1×1 voxel raster were meant to make
classification cheap. Measured:

- The **raster loses to the analytic test** for simple shapes (a sphere): a memory
  lookup cannot beat one distance compare. It only wins when `contains_point` is
  **expensive** — the crossover is a convex polyhedron of ~**24–48 faces** — and a
  raster over a large bounding box is a memory bomb (a 4000³ raster is 64 GB).
- **Templates** are the same shape of result: they amortise an *expensive* per-node
  geometry test over many queries of one complex figure; for cheap analytic shapes,
  the direct classification is as fast without the precompute and selection overhead.

**Takeaway:** default to analytic classification; the template/raster machinery is
for **repeated queries of complex, non-analytic figures**, and can be gated on the
measured expensive-shape crossover rather than used unconditionally.

## 4. The hardware moved the goalposts (the recurring theme)

Several precomputation ideas measured *slower* than recomputing: the raster (§3), a
precomputed steering-force table (slower than the live maths), and offloading a
*moving* broad-phase to the GPU (the per-frame rebuild eats the kernel win). The
common cause is the **memory wall** — arithmetic became cheap and pipelined while a
cache-missing lookup did not. A scheme whose premise is "replace compute with a
table lookup" is working against the current hardware trend. **Takeaway:**
re-benchmark precompute-vs-recompute on current hardware before assuming the lookup
wins; report the negative results.

## 5. The GPU is not a free lunch for dynamic data

The GPU LBVH query kernel is ~100–400× the serial CPU cull, but that is the *kernel
only*. For moving data the hierarchy must be rebuilt every frame, and the honest
per-frame comparison hands the win back to a parallel CPU keep-index at 1 M
entities. **Takeaway:** offload to the GPU for **static / query-dominated /
rebuild-anyway** loads, or when the *entire* loop can be GPU-resident — not reflexively.

## 6. At scale, the interesting problems were operational, not algorithmic

Extending the scheme to large / persistent workloads, the algorithm carried over
unchanged (a sorted space-filling-curve key *is* the on-disk index). The real work
was **engineering**: an on-disk sorted-key store (B-tree vs LSM), the naïve-range
trap and its fixes (cell-probe / BIGMIN), key compression, and layered occupancy
tiers for sparse worlds — see [`STORAGE_AND_SCALE.md`](STORAGE_AND_SCALE.md).
**Takeaway:** the scaling gains were in persistence, layering and encoding, not in
a new partitioning algorithm.

## Threats to validity

- **Single machine, min-of-N.** Absolute numbers are one box; only the *shapes* of
  the curves and the crossover points transfer. No multi-platform sweep.
- **Some tests are "subset" checks** (e.g. a ray-cast validated as a subset of the
  exact capsule) which cannot catch silent *under*-collection — a conservative-but-
  wrong result could pass; those paths are flagged for verified, non-blind review.
- **The template fingerprint check is not fully cross-platform** (libm ULP noise);
  exact-byte equality binds only on a reference machine.

## What held up

- **Measure-everything discipline, negative results included** — the most
  transferable habit and the reason the conclusions are trustworthy.
- The **keep-index** as the right lever for moving-point workloads (prior art, but
  correctly applied and proven here).
- The **"two structures, two jobs"** split (an adaptive in-memory index + a
  sorted-SFC persistent store), validated end to end.
- **Exactness**: every operation gated against a brute-force oracle plus a
  randomised property/fuzz campaign — correctness never traded for speed.
