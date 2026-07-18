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
| [7](#results-7--gpu-sort-the-on-gpu-lbvh-build-and-the-keeprebuild-crossover-2026-07-17) | GPU: radix sort · on-GPU LBVH build · keep↔rebuild crossover + adaptive · quantised nodes | GPU radix **8–17× the CPU at scale** (hierarchical scan + 8-bit/4-pass width; bitonic was ~2× slower); a whole LBVH builds **GPU-resident in ~4.4 ms/frame at 1 M** (verified by traversal-vs-brute); moving-data crossover at **f\* ≈ 30 % → 2.8 % moving** as N grows 262k→4M; adaptive-with-hysteresis beats both pure strategies; **quantised u16 BVH nodes are 1.6× smaller and EXACT** (footprint, not latency). |

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

### 7.1 GPU sort of Morton codes — bitonic (negative) → radix (8–17× the CPU)

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

  So the GPU radix is now **8–17× the CPU** at scale. A *true* single-pass **Onesweep**
  (one decoupled-lookback scan) would cut it further, but it needs guaranteed
  inter-workgroup **forward progress**, which WebGPU/WGSL does **not** promise (it can
  deadlock) — so 8-bit/4-pass is the *portable* step in that direction, and the honest
  ceiling for a spec-clean WebGPU radix. (Follow-on: fold the 8-bit width into the
  build's key-value radix, §7.2, to shave its sort stage.)

### 7.2 A whole LBVH built on the GPU — Morton → radix → Karras → refit

`gpu_lbvh_build_bench` builds an LBVH **entirely GPU-resident**, no round-trip:
Morton codes (30-bit, GPU) → the key-value radix above (codes + primitive index)
→ **Karras** hierarchy (common-prefix ranges + split, with the point index as a
tiebreaker so equal codes stay well-founded) → **AABB refit** (a leaf-init pass,
then a bottom-up climb gated by an atomic per node — only the *second* child to
arrive unions the two boxes). Verified end-to-end by **traversing the GPU-built
BVH on the CPU and comparing to brute force** over random spheres (a pass ⇒ the
hierarchy *and* the refit AABBs are correct), first try, at every size:

| points | build/frame (min of 7) | throughput |
| --- | --- | --- |
| 262 k | 2.28 ms | 115 Mpts/s |
| 1 M | **4.40 ms** | 239 Mpts/s |
| 4 M | 12.98 ms | 308 Mpts/s |

So a **1 M-point BVH rebuilds on the GPU in ~4.4 ms/frame** — down from 8.4 ms once
the radix scan went hierarchical (§7.1); small N pays a little for the extra scan
passes, big N pulls far ahead (4 M 36.3 → 13.0 ms, 308 Mpts/s). (The atomic refit
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
so **wide8-u16 is the best footprint-and-speed point**, and this is the layout to
reach for if a static / query-heavy BVH graduates into the kit (the GPU LBVH build
of §7.2 is its natural producer). This closes the compressed-node thread: *the
literature's latency win is real, and it is the wide SoA node — measured, not
assumed.*

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
