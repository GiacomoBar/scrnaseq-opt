# Optimization Tracker

## Status

| Branch | Fixes | Status | Date |
|--------|-------|--------|------|
| `opt/star-args` | A1, A2, A3, A4 | ✅ Pushed | 2026-06-29 |
| `opt/python-memory` | B1, B2, B3, B4, B5 | ⬜ TODO | — |
| `opt/io-disk` | C1, C2 | ⬜ TODO | — |
| `opt/resource-labels` | D1, D2, D3, D4 | ⬜ TODO | — |

## Changelog

### 2026-06-29

**Branch `opt/star-args`** — committed and pushed

- **A1** `conf/modules.config`: Removed `--twopassMode Basic` from STAR_ALIGN ext.args
- **A2** `conf/modules.config`: Removed `--outWigType bedGraph` from STAR_ALIGN ext.args
- **A3** `modules/local/star_align.nf`: Replaced all `gzip` calls with `pigz -p $task.cpus` for parallel compression. Added `pigz` to conda dependencies.
- **A4** `conf/modules.config`: Added `ext.args = '--genomeSAsparseD 2'` to STAR_GENOMEGENERATE

**Setup**

- Created private repo `GiacomoBar/scrnaseq-opt` from nf-core/scrnaseq v4.1.0
- Configured remotes: `origin` → private fork, `upstream` → nf-core
- Created `.kiro/steering/optimization-plan.md` with full optimization roadmap

## Next Steps

1. Create branch `opt/python-memory` and apply fixes B1-B5
2. Create branch `opt/io-disk` and apply fixes C1-C2
3. Create branch `opt/resource-labels` and apply fixes D1-D4
4. Run correctness test (`-profile test,docker`) on each branch
5. Merge all branches to master and run benchmark comparison
