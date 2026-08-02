# Benchmarks: template-driven culling

Performance analysis of the vectorial-hash culling pipeline. All numbers are
reproducible with the commands below; rerun them after any change to the cull
path or the template bank, and refresh the tables.

## Sections at a glance

| # | What | Headline |
| --- | --- | --- |
| [1](#results-1--vh-bench-single-fixed-template-tree-vs-quadtree) | `vh bench`: single fixed template, binary-split tree vs quadtree | ~4× speedup from a single template; both trees within 10% on uniform data. |
| [2](#results-2--vh-bench-sizes-per-cell-size-selection-the-papers-scheme) | `vh bench-sizes`: per-cell-size selection (the paper's scheme) | 12–19× over no-template baseline; the precise method beats the old "snap" shortcut by ~5×; ~88% of index leaves share storage via content dedup. |
| [3](#results-3--vh-bench-walk-tree-descent-vs-neighbour-walk-flood-fill) | `vh bench-walk`: descent vs neighbour-walk flood fill | Hierarchical descent dominates; ropes is the best neighbour source but still 0.7× of descent and costs ~56% extra on inserts. |
| [4](#results-4--vh-bench-fallback-granularity-as-fallback-aggregation) | `vh bench-fallback`: granularity-as-fallback aggregation | The aggregated fallback is **exact**, costs 0.59 MB vs 1.70 MB of full precomputation, ~3× the no-template baseline. Memory/precompute knob. |
| [5](#results-5--vh-bench-scale-figureleftrightgrid-scale-equivalence) | `vh bench-scale`: figure↔grid scale equivalence | One canonical set serves many query scales: 25× less memory, 10× faster generation; cull cost equals direct at low factors, ~2.5× at factor 8. |
| [6](#results-6--headless-critters-a-full-dynamic-workload) | `critters_headless`: full dynamic workload (updates + culls + churn) | Quadtree ahead 10–35% even on dynamic ops (depth halves `locate`); hysteresis helps the binary tree; `item_limit` is the dominant knob; deterministic cross-structure runs with zero cull mismatches. |
| [7](#results-7--gpu-sort-the-on-gpu-lbvh-build-and-the-keeprebuild-crossover-2026-07-17) | GPU: radix sort · on-GPU LBVH build · keep↔rebuild crossover + adaptive · quantised nodes | GPU radix **8–23× the CPU at scale** (hierarchical scan + 8-bit/4-pass width; bitonic was ~2× slower); a whole LBVH builds **GPU-resident in ~4.4 ms/frame at 1 M** (verified by traversal-vs-brute); moving-data crossover at **f\* ≈ 30 % → 2.8 % moving** as N grows 262k→4M; adaptive-with-hysteresis beats both pure strategies; **quantised u16 BVH nodes are 1.6× smaller and EXACT** (footprint, not latency). |
| [8](#results-8--median-vs-midpoint-and-three-application-workloads-2026-07-27) | Median vs midpoint split (query + parallel build) · frustum index-vs-scan crossover · SPH neighbour search · static k-NN | Median split culls **2.2-2.5×** and k-NNs **1.67×** the midpoint tree on clustered data, and parallelises **3.4× vs 1.6-1.9×** (equal-count forks balance by construction); a frustum index only beats a linear scan **above ~1000 agents** (6.7× at 40k) while the exact LoS narrowphase dominates at scale; in SPH the **query IS the simulation cost**, and keep-index maintain is 3.5-3.9× cheaper than any rebuild but loses the frame on query. **All figures re-measured as medians of repeated idle-gated passes (§8.6) — three first-pass claims did not survive.** |

## Environment

| | |
| --- | --- |
| CPU | AMD Ryzen 7 7800X3D (8 cores / 16 threads) |
| OS | Windows 10 Pro (build 19045) |
| Toolchain | rustc 1.96.0, `--release` (opt-level 3, LTO) |
| Date of the numbers below | 2026-06-10 |

## Methodology

- Deterministic point cloud (xorshift64\*, fixed seed) — every config sees the
  same data; flags `--points`, `--culls`, `--item-limit`, `--seed` vary the
  scenario.
- **Correctness gate before timing**: every configuration must return exactly
  the same hit count, or the bench aborts. Speed without agreement is noise.
- Wall-clock `std::time::Instant` over N repeated culls, single-threaded
  queries, results consumed through `std::hint::black_box`. All structures
  answer the same contract (collect a `Vec` of item references), so no config
  gets a cheaper job.
- Template/bank generation happens before timing and is reported separately —
  it is the offline precomputation cost.

## How to reproduce

```bash
# 4-way: binary-split tree vs quadtree, single fixed template on/off
cargo run -p vectorial-hash-cli --release -- bench

# per-cell-size selection study (the paper's scheme), incl. the old
# snap-to-offset method and the industry uniform-grid baseline
cargo run -p vectorial-hash-cli --release -- bench-sizes

# both accept: --points N --culls N --item-limit N --seed N
```

## Results 1 — `vh bench`: single fixed template, tree vs quadtree

200k uniform points in a 4096² world, item_limit 16, query = drop polygon at
scale 1400 rotated 30°, one 64px-cell template classified per node via
`classify_region`. 50 culls/config, 5246 hits (all configs agree).

| config | avg/cull (ms) | speedup |
| --- | ---: | ---: |
| vectorial (no templates) | 2.275 | 1.0x |
| vectorial + template | 0.587 | 3.9x |
| quadtree (no templates) | 2.234 | 1.0x |
| quadtree + template | 0.533 | 4.2x |

Conclusions:

- A single precomputed template already cuts ~4x off either tree, mostly by
  proving subtrees fully-inside (taken wholesale) or fully-outside (skipped).
- With uniformly distributed points, the binary-split tree and the quadtree
  are equivalent (±10%). The binary tree's edge is structural (cheap
  incremental `remove`/`update` with the merge-up rule), not raw cull speed.
- **Clustered data doesn't change the picture** (`--clusters N` aims the
  query at the first cluster): with 12 clusters and ~27k hits, or 4 clusters
  and ~80k hits, both trees stay within ±1% of each other, with or without
  templates. Cull cost is dominated by classification work that is identical
  in both. Where they do differ: the quadtree *builds* ~1.8x faster (one
  4-way redistribution per level vs two 2-way ones) and only needs square
  template sets, while the binary split owes its place to the dynamic
  mechanics (merge-up with hysteresis, data-aware split axis, non-square
  worlds) rather than to query speed.

## Results 2 — `vh bench-sizes`: per-cell-size selection (the paper's scheme)

200k uniform points in a 4096² world, item_limit 16, query = drop polygon at
scale 350 rotated 30° applied at a **real integer origin — the figure is
never moved to fit any grid**. The bank resolves, per tree-cell size, the
template whose generation offset matches the origin's displacement within the
global virtual grid of that size (one resolution per size per cull, cached,
zero cell-data clones via `PlacedTemplate`). Leaf items use the 1×1 raster:
only boundary (`Maybe`) pixels run exact geometry. 50 culls/config, all
configs agree on the hit count.

Bank generation (offline, 16 threads): 1×1 raster 0.05s; sizes ≤16 in 0.19s
(577 combos → 410 unique); ≤32 in 0.19s (2,625 → 852); ≤64 in 0.14s
(10,817 → 1,081). Content dedup shares identical grids behind `Arc`s — at
≤64, **90% of index leaves point at a shared template**.

Memory (measured via `TemplateBank::memory_usage`, demo-sized bank — 2
figures, 24+1 angles, sizes 8–32 px + 1×1 rasters, 65,625 combos → 7,797
unique grids): **5.84 MB total**, of which the deduplicated template data is
only **1.06 MB** (~136 B per unique grid), the hierarchical lookup index is
**3.20 MB** (one ~41 B entry per key combination plus hash-map overhead —
the explicit price of O(1) lookup per key), and **1.58 MB** are flat cell
copies retained as dedup-map keys, a generation-time aid that could be
dropped after building.

| config | avg/cull (ms) | speedup |
| --- | ---: | ---: |
| no templates (bbox + exact geometry) | 0.138 | 1.0x |
| single 16px grid, `classify_region` (≈ old snap method) | 0.057 | 2.4x |
| bank ≤16 + raster | 0.014 | 9.8x |
| bank ≤32 + raster | 0.012 | 11.9x |
| bank ≤64 + raster | 0.011 | 12.4x |
| bank ≤64, **no** raster | 0.056 | 2.5x |
| quadtree, no templates | 0.145 | 1.0x |
| quadtree, bank ≤64 + raster | 0.008 | 17.1x |
| uniform grid 32px (industry baseline) | 0.136 | 1.0x |
| uniform grid 32px + raster | 0.007 | 19.2x |

### Conclusions

1. **The precise method beats the "easy" method it replaced.** Selecting the
   matching template (figure stays put) is ~4–5x faster than the old
   move-the-figure + single-grid `classify_region` approach — *and* it is
   exact. Per-node classification drops from a region scan to one array read.
2. **Gains saturate once template sizes cover the band where the tree
   actually lives** (leaf cells were 16–32px here). Each extra size family
   keeps helping slightly (≤64 > ≤32 > ≤16) now that resolution is
   zero-clone, but the increments shrink; cells much larger than the figure
   can never classify `In`, so their sets mostly buy corner `Out`s.
   The cost of over-generating is precompute time and RAM, not query time —
   the per-cull size cache caps lookups at one per distinct size.
3. **The 1×1 raster is half the win.** Without it the bank stalls at ~2.5x;
   with it, 12–19x. Replacing exact point-in-polygon (arcs!) with a raster
   read, reserving geometry for boundary pixels, dominates leaf cost.
4. **The technique composes with any spatial structure.** Quadtree + bank
   (17x) and even the flat uniform grid + raster (19x) beat the binary tree +
   bank (12x) *on static uniform data*, because their traversals are simpler
   and the raster equalizes the per-item cost everywhere. The trees' green
   short-circuit matters more as queries grow relative to cell sizes and as
   item density rises; the binary tree additionally keeps its dynamic
   `remove`/`update` merge-up behaviour, which none of the static baselines
   offer.
5. Caveats: single machine, uniform random points, one query shape per run,
   wall-clock timing. Scenarios still to measure: clustered/skewed point
   distributions, many simultaneous queries, larger worlds, mixed query
   sizes, and the planned "granularity as fallback" aggregation.

## Results 3 — `vh bench-walk`: tree descent vs neighbour-walk (flood fill)

Same world and bank setup as Results 2 (bank ≤64 + raster everywhere, so the
comparison isolates the **traversal strategy**). The walk starts at the leaf
containing a seed point inside the figure and expands through leaf
neighbours, stopping at `Out` leaves. Three neighbour sources: Samet-style
ascent/descent over the existing parent pointers (zero storage), point
probing + `locate` from the root (zero storage), and stored per-leaf
neighbour lists — *ropes* — behind the `neighbors` cargo feature (compiled
out entirely when off). All strategies pass the equality gate against
`Tree::cull`.

| config | scale 350 (ms/cull) | scale 1400 (ms/cull) |
| --- | ---: | ---: |
| descent (`Tree::cull`) | **0.012** | **0.076** |
| walk + Samet ascent | 0.024 (0.49x) | 0.213 (0.35x) |
| walk + locate probe | 0.026 (0.45x) | 0.318 (0.24x) |
| walk + ropes (stored) | 0.017 (0.70x) | 0.108 (0.70x) |

Rope maintenance cost: building the 200k-point tree takes 49.9 ms without
the feature vs 78.0 ms with it (~+56% on inserts; splits/merges rewire the
neighbour lists). Descent cull times are unaffected by the feature.

### Conclusions

1. **Descent wins, and the reason is structural**: with per-cell-size
   templates, an internal node classified green/white takes or skips its
   *entire subtree* without visiting leaves. The flood-fill walk has no
   subtree short-circuit — it must touch every leaf in the region, pay a
   visited-set membership check per leaf, and run 4 neighbour queries per
   leaf. The gap *widens* with query size (0.70x → 0.35x for the zero-storage
   variants) precisely because larger queries contain more wholesale-takeable
   subtrees.
2. **Among walk variants, ropes > Samet > probe**, as expected: O(1) stored
   lists beat O(1)-amortized ascent, which beats O(depth) root descents. But
   even ropes lose ~30% to descent while making every insert/split/merge
   ~56% more expensive — a bad trade for this workload.
3. Where a neighbour walk could still win: queries *without* useful templates
   (no green short-circuit to exploit), incremental queries that slide
   between frames (reuse the previous frontier), or region operations that
   are inherently neighbour-based (e.g. connected-component analysis,
   contour extraction). The mechanism stays available (`Tree::cull_walk`,
   `neighbors_samet`/`neighbors_probe`/`neighbors_ropes`) for those cases.
4. References for the strategies: H. Samet, *Neighbor Finding Techniques for
   Images Represented by Quadtrees* (1982); "ropes" as used in stackless
   spatial traversal (Popov et al., *Stackless KD-Tree Traversal for High
   Performance GPU Ray Tracing*, 2007); flood fill / region growing from
   image processing.

```bash
cargo run -p vectorial-hash-cli --release -- bench-walk            # ropes included
cargo run -p vectorial-hash-cli --release -- bench-walk --scale 1400
cargo run -p vectorial-hash-cli --release --no-default-features -- bench-walk  # no rope bookkeeping
```

## Results 4 — `vh bench-fallback`: granularity-as-fallback aggregation

Same scenario as Results 2. The .docx design notes record an exact property:
a template generated for a small cell size can stand in for *any* larger
cell size whose dimensions are an integer multiple, by aggregating blocks
with the rule `all-In → In, all-Out → Out, otherwise → Maybe`. This is
**not an approximation** — a cell is fully inside the figure iff every
sub-cell is, etc. So an aggregated template carries exactly the
classification a directly-generated one would have, only paid at query
time instead of at precomputation.

`PlacedTemplate::aggregated(fx, fy)` realizes the property (aligning the
output to the world grid of the new cell size), and
`TemplateBank::placed_for_or_aggregated(...)` automates the fallback: it
serves the directly-precomputed set when available, otherwise picks the
largest stored sub-size that divides the request and aggregates.

| config | bank size | gen time | avg/cull (ms) | speedup |
| --- | ---: | ---: | ---: | ---: |
| no templates | — | — | 0.155 | 1.0x |
| bank ≤16 + aggregated fallback | 0.59 MB | 0.20 s | 0.092 | 1.7x |
| bank full (every size precomputed) | 1.70 MB | 0.66 s | 0.011 | 14.2x |

Conclusions:

- **Correctness is preserved exactly.** The campaign (Results elsewhere)
  runs 2,000 random scenarios with the aggregating shape in the cull path
  and they all match brute force. The aggregated and the directly-generated
  templates return the same In/Out/Maybe classification for every aligned
  cell — verified by a unit test that picks the same shape and angle and
  asserts cell-by-cell equality.
- **The fallback is a memory/precompute knob, not a precision knob.** It
  costs ~3× the no-template baseline because aggregation builds a fresh
  template per cull (the per-execution `SizeCache` reuses it within a
  single cull, but not across culls); precomputing the full family pays
  itself back at runtime. Use the fallback when: (a) memory or generation
  time matters more than query speed, (b) only a few cell sizes carry the
  load (the fallback covers the rest), or (c) you are prototyping and
  haven't decided which sizes to precompute.

## Results 5 — `vh bench-scale`: figure↔grid scale equivalence

Per the .docx design notes: a template generated for figure F over cells C
is identical, cell by cell, to one for F·k over cells C·k (the
classification of a cell is invariant under uniform scaling). So one stored
set per shape's canonical size can serve any query scale by reading the
shared grid through a multiplier — no extra precomputation, no cell-data
clones (the grid stays behind an `Arc`).

`PlacedTemplate::with_scale` and `TemplateBank::placed_for_scaled` realize
the property at lookup time. Benchmark: a box-shaped query at four scales
× the same cull, with bank A (only the canonical set, served via
`placed_for_scaled`) vs bank B (one stored set per scale):

| query scale | scaled lookup (ms) | per-scale set (ms) | ratio |
| ---: | ---: | ---: | ---: |
| 1× | 0.002 | 0.002 | 1.0x |
| 2× | 0.003 | 0.004 | 1.3x **A** |
| 4× | 0.006 | 0.004 | 0.7x **B** |
| 8× | 0.014 | 0.006 | 0.4x **B** |

Memory: bank A is **25× smaller** (one set, 1 unique grid) and **10×
faster to generate**. The scaled lookup is competitive at low factors, but
cull time degrades at high ones because the canonical grid is small and
the cull walks more sub-cells per query node. The trade-off in words:

- Many query scales of the same shape, memory or precompute matters more
  than the last bit of cull speed → **bank A** wins overwhelmingly.
- A few well-known scales, cull speed dominates → **bank B**.
- The two compose with the granularity-as-fallback (Results 4): generate
  the canonical set + a few aggregating sizes, and the bank covers every
  scale and every cell size through a mix of direct hits, aggregation and
  scale equivalence.

## Results 6 — headless critters: a full dynamic workload

Everything above measures culls over a *static* tree. The critters
simulation exercises the whole dynamic contract at once: per-frame
`update` for every critter (with leaf relocation, splits and merges),
vision culls (one per hunter per frame), attack culls, kills (`remove`)
and respawns (`insert`) — all on one thread. The headless binary
(`critters_headless`) runs the exact simulation of the visual demo
without a window or vsync, deterministic per seed, so binary-tree and
quadtree numbers come from the *identical* event sequence (verified: a
`binary` run and the binary half of a `both` run produce the same kill
count and final tree shape, and the `both` mode's live agreement check
reports zero cull mismatches across entire runs).

```bash
cargo run -p vectorial-hash-demos --bin critters_headless --release -- \
    --mode binary|quad|both --frames 300 --drifters 1200 --hunters 1200 --pulsars 1200 \
    [--split N --merge N --dt S --seed N --fire X --respawn S --csv out.csv]
```

Scenario: targets 1200+1200+1200 (heavy combat keeps ~1.7k alive and a
large respawn queue), dt = 1/60 s, 120 warmup + 300 measured frames.
Mean per-frame timings in µs (move+update and insert+remove are totals;
attack/vision cull are per-cull averages):

| config | steps/s | move+update | attack cull | vision cull | insert+remove |
| --- | ---: | ---: | ---: | ---: | ---: |
| binary, split 3 / merge 3 | 62 | 186 | 6.2 | 21.3 | 5.3 |
| binary, split 3 / merge 1 (hysteresis) | 68 | 157 | 6.4 | 20.3 | 4.6 |
| binary, split 6 / merge 3 | 108 | 133 | 3.9 | 12.4 | 3.9 |
| quadtree, split 3 / merge 3 | **83** | 180 | 4.3 | 16.0 | 4.4 |
| quadtree, split 6 / merge 3 | **120** | 126 | 3.7 | 11.4 | 3.7 |

At a softer 400+400+400 the whole simulation costs ~1.8 ms/frame
(563 steps/s, binary) — the *visual* demo is vsync-bound, not sim-bound.

### Conclusions

1. **The quadtree comes out ahead in this dynamic workload too** — about
   10–35% across configurations, and not only on culls: `update`,
   `insert` and `remove` are also faster. The structural reason is depth:
   one 4-way level does the work of two binary levels, and `locate` (the
   first step of every dynamic operation) walks half the levels. This
   *revises* the earlier static-bench framing that "the binary split's
   edge is the dynamics": measured under real churn, fine-grained pair
   merging is *more* maintenance, not an advantage.
2. **Hysteresis helps the binary tree** (62 → 68 steps/s with merge 1
   vs merge 3): with `merge == split` the pair-granular merge rule
   genuinely flaps under churn. The quadtree's 4-way merge is naturally
   hysteretic (four leaves rarely sum under the threshold), which is
   part of its win at default settings.
3. **`item_limit` is the dominant knob for both**: split 6 / merge 3
   nearly doubles throughput over split 3 / merge 3 (binary 62 → 108,
   quad 83 → 120) — at ~1.7k items, limit 3 over-subdivides.
4. What remains genuinely in the binary split's favour: anisotropic
   distributions (data-aware split axis — not exercised by this roughly
   uniform scenario), rectangular / non-power-of-two worlds, smaller
   per-node footprint, and finer-grained memory growth. For uniform-ish
   dynamic worlds on square maps, the measured recommendation is the
   quadtree with a generous `item_limit`.
5. Vision culls dominate the budget (one per hunter per frame); the next
   scaling lever is re-evaluating prey every N frames rather than making
   individual culls faster.

## Industry context

What games/physics engines typically use for this class of query (and what we
benchmarked against):

- **Uniform grids / spatial hashing** — the classic broadphase; simple and
  extremely fast for roughly uniform object sizes. Covered in depth in
  Christer Ericson, *Real-Time Collision Detection*, ch. 7 "Spatial
  Partitioning" (cell-size tradeoffs, hashed storage)
  ([book](https://www.routledge.com/Real-Time-Collision-Detection/Ericson/p/book/9781558607323),
  [chapter contents](https://www.oreilly.com/library/view/real-time-collision-detection/9781558607323/xhtml/c07.xhtml)).
  Our `UniformGrid` baseline implements exactly this.
- **Quadtrees/octrees**, including Thatcher Ulrich's **loose octrees**
  (Game Programming Gems 1, 2000) which relax cell bounds to avoid small
  objects landing in huge nodes
  ([Ulrich's write-up](https://www.tulrich.com/geekstuff/partitioning.html)).
  Our reference quadtree is the strict variant; loose bounds matter for
  objects with extent, less for point items.
- **Dynamic AABB trees (BVHs)** — the broadphase in Box2D (`b2DynamicTree`,
  inspired by Bullet's `btDbvt`), Bullet and others; binary bounding-volume
  hierarchies rebalanced incrementally
  ([Box2D docs](https://box2d.org/documentation/group__tree.html)).
  For point items an AABB tree degenerates to roughly what our binary-split
  tree already is; the comparison would become meaningful with area items.
- General surveys of broadphase choices and their tradeoffs:
  [Build New Games — broad phase collision detection](http://buildnewgames.com/broad-phase-collision-detection/),
  [GameDev.net spatial partitioning discussion](https://www.gamedev.net/forums/topic/598183-spatial-partitioning/).

None of these precompute shape-vs-grid classification templates; they all run
exact (or conservative AABB) per-object tests after the broadphase. The
template bank + 1×1 raster is orthogonal to the choice of structure — as the
numbers show, it accelerates the industry baselines too.

## Results 7 — GPU: sort, the on-GPU LBVH build, and the keep↔rebuild crossover (2026-07-17)

Environment: NVIDIA RTX 4080 SUPER via `wgpu` compute (WGSL), Windows. All GPU
results are **verified against a CPU reference** (a build/sort is only reported
once its output matches brute force / a CPU sort exactly). Reproducible benches
live in the kit (`vectorial-hash-demos/examples/`): `gpu_spatial_bench`,
`gpu_sort_bench`, `gpu_radix_bench`, `gpu_lbvh_build_bench`; and the library
examples `compact_bench` and `compressed_bvh_bench`.

### 7.1 GPU sort of Morton codes — bitonic (negative) → radix (8–23× the CPU)

The on-GPU LBVH build needs the Morton-code **sort** on the GPU. Two attempts:

- **Bitonic** (`gpu_sort_bench`) — verified == a CPU sort, but **~2× *slower***
  than `sort_unstable` (log²(N) compare-exchange passes, one dispatch/submit each;
  the log² work factor + per-pass sync sink it). An honest negative — the right
  primitive is a radix sort.
- **Stable 4-bit LSD radix**, all-GPU (`gpu_radix_bench`) — 3 kernels/pass
  (per-tile histogram → exclusive scan → stable local-rank scatter), ping-pong,
  8 passes, no CPU in the loop. Verified == CPU at every size. **The scan was the
  whole story** — three iterations, each a lesson in where the real cost hid:
  1. a **single-workgroup serial** scan left it only *at parity* with the CPU;
  2. parallelising it **16-way** (one thread per digit) put it ~2× clear
     (262 k 1.09 ms/2.37× · 1 M 6.31 ms/1.77× · 4 M 24.51 ms/1.97×);
  3. a **hierarchical multi-workgroup** scan (reduce per block of tiles → scan the
     blocks → add back) removed the bottleneck that only bit at scale — the per-digit
     tile prefix is now parallel over blocks, not one workgroup walking every tile:

  | keys | GPU radix | CPU `sort_unstable` | speedup |
  | --- | --- | --- | --- |
  | 262 k | 1.68 ms | 2.53 ms | **1.51×** |
  | 1 M | 2.20 ms | 10.47 ms | **4.76×** |
  | 4 M | 4.32 ms | 45.77 ms | **10.59×** |

  So **1 M went 6.3 → 2.3 ms and 4 M 24.5 → 4.3 ms** (small N pays a little for the
  extra scan passes — 262 k 1.09 → 1.68 ms — but that is the fast regime anyway).
  Lesson (recurs throughout): the bottleneck was a **serial section**, not the
  algorithm — Amdahl in miniature.
- **Wider digit → fewer global passes** (the Onesweep direction). Sorting 32-bit
  keys **8 bits at a time = 4 passes** instead of **4 bits = 8 passes** halves the
  global histogram+scatter round-trips, at the cost of a 256-bucket (vs 16) histogram
  and scan per pass. Measured — it wins across the board:

  | keys | 4-bit ×8 | 8-bit ×4 | 8-bit vs 4-bit | 8-bit vs CPU |
  | --- | --- | --- | --- | --- |
  | 262 k | 1.70 ms | **0.93 ms** | 1.83× | 2.75× |
  | 1 M | 2.20 ms | **1.30 ms** | 1.70× | **8.11×** |
  | 4 M | 4.35 ms | **2.79 ms** | 1.56× | **16.78×** |
  | 8 M | 8.13 ms | **5.42 ms** | 1.50× | **22.57×** |
  | 16 M | 16.28 ms | **12.15 ms** | 1.34× | 17.23× |

  So the GPU radix is **8–23× the CPU** at scale, peaking ~**22.6× at 8 M** — the
  8-bit width holds its ~1.3–1.5× edge over 4-bit throughout (the 256-bucket scan
  overhead only slowly erodes it), and the GPU-vs-CPU ratio softens past 8 M as the
  sort turns bandwidth-bound while `sort_unstable` scales sub-linearly. A *true*
  single-pass **Onesweep**
  (one decoupled-lookback scan) would cut it further — **and it was attempted, and
  measured un-implementable in portable WGSL** (`gpu_onesweep_scan_bench`). The
  look-back publishes a per-tile aggregate then a full prefix and spins on
  predecessors' flags; it needs a **device-scope** acquire/release between one spinning
  thread and other workgroups (CUDA's `__threadfence()` + `volatile`). WGSL has **no
  device fence** — its only ordering primitive `storageBarrier()` is **workgroup-scope
  and must be called uniformly**, so it cannot order a single look-back thread against
  other workgroups. Measured on native NVIDIA (RTX 4080 SUPER): the scan returns a
  **wrong** prefix — a tile reads a predecessor's flag *before* that predecessor's
  aggregate is visible — with **no** spin-timeout, i.e. it is a **memory-ordering**
  wall, not a forward-progress one (refining the earlier wording). So **8-bit/4-pass is
  the honest portable ceiling** for a spec-clean WebGPU radix. (Follow-on done: the
  8-bit width is folded into the build's key-value radix, §7.2.)

### 7.2 A whole LBVH built on the GPU — Morton → radix → Karras → refit

`gpu_lbvh_build_bench` builds an LBVH **entirely GPU-resident**, no round-trip:
Morton codes (30-bit, GPU) → the key-value radix above (codes + primitive index)
→ **Karras** hierarchy (common-prefix ranges + split, with the point index as a
tiebreaker so equal codes stay well-founded) → **AABB refit** (a leaf-init pass,
then a bottom-up climb gated by an atomic per node — only the *second* child to
arrive unions the two boxes). Verified end-to-end by **traversing the GPU-built
BVH on the CPU and comparing to brute force** over random spheres (a pass ⇒ the
hierarchy *and* the refit AABBs are correct), first try, at every size:

| points | build/frame (min of 7), 4-bit → 8-bit sort | throughput |
| --- | --- | --- |
| 262 k | 2.28 → **1.58 ms** | 166 Mpts/s |
| 1 M | 4.40 → **3.7 ms** | 273 Mpts/s |
| 4 M | 12.98 → ~14 ms (flat) | 283 Mpts/s |

So a **1 M-point BVH rebuilds on the GPU in ~3.7 ms/frame** — 8.4 ms → 4.4 ms once
the radix scan went hierarchical (§7.1), then the build's own key-value radix took
the **8-bit / 4-pass** width (§7.1). The width helps where the *sort* is a big slice
of the build (262 k **1.4×**, 1 M **1.2×**); at 4 M the sort is a smaller fraction
(Karras + refit dominate) and the 256-bucket scan overhead makes it a **wash** — an
honest, size-dependent win, verified == brute at every size. (The atomic refit
relies on the platform's atomic ordering for cross-workgroup visibility — correct
on this NVIDIA hardware per the verify; a level-by-level refit would be
spec-portable.)

### 7.3 The moving-data crossover — GPU rebuild vs CPU keep-index

The point of the on-GPU build: for **moving** data, a per-frame rebuild competes
with the CPU keep-index (`update_ref` in place). The keep-index's real edge is
that it **skips items that didn't move**, so its cost is ~linear in the *moving
fraction* while the GPU rebuild is flat. Measured on the same points (keep at its
best case — tiny jitter, ~no relocations):

| N | GPU rebuild | CPU keep (all, best) | CPU rebuild (`bulk_load_par`) | GPU beats keep | f\* moving |
| --- | --- | --- | --- | --- | --- |
| 262 k | 2.28 ms | 5.69 ms | 30.71 ms | 2.5× | **29.9 %** |
| 1 M | **4.40 ms** | 48.55 ms | 132.81 ms | 11.0× | **7.6 %** |
| 4 M | 12.98 ms | 451.52 ms | 681.95 ms | 34.8× | **2.8 %** |

(These are the post-hierarchical-scan build numbers; the GPU column is §7.2. "Keep"
here moves *all* N with tiny jitter — its **best** case; real motion with
relocations only grows it.) The rule is by **moving fraction**, not just N: with
*most* of the cloud moving the parallel GPU rebuild wins (a serial maintain-*all*
pass loses to a parallel from-scratch build); with a *small* fraction moving the CPU
keep-index wins by skipping the rest (the demos' regime — a mostly-dormant
population keeps for ~nothing). They cross at **f\* ≈ 30 % (262k) → 7.6 % (1 M) →
2.8 % (4 M)** moving — and **f\* drops sharply with N**: the serial keep pass scales
~linearly while the parallel GPU build stays near-flat, so the bigger the world the
less motion it takes to justify rebuilding on the GPU.

### 7.4 Adaptive keep↔GPU with hysteresis

Since keep-cost is ~linear in the moving fraction and the GPU rebuild is flat, an
**adaptive** policy — keep below f\*, GPU rebuild above, with a **hysteresis**
dead-band to avoid thrashing at the boundary — beats *both* pure strategies. On a
120-frame wave whose moving fraction ramps 0→1→0 (1 M points): adaptive **504 ms**
vs pure-GPU 527 vs pure-keep 4163. (The margin over pure-GPU is small here because
a 0→1→0 ramp spends most frames *above* f\*=7.6 %, where GPU already wins; adaptive's
edge is the low-fraction frames it hands back to keep.) Near f\* the two costs are
~equal, so the damage of a needless flip is the **switch itself** (rebuild warm-up,
frame-time variance) — a ±25 %-of-f\* dead-band roughly **halves the switch count**
when the load hovers at f\* (**110 → 64** over 200 noisy frames) at ~equal raw cost.

### 7.5 Cache-locality: `Tree3::compact()`

A one-pass DFS pre-order reorder of the node arena (a node lands adjacent to its
first child; freed slots reclaimed) — a pure layout change (identical
cull/knn/handle results, brute-force-gated). On a churned keep-index tree it buys
a steady **~1.16× cull** (100k: 3.66→3.13 µs/query; 200k: 5.50→4.74), not the
literature's 26–300 % (that was pathological / other structures). One pass,
amortises over many frames; best before a query-heavy phase after churn.

### 7.6 Compressed (quantised) BVH nodes — exact, and a footprint (not latency) win

A BVH node stores an AABB (2 corners × 3 axes = 6 numbers) plus 2 child indices.
The memory-wall lever the literature names is **quantising the box**: instead of a
32-bit float per coordinate (absolute world position), store each coordinate as a
**u16 index into a 65 536-step grid spanning the root box** —
`q = round((coord − root_min) / root_span × 65535)`. Node size drops from **32 B**
(6×f32 + 2×u32) to **20 B** (6×u16 + 2×u32) = **1.6× smaller**.

**The exactness argument (the key point).** A u16 grid is coarse (step =
root_span/65535 ≈ 0.016 units for a 1024-wide scene), so a quantised box does not
match the true box. Used naïvely it could *shrink* a box and **miss** a point — a
wrong answer. Two moves make it **provably exact**:
1. **Round outward** — min to the grid `floor`, max to the grid `ceil` — so the
   dequantised box is always a **superset** of the true box. It never shrinks ⇒
   never misses; at worst it is slightly too big ⇒ a few extra (empty) node visits.
2. **Exact leaf test** — the quantised boxes decide only *which subtrees to
   descend*; the actual points live in a separate f32 array, and the leaf tests the
   query against the **exact point**. So quantisation changes only *how many nodes
   you visit*, never *what you return*.

⇒ The quantised cull is **bit-for-bit identical to brute force** (verified over
random spheres at 200k / 1 M / 4 M). *This corrects an earlier hasty claim that
quantised AABBs "break the exact-query contract" — they do not, given outward
rounding + exact leaf tests. The conservative boxes cost traversal, never accuracy.*

Measured (RTX 4080 SUPER, min-of-8, a clumpy 3D cloud so the tree is non-trivial;
`vectorial-hash/examples/compressed_bvh_bench.rs`):

| N | node (full → quant) | arena (full → quant) | over-visit | cull full → quant |
| --- | --- | --- | --- | --- |
| 200 k | 32 → 20 B (1.6×) | 12.8 → 8.0 MB | **0.0 %** | 18.4 → 18.6 µs (1.01× slower) |
| 1 M | 32 → 20 B (1.6×) | 64 → 40 MB | **0.0 %** | 206.6 → 204.4 µs (1.01× faster) |
| 4 M | 32 → 20 B (1.6×) | 256 → 160 MB | **0.0 %** | 778 → 847 µs (1.09× slower) |

Two readings. **(a)** Over-visit is **0 %**: at u16 the ~0.016-unit outward rounding
is far below the clump scale, so the conservative boxes are traversal-identical to
the exact ones — the footprint drop is genuinely free of traversal cost. **(b)**
Cull latency is a **wash** (±≤10 %, noise-dominated): the per-node dequantise (a
float mul-add × 3 axes) offsets the smaller-node cache win, and a *binary* BVH
traversal is **pointer-/branch-bound, not bandwidth-bound**, so shrinking the node
does not move the critical path.

**Verdict.** Quantisation is a **footprint** lever — fit ~1.6× more BVH in
cache/VRAM (it matters for the GPU build against WebGPU's
`maxStorageBufferBindingSize`, and for huge static worlds) — **not** a cull-speed
lever for the binary layout, and it is **exact**. The literature's *latency* win
comes from **wide (8-ary) compressed nodes**: one node holds 8 children, all 8
child boxes tested with SIMD, which amortises the pointer-chase and vectorises the
box tests. That wide-node layout — not quantisation per se — is where the latency
win lives.

**And that wide node, measured (`wide_bvh_bench`).** An 8-ary BVH with each node's
8 child boxes stored **SoA** (`lo[axis][8]`, `hi[axis][8]`) so the sphere-vs-8-boxes
test is a fixed 8-wide loop LLVM **auto-vectorises to AVX** (`-C target-cpu=native`);
leaves hold ≤8 points, tested exactly ⇒ verified **== brute force**. Three BVHs over
the same clumpy cloud (RTX 4080 SUPER box, min-of-8):

| N | bin-f32 | wide8-f32 | wide8-u16 | nodes/query (bin→wide) | arena (bin→wide-u16) |
| --- | --- | --- | --- | --- | --- |
| 200 k | 15.1 µs | **8.8 µs (1.71×)** | 9.8 µs (1.53×) | 1460 → 42 (35×↓) | 12.8 → 1.5 MB |
| 1 M | 88.9 µs | **41.2 µs (2.16×)** | 45.1 µs (1.97×) | 6566 → 196 (34×↓) | 64 → 9.4 MB |
| 4 M | 626 µs | **283 µs (2.21×)** | 298 µs (2.10×) | 24427 → 1117 (22×↓) | 256 → 59 MB |

So the wide node is a **real ~2× latency win** (growing with N) where the *binary*
u16 node was only a wash — the difference is going **wide**, not the quantisation:
the 8:1 fan-out visits **~30× fewer nodes** (shallow tree, far fewer pointer-chases)
and the 8-box test vectorises. The arena also shrinks sharply (points batch into
≤8-point leaves ⇒ ~64× fewer internal nodes): 1 M drops **64 → 13 MB** (f32) / **9.4
MB** (u16). Quantising the *wide* node to u16 costs the same small dequantise offset
as the binary case, yet stays ~2× over binary **and** ~1.4× smaller than wide-f32 —
so **wide8-u16 is the best footprint-and-speed point**. **But the 2× is over a
*strawman*** — reference-checked (2026-07-19) the bench also culls the same cloud with
the kit's **shipping `Tree3` / `Octree3`**: wide8-u16 is only **~1.0× vs Tree3** (200k
1.03× · 1 M **0.99×**, Tree3 *faster*) and **~1.05× vs Octree3**. The kit's tuned arena
descent already sits at the wide node; the 2× was purely over a naive pointer-BVH. So
the honest verdict is *not* "graduate a wide BVH into the kit" — the win evaporates
against the real cull; the 8-ary SoA layout's home stays the **GPU LBVH** (§7.2), where
the alternative is a naive kernel. The compressed-node thread's real lesson: *the
literature's latency win is real over a naive BVH, but the kit's cull already captures
it — measured, not assumed, on both sides.*

### 7.7 The honest negatives (what we did NOT do, and why)

- **Uniform-density collision wants a grid, not a BVH.** A hash grid is
  O(1)-per-cell and already the right broad-phase for a uniform storm; a BVH's
  edge is *non-uniform* density or *culling by shape* (ray / frustum / sphere).
  So the GPU LBVH build was **not** retrofitted into the uniform-collision demo.
- **`compact()` was not wired into the demos** — their per-frame cull is a small
  slice, so ~1.16× of it isn't worth the periodic compaction hitch.
- **The GPU wins the query, but not always the frame.** A GPU broad-phase *kernel*
  is ~100–400× the serial CPU cull, but a *moving* cloud must rebuild the index —
  which is where §7.3's fraction-dependent crossover, not the kernel speed,
  decides CPU-keep vs GPU-rebuild.


## Results 8 — median vs midpoint, and three application workloads (2026-07-27)

Four measurements taken while building three application demos in the kit
(`fluid_wgpu`, `pointcloud_wgpu`, `stealth_wgpu`). They are here rather than in the kit
because each is a *methodology* result — the kit keeps only the conclusion.

Environment as §7 (Windows 10, RTX 4080 SUPER box, 16 threads, `--release`).

**Every figure below is the median of repeated passes** taken by the kit's `bench-runner`
(`cargo run -p bench-runner --release -- --group <g> --repeat N`), which waits for the
machine to be idle before each pass and reports the peak-to-peak spread. This section was
**re-measured that way on 2026-07-27**, and §8.6 records the three figures from the first,
hand-taken pass that did not survive.

### 8.1 The median split: worse build, better query — and the better parallel build

`KdTree3` splits at the point-count **median**; `Tree3`/`Octree3` split at the spatial
**midpoint**. 200 000 points, min-of-5, `examples/kdtree3_bench`:

| | build serial | build par (16 thr) | speed-up | cull ×64 | knn k=16 |
| --- | ---: | ---: | ---: | ---: | ---: |
| `KdTree3` uniform | 20.13 ms | **6.03 ms** | **3.35×** | 4.87 ms | 2.34 ms |
| `KdTree3` clustered | 20.57 ms | **6.02 ms** | **3.38×** | **0.39 ms** | 5.45 ms |
| `Tree3` uniform | 40.49 ms | 25.77 ms | 1.56× | 6.07 ms | 2.74 ms |
| `Tree3` clustered | 55.86 ms | 29.98 ms | 1.86× | 0.87 ms | 9.12 ms |
| `KdTree2` (2D, clustered) | 7.96 ms | **2.85 ms** | **2.79×** | 7.42 ms | 1.47 ms |

Two findings:

1. **On clustered data the median split culls 2.2–2.5× and answers k-NN 1.67× faster**
   than the midpoint binary tree (and beats `Octree3`/`LinearOctree3` by more). On
   *uniform* data the gap narrows to 1.24×/1.17× — there is nothing to balance, which is
   the control that makes the clustered number meaningful. The cull ratio is quoted as a
   range on purpose: the k-d cull is ~0.39 ms, the smallest quantity in the table, and it
   moves ±7% between passes.
2. **The median split also parallelises about twice as well** (3.4× vs 1.6–1.9× on 16
   threads; the 2D twin `KdTree2` repeats it at 2.8×). This is structural, not incidental: `rayon::join` is worth what its *slower*
   half is worth, a median split hands each fork exactly `n/2` points by construction, and
   a midpoint split halves the *box* — on clustered data one side can receive almost
   everything and becomes a serial tail with idle threads beside it. The literature
   presents the median split's O(n) selection as its cost; **that cost is the part that
   recovers best from cores**, and threaded, the k-d build (6.0 ms) is faster than every
   serial build measured, including the linear octree's 13.6 ms.

The parallel build is **node-for-node identical** to the serial one (asserted in the kit's
tests: node count, every leaf box and range, depth, and all query answers), because the
serial build already emits parent-then-left-then-right, so a subtree can be built into its
own node vector and spliced in with an id shift.

### 8.2 When an index beats a linear scan: the frustum-cull crossover

A guard's view cone is a 6-plane `Polyhedron3`; "who is in my cone" can be an indexed cull
or a linear scan of every agent against the same six half-spaces. Both were run **every
frame, on the same data, with the answers compared** (9 cones, 90 occluders):

Means over 600 stepped frames, median of 3 passes:

| crowd | indexed cull | linear scan | winner |
| ---: | ---: | ---: | :--- |
| 40 | 3.1 µs | **1.1 µs** | scan, 2.7× |
| 160 | 7.6 µs | **3.7 µs** | scan, 2.1× |
| 640 | 22.8 µs | **16.8 µs** | scan, 1.4× |
| 2 560 | **92.0 µs** | 224.4 µs | index, 2.4× |
| 10 240 | **226.8 µs** | 934.8 µs | index, 4.1× |
| 40 000 | **514.4 µs** | 3 431.7 µs | index, 6.7× |

**The crossover is ~1000 agents** — the same order as §6's `BRUTE_FORCE_MAX` for sphere
culls, reached from a completely different query verb. Below it the index is honestly
slower: a `contains_point` against six planes is a handful of FMAs, and the traversal
costs more than looking at everything.

Also worth recording: at 40 000 agents the **exact line-of-sight** stage (capsule
broadphase → `Polyhedron3::segment_hit` per candidate) costs 7 600 µs — an order of
magnitude more than either broadphase. Where a narrowphase runs per *candidate*, tightening
the query volume beats optimising the broadphase.

### 8.3 SPH: the neighbour search *is* the simulation

Position-based fluid, 2 200 particles, 3 constraint iterations/step, per frame:

| neighbour index | maintain | query | physics | fps |
| --- | ---: | ---: | ---: | ---: |
| `MortonGrid` (rebuilt each step) | 0.107 ms | **1.541 ms** | 1.082 ms | **337** |
| `Tree` + `ItemRef` (kept, relocated) | **0.031 ms** | 1.884 ms | 1.095 ms | 306 |
| `LinearQuadTree` (rebuilt each step) | 0.121 ms | 1.620 ms | 1.077 ms | 327 |

The keep-index tree's maintain is **3.5–3.9× cheaper than either rebuild** on a workload
where *every* item moves every step — the strongest case yet for the `ItemRef` thesis —
but it gives more than that back in query (+22%), because a kept tree drifts from the
ideal partition while a rebuild is always perfectly fitted; on this workload keeping the
index is a net loss. The two rebuilt structures are within 5% of each other, the flat grid
marginally ahead. **Query dominates every mode**, which is the headline: in SPH the
neighbour search is the simulation cost and the physics is cheap arithmetic.

### 8.4 k-NN per point on a static skewed cloud

150 000 points on surfaces (ground sheet, building shells, canopies — dense in thin sheets,
empty between), colouring each point by its local density, i.e. one k-NN query per point:

Median of 7 passes:

| structure | build | k-NN over all points | per query | spread |
| --- | ---: | ---: | ---: | ---: |
| `KdTree3` | 11.0 ms | 214.0 ms | **1.78 µs** | 16% |
| `Octree3` | 18.5 ms | 215.8 ms | 1.80 µs | 3% |
| `MortonGrid3` | **6.3 ms** | 322.3 ms | 2.69 µs | 2% |

Both trees are **~1.5× the flat grid** on k-NN, and are **tied with each other** — 214.0 vs
215.8 ms sits well inside the k-d tree's own 16% run-to-run spread. So on real (not
synthetic) skew the median split's advantage over the midpoint octree shows up in the
**build** (1.7×), not the query; against the *grid*, adaptivity wins either way. The flat
grid still wins the build outright. Build-once-query-many favours a tree;
rebuild-often-query-rarely favours the grid.

### 8.5 Methodology note: an index only knows what it holds

The 8.2 comparison disagreed on ~77% of frames before it was trusted. The library was
innocent — a standalone check (400 random frustums × 4 000 points) found **0**
disagreements between `Tree3::cull` and per-point `contains_point`. The cause was that some
agents had drifted **outside the index's world box**, where `bulk_load` correctly drops
them while a linear scan still counts them. Neither side was wrong; they were answering
questions about different sets.

The general point for any index-vs-brute-force comparison, this thread's included: verify
that both sides see the same population *before* attributing a difference to the algorithm.
It is also why an index-vs-scan comparison is worth running continuously rather than once —
it is a live invariant, not a benchmark.

### 8.6 What re-measuring changed (and the tool that did it)

§8 was first written from single, hand-taken readings — each bench run once, whenever the
work happened to finish, on a machine that was doing other things. Re-running the whole
set with the kit's `bench-runner` (idle-gated, repeated, spread-reporting) contradicted
three of the published figures:

| claim | hand-taken | median of repeated passes |
| --- | ---: | ---: |
| `KdTree3` cull vs `Tree3`, clustered | 3.06× | **2.2–2.5×** |
| fluid: which index wins the neighbour query | `LinearQuadTree` | **`MortonGrid`, by 5%** |
| point cloud: `KdTree3` k-NN vs `Octree3` | 1.12× | **tied** (inside the k-d spread) |

Two of the three were *qualitative* claims — "the adaptive structure wins the query", "the
median split answers k-NN faster than the octree" — reversed by noise of a few percent.
The absolute times moved by 25–30% between sessions on the same machine and binary, while
*ratios measured within a single pass* stayed stable to ~2%. Two rules follow, and they
cost nothing:

1. **Quote ratios, not absolutes**, unless the environment is pinned in the same table.
2. **A number seen once is not a measurement.** Report the spread; if it is above ~10%,
   quote a range and say which quantity is the small one.

A third rule came from the stealth demo rather than the tool: it reported *one frame's*
values, and across three passes one landed on a frame that had not stepped and printed a
plausible, clean **zero**. Summaries must aggregate over the run, not sample it.


---

## Results 9 — counting instead of timing, and the defect it found (2026-07-29)

Section 8 ended on rules about how to *time* things. This section is about not timing them.

What differs between a binary tree, an octree, a k-d tree and a uniform grid answering the
same query is **how much work each does**: node boxes classified, points tested. Those are
integers. They are identical on every run, on every machine, at every optimisation level, and
under any background load — verified byte-for-byte between an idle run and one taken with 32
processes burning CPU, and verified again by a CI job on Linux against numbers blessed on
Windows. No clock is involved, so none of section 8's problems exist.

The instrumentation needs no library change. `cull` accepts any shape, so the query is wrapped
in a counter that tallies each `classify_aabb`/`contains_point` and forwards. `knn` and
`raycast` take a point and a ray, so for those the counter goes in the **item** instead: a
traversal must ask an item where it is before testing it, so counting `position()` counts leaf
work for every verb and every structure.

### What the counts say

**The median split does not beat the binary tree; it beats skew.** k-NN, k=8, 200k points:

| distribution | `KdTree3` points tested/query | `Tree3` | ratio |
| --- | ---: | ---: | ---: |
| clustered (6 tight blobs) | 219 | 404 | **1.8×** |
| uniform | 86.6 | 92.1 | **1.06×** |

The advantage is not a property of the structure. It is the structure meeting non-uniform data,
and on uniform data the cheaper build wins and the k-d tree has nothing to sell. A timing
experiment can show the 1.8×; it cannot show *why*, because a duration has no units of cause.

**A uniform grid's k-NN collapses under clustering** — 596 points tested per query on uniform
data, **166 640** on clustered, out of 200 000. The shell expansion crosses empty cells until
it reaches a blob, then has to scan the blob whole.

**Approximate queries should be quoted with recall, not just speed.** The DDA ray walks visit
only the leaves the centre ray crosses, so they return a strict subset of the exact capsule
answer — documented, but never checked. Counted: zero invented hits across all three walks,
and the trade is 75% of the hits for 23% of the point tests (binary tree), 50% for 11%
(octree). "Faster" alone would have been a misleading way to describe either.

### The defect the counting found

The 166 640 above is not only clustering. `MortonGrid3` derives its cell size from a single
`levels` value for all three axes, so a world that is not a cube does not get cubic cells: at
levels 5 a 1000×300×1000 world has cells of 31.25 × 9.375 × 31.25. An expansion that grows one
Chebyshev radius for all three axes is then **isotropic in cell space and anisotropic in world
space** — to reach 30 units along the short axis it drags the wide axes out to ±125 and scans
everything between. Growing each axis separately, always the one currently narrowest in world
units, keeps the scanned region near-cubic. The stopping rule is unchanged and still exact:
the region is still a box, so the nearest unscanned point is still at least `safe` away.

| world aspect (h/w) | points tested/query | ms/query | speed-up |
| ---: | ---: | ---: | ---: |
| 1.00 (cubic cells) | 14 946 → 14 823 | 0.2012 → 0.0883 | **2.3×** |
| 0.50 | 33 846 → 14 595 | 0.5713 → 0.0889 | **6.4×** |
| 0.30 | 41 529 → 12 933 | 1.0779 → 0.0867 | **12.4×** |
| 0.15 | 45 510 → 11 452 | 1.3533 → 0.0784 | **17.3×** |
| 0.05 | 48 219 → 10 591 | 1.6112 → 0.0761 | **21.2×** |

(50k clustered points, k=8, levels 5. The 2D grid had the identical defect on a non-square
rectangle and the identical fix, 1.9× to 8.6× — one fewer axis to over-scan.)

### Two rules, both learned the hard way here

**A count only counts what you counted.** Look at the cubic row: the point count barely moves
and the time still falls 2.3×. Every one of those saved cycles went on **cells never visited** —
the old enumeration walked the whole Chebyshev shell and rejected out-of-grid cells one at a
time, the new one clamps its loops to the grid. Visiting an empty cell costs real time and
calls nobody's `position()`. Counts *prove* an algorithmic change; they do not *bound* one. When
a count says nothing changed and a clock disagrees, the clock may be measuring something nobody
thought to count.

**A ratio is only true while both of its terms stand still.** The kit's documentation said the
adaptive linear octree answered a clustered k-NN ~5× faster than the uniform grid. It did, when
it was written. After the per-axis fix the grid got ~3.6× faster on that workload and the gap
collapsed to **1.4–1.7×** (three paired runs; quoted as a range because the third read 1.74
where the first two read 1.41, spread 30%). The linear octree did not get worse — its rival got
better, and nothing in the repository would have noticed if the benchmark that produced the
figure had not still been runnable. Publish the *measurement*, not only the number.

### And the gate that follows from all of this

Because the counts have no variance, they are the only quantity here that can be gated with
`==` rather than a tolerance. Twenty traversal counts across ten structures and three verbs are
now checked exactly, in CI, on every push. A timing gate has to pass anything within ~25%,
which means a traversal change costing 15% more work reads as noise; this one cannot miss it.

Two cautions, both from building it. The first version of the gate blessed `tested = 0` for
four of five 3D structures — uniform query points in a world holding six tight clusters mostly
land in empty space, so it was a ratchet holding nothing. And it was only trusted after being
checked against a deliberate perturbation: changing a leaf capacity from 16 to 15 must fail it,
and does. **A test that has never been watched fail is a comment.**

## Results 10 — when a measurement is not measuring what its name says (2026-08-02/03)

Two nights whose findings are almost all *methodological*, and whose most useful output is a
list of things that were believed on insufficient evidence — several of them written down here
by us. The measurements live in `vectorial-hash-kit`; the reasoning belongs here.

### 10.1 A comparison can be rigged by omission, not by intent

The kit's stealth demo races an index against a linear scan and reports which wins. It also
rebuilt that index from scratch every frame **outside every timer**. So the index's cost was its
queries, the scan's cost was its queries, and the index's maintenance was free because nobody
had written a clock around it.

Charging it changes the verdict completely (600 stepped frames, means):

| crowd | kept: maintain + cull | rebuilt: maintain + cull | scan | verdict, kept |
| ---: | ---: | ---: | ---: | ---: |
| 200 | 1.4 + 9.8 = 11.2 µs | 25.5 + 9.1 = 34.6 µs | 4.5 µs | scan wins, 0.40× |
| 2 000 | 15.2 + 78.9 = 94.1 µs | 339.7 + 80.0 = 419.7 µs | 151.4 µs | index, 1.61× |
| 20 000 | 296.8 + 351.0 = 647.8 µs | 4 903.0 + 354.0 = 5 257.0 µs | 1 760.2 µs | index, 2.72× |

Charged for the rebuild, **the index never wins at any size measured**. The demo would have been
answering "the index does not pay here" — correctly, for a workload nobody would write.

The general form is worth stating because it is easy to reach with no bad intent: *if one side of
a comparison has a cost the other does not, and that cost sits outside the timed region, the
comparison measures the omission.*

### 10.2 The threshold that is not a number

"Below how many items does a linear scan beat an index?" had two competing answers in one
codebase — 512 shipped, 182 measured — and the disagreement stood for months. It was
unresolvable as posed. A scan costs per **query**; an index costs per **move**. Sweeping both
axes (500³ world, 300 frames, a quarter of the population moving each frame, every arm run
through the same index with its backend pinned so only the backend differs):

| population | 1 cull/frame | n/16 | n/4 | n culls/frame |
| ---: | --- | --- | --- | --- |
| 64 | scan 1.96× | scan 1.41× | scan 1.10× | scan 1.06× |
| 128 | scan 2.07× | scan 1.12× | **keep 1.10×** | **grid 1.28×** |
| 512 | scan 3.80× | keep 1.45× | keep 1.80× | grid 1.97× |
| 2 048 | **scan 7.00×** | keep 2.82× | grid 4.60× | grid 5.33× |

Read along a row: the winner changes with query load alone. A scan still wins **7× at 2 048
items** if you barely query it, and loses at 128 if you query hard. Any single-number answer is a
point on this surface mistaken for the surface. The kit now splits it into an unconditional floor
— set from the case *least* favourable to a scan, so it can only ever override a wrong choice —
and a load-aware rule that sees the query count.

### 10.3 Bisection assumes monotonicity; a noisy predicate has none

The calibration tool found that threshold by bisecting on "does the index win at n?". Near the
crossover the two costs are within measurement noise, so the predicate flips at random and the
search walks off. Its own printed trace said exactly that, and had been read as a result:

```
527  scan     975  index     927  scan     935  scan     942  index
```

Replaced with a ladder: every rung measured and printed, the answer being the largest population
where the scan wins on *every* rung up to it. A single noisy flip then costs one rung of
conservatism instead of an order of magnitude. **Bisection is a search, not a measurement, and it
inherits every property it assumes.**

### 10.4 A performance gate on a shared machine is a coin flip

Two consecutive runs of the same binary, minutes apart, on a desktop at 76 % CPU behind a chat
client and an editor:

| op (untouched by any commit) | run 1 | run 2 |
| --- | ---: | ---: |
| `cull_tree3_x64` | −2.8 % | **+54.0 %** |
| `cull_octree3_x64` | −2.2 % | **+60.3 %** |

The fix is not a wider tolerance — that only moves where the coin lands. A suspected regression
is re-measured over further passes and must survive all of them, compared on the best reading:
each op's estimator is already min-of-N, so extra passes widen N exactly where it matters, and a
transient cannot be the minimum of every pass. First run of the confirming gate: **14 ops over
threshold on pass one, 1 survived.** That ratio is the argument.

### 10.5 In an interleaved sweep, position in the frame is a confound — and controls matter

A sweep that maintains several structures and then culls them all times each arm in a different
cache state. We reported a kept grid culling 1.09–1.17× faster than a rebuilt one holding the
same points at identical parameters, and "explained" it by frame position on the strength of one
experiment: swapping the two arms moved the ratio from 1.11× to 1.06×.

Rotating *every* arm's position each frame — the actual control — left the ratio unchanged. The
swap had moved the number by less than the run-to-run spread. Seven hypotheses have now been
tested. Six are refuted: identical traversal counts (so nothing algorithmic), rotation, a warm
isolated bench (1.00×), a cold isolated bench (0.97×), and equal populations asserted every
frame. The seventh — interaction with the *other* arms' cache traffic, which rotation does not
equalise — moves the ratio the right way (0.96× → 1.01/1.05/1.10× with 16 MB walked before each
arm) but not far enough, with a spread as large as the effect.

It stands as **unexplained**, deliberately. The six failed reproductions are more useful to the
next person than a fourth guess would be. **One A/B on a noisy metric is not evidence for a cause
any more than it is for an effect.**

### 10.6 A negative result that saved a feature from being built

An in-place `update` on a hash-bucketed uniform grid looked slow because of its predicate scan of
the target cell — which suggested a handle layer, the O(1) trick that makes the kit's pointer
trees 5.7× cheaper to maintain. Before building it, the cost was decomposed by holding the item
count fixed and varying only cell occupancy:

| mean items/cell | 39.1 | 4.9 | 1.3 | 1.0 |
| --- | ---: | ---: | ---: | ---: |
| stayed in its cell, ns/item | 92.4 | 72.5 | 100.8 | 98.5 |
| a fresh insert, ns/item | 80.5 | 94.7 | 148.4 | 136.1 |

Occupancy varies 39× and the cost is flat — and *highest* where cells hold one item each. The
scan is not the cost; the hash lookup is, and a handle would still have to reach the bucket.
**The feature was designed, measured against, and dropped.**

The rule that replaced it is worth more than the feature would have been: such an update *saves
the calls you do not make, not the calls you do*. It is a wash per call, and wins 8× at 10 % of
items moving and 938× at 0.1 % — entirely because those callers skip it. And the break-even is
not a constant: `f* = insert / cross` reads 0.66 at 2 000 items, 0.58 at 20 000, 0.55 at 200 000
and **0.46 at 1 000 000**, because crossing a cell degrades 2.6× with population against a fresh
insert's 1.8×.

### 10.7 Warm-start migration

An index that changes its own representation at runtime must rebuild into the new one, and
typically does so from arrival order — which is spatially arbitrary, while the structure being
discarded has spent its whole life sorting exactly those points. Handing the successor that
order instead (50 000 points):

| target | from arrival order | from the predecessor's order | speed-up |
| --- | ---: | ---: | ---: |
| uniform grid | 7 009 µs | 6 550 µs | 1.07× |
| k-d tree (median split) | 8 181 µs | 5 774 µs | **1.42×** |
| binary tree, inserts | 15 625 µs | 8 633 µs | **1.81×** |
| binary tree, bulk load | 13 205 µs | 10 508 µs | 1.26× |

Two caveats matter more than the numbers. The ordering has to be *free*: a grid hands back
Z-order for the cost of sorting its cell keys and a tree hands back DFS order, but deriving one
by sorting all N points makes the migration pay for the sort it is trying to avoid. And we could
not price it end-to-end — migrations are ~2 % of a representative run and the saving is under 1 %
of the total, against tens of percent of run-to-run noise. The per-migration figure is directly
measured; the end-to-end figure does not exist, and saying so is more useful than quoting noise.

A related trap: the first attempt to measure this reported **zero effect**, because that
workload's migrations never *left* the one backend able to supply an order. A feature that cannot
fire looks exactly like a feature that does not help. The instrumentation now counts how often
the optimisation actually ran, printed next to the timing.
