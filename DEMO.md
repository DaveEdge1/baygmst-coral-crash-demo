# Demo: single sparse archive crashes the RP reducer

## What this repo shows

Selecting a **single, sparse archive type** for the reduced-proxy step crashes
the upstream RP reducer
([`paleopresto/BayGMST_R`](https://github.com/paleopresto/BayGMST_R) →
`utils/PAGES2k_reducedProxy_UNSC.R`). This template instance sets:

```yaml
# config/user_config.yml
ptype: "coral"
rp_method: "PCR"
```

and is otherwise the stock [`presto-BayGMST`](https://github.com/DaveEdge1/presto-BayGMST)
template. The `BayGMST Reconstruction` workflow downloads the archived
`Pages2kTemperature` compilation, converts it to a PAGES2k-style proxy matrix,
and then runs the reducer — which fails.

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

Coral records are sparse early in the millennium, so the **oldest** segment
retains a *single* qualifying proxy. Without `drop = FALSE`, `proxy_finite`
collapses from a matrix to a numeric vector, and `apply(..., MARGIN = 2, ...)`
fails with `dim(X) must have a positive length`.

A `drop = FALSE` patch is **necessary but not sufficient**: with it, segment 1
fits a degenerate 1-PC PCR and then dies at the RP assignment
`RP[timeSpan, k] <- cbind(one, pc_all) %*% coeff` (dimension mismatch). The
8 × 250-year segmentation has cascading assumptions that break when a selection
is too sparse.

Dense selections are unaffected: `ptype: "ALL"` and multi-archive selections
such as `ptype: ["coral", "tree"]` (202 proxies) run to completion.

Note: this is a *non-empty* selection (coral has 72 proxies in the screened
set), so the PReSto-side empty-selection guard in `PAGES2k_reducedProxy_UNSC.R`
does not fire — the crash is downstream of it, in the upstream segment loop.

## Reproduce locally (no GitHub Actions)

```bash
docker build -t baygmst:local .
# run only the reducer against the baked-in PAGES2k reference data:
docker run --rm \
  -v "$PWD/config/user_config.yml:/app/config/user_config.yml:ro" \
  -v "$PWD/results:/results" \
  --entrypoint Rscript baygmst:local /app/scripts/run_reducer.R
```

## Reproduce on the actual upstream code

Set `cfg$ptype <- "coral"` in the `config.yml` that
`paleopresto/BayGMST_R/utils/PAGES2k_reducedProxy_UNSC.R` reads, then source the
script. It crashes at the first segment as above.
