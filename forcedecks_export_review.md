# ForceDecks export script review

## High-impact issues found

1. **Credential loading is currently broken**
   - `Sys.getenv()` is being called with the secret values as variable names, e.g. `Sys.getenv("d8z...")`.
   - This will almost always return an empty string unless those exact (incorrect) variable names exist.
   - It should use `VALD_CLIENT_ID`, `VALD_CLIENT_SECRET`, and `VALD_TENANT_ID`.

2. **Secrets are exposed in source comments and checks**
   - Real-looking credentials appear directly in the script (including commented fallback values and placeholder checks).
   - Rotate the secret immediately in VALD Hub and remove all hard-coded values from the script/history.

3. **Latest CMJ/IMTP selection should sort by parsed datetime**
   - `slice_max(order_by = recordedutc)` can be wrong when `recordedutc` is not a true datetime type.
   - Safer: compute `recordedutc_parsed <- parse_utc(recordedutc)` and use that for ordering.

4. **Date window note**
   - The configured start date is `2026-01-01T00:00:00Z`. If you expected historical exports, this excludes all tests before January 1, 2026.

## Recommended patch (credential block)

```r
use_saved_credentials <- FALSE

if (!isTRUE(use_saved_credentials)) {
  vald_client_id     <- Sys.getenv("VALD_CLIENT_ID", unset = "")
  vald_client_secret <- Sys.getenv("VALD_CLIENT_SECRET", unset = "")
  vald_tenant_id     <- Sys.getenv("VALD_TENANT_ID", unset = "")
  vald_region        <- Sys.getenv("VALD_REGION", unset = "euw")

  missing_creds <- nchar(vald_client_id) == 0 ||
                   nchar(vald_client_secret) == 0 ||
                   nchar(vald_tenant_id) == 0

  if (missing_creds) {
    stop(
      "VALD credentials are missing. Set VALD_CLIENT_ID / VALD_CLIENT_SECRET / ",
      "VALD_TENANT_ID as environment variables, or set use_saved_credentials <- TRUE.",
      call. = FALSE
    )
  }

  valdr::set_credentials(
    client_id     = vald_client_id,
    client_secret = vald_client_secret,
    tenant_id     = vald_tenant_id,
    region        = vald_region
  )
}
```

## Recommended patch (latest per player)

```r
pick_latest_per_player <- function(df, type_filter_fn) {
  df |>
    filter(type_filter_fn(testtype)) |>
    mutate(recordedutc_parsed = parse_utc(recordedutc)) |>
    filter(!is.na(recordedutc_parsed)) |>
    group_by(profileid) |>
    slice_max(order_by = recordedutc_parsed, n = 1, with_ties = FALSE) |>
    ungroup() |>
    select(-recordedutc_parsed)
}
```
