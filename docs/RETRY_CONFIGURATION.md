# Retry Configuration

The provider automatically retries rate-limited (429) and transient server-error (5xx) responses using exponential backoff with jitter. Connection errors on idempotent requests (GET, PUT, DELETE) are also retried.

## Default behaviour

| Setting | Default | Description |
|---|---|---|
| Strategy | `backoff` | Exponential backoff with ±25% jitter |
| Initial interval | 500 ms | Wait before the first retry |
| Max interval | 60 s | Ceiling on any single wait; `Retry-After` values are capped here too |
| Max elapsed time | 300 s (5 min) | Total budget across all attempts; give up when this is exhausted |
| Retry connection errors | true | Retries broken pipe / connection reset on idempotent methods |

When the provider is waiting it logs a warning at the Terraform log level:

```
WARN API rate limited, waiting before retry
  wait_seconds = 12.5
  attempt      = 2
```

Enable Terraform logging with `TF_LOG=WARN` (or `DEBUG`) to see these messages.

## Tuning

### Provider attributes

```hcl
provider "netskope" {
  retry_max_elapsed_time = 30   # seconds — fail fast in CI
  retry_disabled         = true # disable retries entirely
}
```

### Environment variables

```bash
export NETSKOPE_RETRY_MAX_ELAPSED=30   # seconds
export NETSKOPE_RETRY_DISABLED=true
```

Provider attributes take precedence over environment variables when both are set.

## Recommended settings by context

| Context | Setting | Reason |
|---|---|---|
| Interactive apply | default (300 s) | Allows time for transient rate limits to clear |
| CI pipeline | `retry_max_elapsed_time = 30` | Fails fast; surface errors rather than blocking the pipeline |
| Parallel acceptance tests | default | Tests run with `-parallel 8`; rate limits are normal and expected |
| Disabled (debugging) | `retry_disabled = true` | Returns raw API responses immediately for diagnosis |

## NPA private app API rate limits

The NPA private app read endpoints (`GET /api/v2/steering/apps/private` and `GET /api/v2/steering/apps/private/{id}`) have the following limits:

| Endpoint | Per-second | Per-hour |
|---|---|---|
| `GET /apps/private` | 10 | 5,000 |
| `GET /apps/private/{id}` | 10 | 5,000 |

Writes (`POST`, `PUT`, `DELETE`) and all other routes are unchanged.

> **Previous limits were 30 req/s and 1,000 req/hr.** The hourly cap was the primary blocker for Terraform users because a `terraform plan` or `terraform refresh` reads every private app it manages. The new 5,000/hr limit allows roughly 80 reads per minute sustained; the lower per-second rate (10) is still above the natural read rate of a single Terraform run.

The provider does not throttle requests on the client side. It only reacts to `429` responses using the retry behaviour described above.

Things to keep in mind:

- **Limits apply per tenant.** Several workspaces or CI jobs that manage private apps in the same tenant share the same budget.
- **Every managed app costs one read per plan or refresh.** Each `netskope_npa_private_app` resource and `netskope_npa_private_app` data source issues one `GET /apps/private/{id}`; each `netskope_npa_private_apps_list` data source issues one `GET /apps/private`. For large estates, `terraform plan -refresh=false` or `-target` reduces reads where that is appropriate.
- **Backoff handles the per-second limit, not the hourly one.** If the hourly limit is exhausted, the wait for the window to reset can be far longer than the 60 s `Retry-After` cap, and the provider gives up after `retry_max_elapsed_time` (300 s by default). The run then fails with a `429` error. Increasing `retry_max_elapsed_time` does not help here; wait for the limit to reset, or reduce the number of reads, then run again.

## How the budget is enforced

Before sleeping, the provider checks whether the proposed wait would exceed the remaining budget. If it would, the request fails immediately rather than sleeping and then failing — this prevents a large `Retry-After` value from blocking a pipeline for its full duration when time has already run out.

The `Retry-After` header value is also capped to `MaxInterval` (60 s by default), so a misbehaving API cannot force a wait longer than the configured ceiling regardless of what it returns.
