# Research — empirical studies behind the vectorial-hash index

This folder holds the **methodology, raw results, and analysis** behind the
performance claims for the vectorial-hash spatial-index scheme (the paper:
[`../Multidimensional vector index.pdf`](../Multidimensional%20vector%20index.pdf)).
It is the *record of the investigation* — the path taken to reach the published
conclusions — kept here with the theory so the studies stay attributable to this
work and can't be re-published as someone else's.

The runnable reference implementation (Rust) is a separate project,
**vectorial-hash-kit**; its user-facing docs keep the *summary conclusions and
recommendations* ("use structure X under conditions Y") and headline comparatives,
and link back here for the full derivation.

## Contents

| File | What |
| --- | --- |
| [`BENCHMARKS.md`](BENCHMARKS.md) | Template-driven culling: methodology + full reproducible result tables (single-template, per-cell-size selection, descent vs neighbour walk, granularity-as-fallback, figure↔grid scale equivalence, full dynamic critters workload). |
| [`UPDATE_STRATEGIES.md`](UPDATE_STRATEGIES.md) | Empirical comparison of the `Tree::update` relocation strategies (Legacy / Lca / LcaRopes), the `neighbors`-feature cost, and the IntegerTree bit-shift experiment — the 135-cell formal sweep + analysis. |
| [`STORAGE_AND_SCALE.md`](STORAGE_AND_SCALE.md) | Large / persistent / on-disk workloads and structure selection: sorted space-filling-curve key over a KV (cell-probe vs the naïve-range trap), a real redb cold store, B-tree vs LSM engines, layering for sparse worlds, Morton vs Hilbert locality, sorted-key compression, an LBVH built from Morton codes, and a churn/crossover-driven structure advisor. |
| [`LESSONS_LEARNED.md`](LESSONS_LEARNED.md) | Honest, measurement-driven self-critique: where the original thesis did **not** hold up (the binary split rarely wins outright; templates/raster are conditional; the biggest win is a known technique — the contribution is the *measurement*; the memory wall beats several precompute ideas; the GPU is no free lunch for moving data), plus threats to validity. |
| [`benchmarks/`](benchmarks) | The raw data (`sweep_*.csv`) and the PowerShell drivers/analysers (`sweep_*.ps1`, `analyze_*.ps1`, `run_night.ps1`) that produced those tables. |

## Reproducing

The sweep/analysis scripts drive the **vectorial-hash-kit** binaries (`vh bench-*`,
`critters_headless`) — they are archived here for provenance and are run from a
checkout of that implementation repo, not standalone. Each `sweep_*.ps1` writes its
`sweep_*.csv`; the matching `analyze_*.ps1` reduces it to the tables in the two
Markdown studies.
