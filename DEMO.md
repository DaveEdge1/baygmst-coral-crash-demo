# Demo: sparse single-archive selection crashes the RP reducer

## What this repo shows

Selecting a **single, sparse archive type** for the reduced-proxy step crashes
the upstream RP reducer
([`paleopresto/BayGMST_R`](https://github.com/paleopresto/BayGMST_R) →
`utils/PAGES2k_reducedProxy_UNSC.R`). This template instance sets:

```yaml
# config/user_config.yml
ptype: "documents"
rp_method: "PCR"
```

and is otherwise the stock [`presto-BayGMST`](https://github.com/DaveEdge1/presto-BayGMST)
template **plus the pending `make.names`-safe-ID fix** (see *"Why the fix is
included"* below). The `BayGMST Reconstruction` workflow downloads the archived
`Pages2kTemperature` compilation, converts it to a PAGES2k-style proxy matrix,
selects the `documents` proxies, and runs the reducer — which fails.

## Expected failure

In the `reconstruct` job, step **"Run BayGMST pipeline"** (the R reducer):

```
[1] "Method to compute RP: PCR"
[1] 1
Error in apply(X = proxy_finite, MARGIN = 2,
               function(x) sum(is.na(x))/dim(proxy_finite)[1]) :
  dim(X) must have a positive length
Execution halted
```

## Root cause

The reducer cuts the record into 8 × 250-year segments and, per segment, keeps
only the proxies that extend back far enough:

```r
proxy_finite = proxy_filled[timeSpan, segProxies]   # no drop = FALSE
cleanNANs <- apply(X = proxy_finite, MARGIN = 2, ...)
```

`documents` proxies are sparse early in the millennium, so the **oldest**
segment retains a *single* qualifying proxy. Without `drop = FALSE`,
`proxy_finite` collapses from a matrix to a numeric vector, and
`apply(..., MARGIN = 2, ...)` fails with `dim(X) must have a positive length`.

A `drop = FALSE` patch is **necessary but not sufficient**: with it, a 1-PC PCR
is fitted and the run then dies at the RP assignment
`RP[timeSpan, k] <- cbind(one, pc_all) %*% coeff` (dimension mismatch). The
8 × 250-year segmentation has cascading assumptions that break when a selection
is too sparse.

### Sparsity, not the specific archive

The crash is about **how sparse the selection is**, not which archive it is.
Against `Pages2kTemperature 2_2_0`:

| `ptype`      | proxies | reducer outcome                                            |
|--------------|--------:|------------------------------------------------------------|
| `documents`  |      16 | ❌ `dim(X) must have a positive length` (shown here)        |
| `speleothem` |       5 | ❌ `dim(X) must have a positive length`                     |
| `coral`      |     152 | ❌ `RP[timeSpan,k] <- cbind(one,pc_all) %*% coeff` mismatch |
| `borehole`   |       4 | ❌ same `cbind ... %*% coeff` mismatch                      |
| `ice`        |      78 | ✅ completes                                                |
| `coral`+`tree` | 2513  | ✅ completes                                                |

## Why the `make.names`-safe-ID fix is included

The stock template has a **separate** bug that masks this one: the LiPD adapter
emitted proxy IDs with hyphens (e.g. `Ocn-Mayotte.Zinke.2008__d18O`), which
`read.csv(check.names = TRUE)` rewrites in the matrix *header* but not in
`metadata$X`. Every archive-subset selection therefore matched **zero** columns
and crashed earlier with `undefined columns selected`, never reaching the
segment loop.

This repo includes the pending fix for that
([presto-BayGMST#3](https://github.com/DaveEdge1/presto-BayGMST/pull/3)) so the
selection actually populates the proxy matrix and the **upstream segment-loop
bug** above is reached. Once #3 merges, a fresh template instance with
`ptype: documents` reproduces this directly.

## Reproduce locally (no GitHub Actions)

```bash
docker build -t baygmst:local .
docker run --rm \
  -v "$PWD/lipd_legacy.pkl:/proxies/lipd_legacy.pkl:ro" \
  -v "$PWD/config/user_config.yml:/app/config/user_config.yml:ro" \
  -v "$PWD/results:/results" \
  baygmst:local
# (download lipd_legacy.pkl from
#  https://lipdverse.org/Pages2kTemperature/2_2_0/Pages2kTemperature2_2_0.pkl)
```

## Reproduce on the upstream code directly

Set `cfg$ptype <- "documents"` (or any sparse single archive) in the `config.yml`
that `paleopresto/BayGMST_R/utils/PAGES2k_reducedProxy_UNSC.R` reads, against a
proxy matrix whose column names match `metadata$X`, then source the script. It
crashes at the first segment as above.
