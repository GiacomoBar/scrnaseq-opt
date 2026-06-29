# Optimization Plan — nf-core/scrnaseq Performance

## Objective

Reduce memory footprint, wall-time, and disk usage of the nf-core/scrnaseq pipeline (v4.1.0) without altering scientific correctness of the output.

## Repo Setup

- Private fork: `GiacomoBar/scrnaseq-opt`
- Upstream: `nf-core/scrnaseq` (tracked as `upstream` remote)
- One branch per optimization group, merged into `master` after validation

---

## Optimization Groups

### A. STAR Alignment Args (`opt/star-args`)

**Files:** `conf/modules.config`, `modules/local/star_align.nf`

| # | Change | Rationale |
|---|--------|-----------|
| A1 | Remove `--twopassMode Basic` | Two-pass alignment is designed for novel splice junction discovery in bulk RNA-seq. In scRNA-seq the reads are short (28+90bp) and we quantify known genes only. Removing this cuts ~30-40% STAR runtime and ~20% RAM. |
| A2 | Remove `--outWigType bedGraph` | BedGraph output is never used downstream by the pipeline. Saves significant I/O on large genomes. |
| A3 | Replace `gzip` with `pigz -p $task.cpus` for post-alignment compression | Compression of MTX/TSV output files is currently single-threaded. `pigz` uses all allocated CPUs for 5-10x speedup. |
| A4 | Add `--genomeSAsparseD 2` for STAR genome generation | Reduces genome index memory footprint by ~30% (32GB→22GB for GRCh38) at negligible speed cost. |

**Validation:**
- Output h5ad matrices must be bit-identical to baseline
- Compare `peak_rss` and `realtime` from Nextflow trace

---

### B. Python Memory Optimization (`opt/python-memory`)

**Files:** `modules/local/templates/mtx_to_h5ad_star.py`, `modules/local/templates/mtx_to_h5ad_simpleaf.py`, `modules/local/templates/concat_h5ad.py`, `modules/local/templates/mtx_to_h5ad_kallisto.py`, `bin/filter_gtf_for_genes_in_genome.py`

| # | Change | Rationale |
|---|--------|-----------|
| B1 | `mtx_to_h5ad_star.py`: Remove `.X.copy()` for count layer | The sparse matrix is not mutated after assignment. `.copy()` doubles peak RAM for no reason. |
| B2 | `mtx_to_h5ad_simpleaf.py`: Remove duplicate sort+`.copy()` | Identical sort+copy is performed twice. Remove the second occurrence. |
| B3 | `concat_h5ad.py`: Iterative concatenation instead of loading all h5ad into dict | Current approach loads N files fully into memory, then concatenates. Iterative approach: concat in batches of 10, reducing peak from O(2N) to O(N). |
| B4 | `mtx_to_h5ad_kallisto.py`: Reduce intermediate copies in lamanno/nac workflows | Use single `ad.concat(..., join="outer")` call instead of multiple concat+subset patterns. |
| B5 | `filter_gtf_for_genes_in_genome.py`: Replace `g.readlines()` with `for line in g:` | Avoids loading entire GTF (1-2GB for human) into memory. Streaming iteration is O(1) in RAM. |

**Validation:**
- Output h5ad files: identical gene counts, cell counts, sparse matrix checksums
- `peak_rss` reduction measured from Nextflow trace

---

### C. I/O & Disk Optimization (`opt/io-disk`)

**Files:** `nextflow.config`, all `templates/mtx_to_h5ad_*.py`, `templates/concat_h5ad.py`

| # | Change | Rationale |
|---|--------|-----------|
| C1 | Default `publish_dir_mode` to `'link'` instead of `'copy'` | Eliminates duplication of large output files (BAMs, matrices). Halves disk usage. Users on shared filesystems benefit immediately. |
| C2 | Write h5ad with `compression="gzip"`: `adata.write_h5ad(..., compression="gzip")` | Reduces h5ad file size by 60-70% on disk. Read speed impact is negligible for downstream analysis. |

**Validation:**
- Output files accessible and readable
- Downstream tools (Scanpy, CellBender) can read compressed h5ad
- Disk usage delta measured

---

### D. Resource Label Tuning (`opt/resource-labels`)

**Files:** `conf/base.config`, `conf/modules.config`, `modules/local/mtx_to_h5ad.nf`, `modules/local/concat_h5ad.nf`

| # | Change | Rationale |
|---|--------|-----------|
| D1 | `MTX_TO_H5AD`: change to `process_single` (1 CPU) + increase memory | Python/Scanpy conversion is single-threaded. 6 CPUs allocated = 5 wasted per job. |
| D2 | `CONCAT_H5AD`: change to `process_single` + `process_high_memory` | Same rationale. Needs more RAM, not more CPUs. |
| D3 | Add resource caps: `memory = { Math.min(X.GB * task.attempt, 200.GB) }` | Prevents retry escalation from requesting impossible resources on scheduler. |
| D4 | FASTQC: add `--threads $task.cpus` and assign `process_low` | Currently single-threaded on millions of reads. Multi-threading cuts time proportionally. |

**Validation:**
- No scheduler rejections
- Jobs complete within resource limits
- Effective CPU utilization (`%cpu`) improves

---

## Branch Strategy

```
master (baseline)
├── opt/star-args        (A1-A4)
├── opt/python-memory    (B1-B5)
├── opt/io-disk          (C1-C2)
└── opt/resource-labels  (D1-D4)
```

Each branch is tested independently, then merged to master sequentially.

## Testing Protocol

### Correctness Test (every branch)
```bash
nextflow run . -profile test,docker --outdir results_test
# Compare output vs baseline results on master
```

### Benchmark Test (after all merges)
```bash
# Dataset: 3-5 samples, 10k cells each (10x public datasets)
# Capture: Nextflow trace (peak_rss, realtime, %cpu, disk)
# Compare: master baseline vs optimized master
```

### Metrics to Track

| Metric | Source | Target |
|--------|--------|--------|
| Peak RSS per process | `execution_trace.txt` | -30% for STAR, -50% for Python |
| Wall-time per sample | `execution_trace.txt` | -30% for STAR |
| Total disk usage | `du -sh results/` | -50% |
| CPU utilization | `%cpu` in trace | >80% for compute processes |

---

## Sync with Upstream

```bash
git fetch upstream
git merge upstream/master  # periodic sync
```

Only pull upstream changes that don't conflict with optimizations. If nf-core changes the same files, resolve manually.
