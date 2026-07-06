# Storage & scale — on-disk cold indexing, space-filling-curve keys, and structure selection

Empirical studies extending the vectorial-hash scheme to **large / persistent /
on-disk** workloads and to **choosing the right structure** for a given load.
Companion to [`BENCHMARKS.md`](BENCHMARKS.md) (culling) and
[`UPDATE_STRATEGIES.md`](UPDATE_STRATEGIES.md) (relocation). All numbers are
min-of-N wall time on one machine (16 threads); each table names the
reproducing example in the implementation repo (`vectorial-hash-kit`,
`crates/vectorial-hash/examples/*.rs`).

The published scheme is an in-memory adaptive index. These studies map it onto
the **cold** end (an on-disk store of a world larger than memory) and quantify
where each structure wins, so the summary recommendations ("use X under Y") are
attributable to measurement.

---

## 1. On-disk cold index — a sorted space-filling-curve key over a KV

The in-memory `MortonGrid3` is a `HashMap<u64, Vec<T>>`: unordered, RAM-only.
For a persistent index of a world that doesn't fit in memory, the right shape is
a **sorted** Morton/Hilbert key over a B-tree KV store (the key doubles as the
persistence primary key). A spatial box query becomes a **key-range operation**.

**The naïve single-range trap.** Scanning `[min_corner_code … max_corner_code]`
of the box is pathological: the Z-curve leaves and re-enters the box, so the
range spans almost the whole dataset. Two correct strategies: **cell-probe**
(enumerate the box's cells, probe each) or **run-decomposition** (BIGMIN /
GetNextZ — jump to the next in-box key). Measured AoI query (world 10 000³, 1 M
objects, bubble r=500), `cold_index_bench`:

| N | BTree naïve range | BTree cell-probe | MortonGrid3 (hash) | Tree3 (adaptive) |
| --- | --- | --- | --- | --- |
| 100 000 | 84.1 µs | 3.4 µs | 3.0 µs | 3.1 µs |
| 1 000 000 | 2 573.7 µs | **5.8 µs** | 8.8 µs | 15.4 µs |

The naïve range is 300–2 500× slower. The **sorted cell-probe is competitive
with (here faster than) the in-memory hash grid** at 1 M — sorted keys place a
box's cells adjacent in memory (cache-friendly) where the hash scatters them.

## 2. Real on-disk store (redb B-tree) — cost of persistence

`cold_store_redb`, 1 M objects, coarse-cell key, cell-probe query:

```
WRITE  5.6 M obj/s   file 67.9 MB on disk
QUERY  cold (fresh open) 79 µs   warm (page cache) 12.7 µs
```

Persistent + unbounded, yet the warm query is only ~1.5× the in-memory grid —
**persistence is not a speed sacrifice** for a throttled "what to load" query.

## 3. Engine: B-tree vs LSM (`cold_store_engines`, 1 M)

| engine | write M obj/s | size MB | query cold µs | query warm µs |
| --- | --- | --- | --- | --- |
| redb (COW B-tree) | 6.1 | 67.9 | 66.6 | **12.3** |
| fjall (LSM) | 5.5 | **50.5** | 51.4 | 21.7 |

B-tree wins **warm reads** (point lookup vs LSM's multi-level check); LSM wins
**on-disk size** (compact SSTables). Write is ~equal *here* because the test is
a one-shot bulk-sorted load (the friendly case for both); the LSM's write
advantage appears under sustained **random/incremental** writes. Pick by the
store's real write:read ratio.

## 4. Layering for a sparse world (`cold_layered_bench`)

A single fixed cell level wastes a probe on every empty cell of a query box over
the void. A coarse **occupancy tier** (skip empty coarse cells in O(1); the
Morton prefix *is* the hierarchy) fixes it. World 100 000³, 1 M objects,
region-load query, layered vs fixed-fine:

| populated | fixed-fine (probes, µs) | layered (probes, µs) | speedup |
| --- | --- | --- | --- |
| dense (100 %) | 30 454 / 455 | 29 892 / 623 | 0.7× |
| 30 % | 30 454 / 568 | 9 307 / 178 | 3.2× |
| 10 % | 30 454 / 455 | 3 550 / 62 | 7.4× |
| sparse (3 %) | 30 454 / 714 | 1 113 / 31 | **22.7×** |

An **adaptive** win: up to 22.7× over sparse space, but 0.7× (overhead) when
dense — cheap coarse-skip where sparse, no benefit where full. This is the
S2/H3/tile-pyramid design.

## 5. Curve locality — Morton (Z-order) vs Hilbert (`cold_index_bench`)

Contiguous key-runs a box of side *s* cells maps to (fewer = better locality →
fewer/cheaper range scans; a verified Skilling Hilbert3 encoder):

| box side | Morton runs | Hilbert runs | Morton/Hilbert |
| --- | --- | --- | --- |
| 4 | 25 | 16 | 1.62× |
| 8 | 114 | 65 | 1.75× |
| 16 | 462 | 249 | 1.86× |
| 32 | 1904 | 993 | **1.92×** |

Hilbert has ~1.6–1.9× better locality (advantage grows with box size) at a
pricier encode → default Morton (cheap re-key), offer Hilbert for read-heavy
dense zones, behind a key trait.

## 6. Key compression (`key_compression_bench`)

Sorted Morton keys share long prefixes → delta + Frame-of-Reference bit-packing
shrinks them; decode is a few int ops/key. 1 M keys:

| distribution | key bits | raw KB | packed KB | ratio |
| --- | --- | --- | --- | --- |
| uniform (worst) | 21 | 7 812 | 5 631 | 1.39× |
| uniform (worst) | 12 | 7 812 | 2 335 | 3.35× |
| clustered (real) | 21 | 7 812 | 4 762 | 1.64× |
| clustered (real) | 12 | 7 812 | 1 466 | **5.33×** |

Compression tracks delta entropy: coarser keys + denser/clustered data → more.
Decode ~5 ns/key. (The Lucene-BKD trick; combines with more tricks for higher
ratios.)

## 7. LBVH — a BVH built from Morton codes (`lbvh_bench`)

Sort by Z-order, split at the highest differing bit (Karras topology), refit
AABBs. vs `Tree3::bulk_load`, world 10 000³, bubble r=500:

| N | LBVH build | Tree3 bulk build | LBVH cull | Tree3 cull |
| --- | --- | --- | --- | --- |
| 100 000 | 5.6 ms | 27.3 ms | 2.58 µs | 3.05 µs |
| 1 000 000 | **66.8 ms** | 311.5 ms | 13.9 µs | 14.1 µs |

~5× faster build for competitive query — a win for rebuild-heavy static builds,
and the construction is fully data-parallel (the GPU broad-phase route, reusing
the Morton encoding).

## 8. Structure selection — churn, crossover, and a self-tuning advisor

**Relocation rate** (`churn_relocation_bench`, 200 k moving, leaf ≈ 345 wu): the
fraction of moves that leave their leaf (the keep-index's expensive path).

| per-tick move | = × leaf | relocations |
| --- | --- | --- |
| 7 wu | 0.02× | 3.5 % |
| 34 wu | 0.10× | 16.4 % |
| 103 wu | 0.30× | 44.0 % |
| 342 wu | 1.00× | 89.1 % |

At realistic small moves only 3–16 % relocate → the keep-index is near-optimal;
a looser/coarser structure only pays for fast movers (≥ ~0.2–0.3× leaf/tick).

**Brute-vs-index crossover** (`threshold_bench`, single AoI query, pre-built):
brute linear scan wins to ~1 000 points; `Tree3::cull` overtakes ~1 500 (its
descent + result-alloc is ~100 ns fixed; a scan is ~1 ns/point). Paying the
build cost (few queries per rebuild) drops the crossover to the low hundreds.

**The advisor** (`vectorial_hash::advisor`): a `SpatialProfile` accumulates the
relocation rate + query:move ratio (EMA) and `recommend()` returns
`BruteForce` / `KeepIndexTree` / `CoarserOrRebuild` by these crossovers — a
*self-tuning* index that picks the structure per region/layer from local rates
rather than a global guess.

---

## Summary recommendations

- **Cold / on-disk index:** sorted Morton (or Hilbert) key over a KV; query by
  cell-probe or BIGMIN, never a naïve range. Layer a coarse occupancy tier for
  sparse worlds. Compress the sorted keys. Engine by write:read (B-tree
  read-heavy, LSM write-heavy/space).
- **In-memory:** the adaptive keep-index tree for moving points; a Morton grid
  for rebuild-per-frame; LBVH for fast static/parallel builds; brute force below
  ~500–1 000 items. Let the advisor pick from measured local rates.
