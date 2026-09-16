# observe

Ship logs, metrics and errors from a [Brood](https://broodlang.org) app to a monitoring
provider — batched, bounded, and never on the request path. Vendor-neutral: a provider is
any value with a `Provider` impl. [`observe-appsignal`](../observe-appsignal) is the first.

Brood already has the two seams every monitoring integration hangs off — `log` (a logger
whose backends are any `LogBackend` value) and `telemetry` (Erlang-style events, handlers
in an isolated listener). `observe` is the part that gets a record *out* of the process.

## Usage

```brood
(observe/start {:provider (observe-appsignal/provider {:api-key … :log-key …})
                :env "prod" :revision "abc123" :app "hive"})

(observe/attach-all)   ; everything below, in one call
```

`attach-all` is what a hatch app on `store` gets without writing a line of
instrumentation — the Phoenix/Ecto-integration idea:

| source | event | becomes |
|---|---|---|
| hatch `http/server` | `[:hatch :request :stop]` | `request_duration` timing by route, method, status |
| | `[:hatch :request :exception]` | an error report (`web` namespace) |
| hatch live / channels | `[:hatch :live :exception]`, `[:hatch :channel :exception]` | error reports |
| hatch rate limiter | `[:hatch :ratelimit :denied]` | `ratelimit_denied` count by method, path |
| `store` repo | `[:store :query :stop]` | `query_duration` timing by statement kind and table |
| | `[:store :query :exception]` | an error report with the SQL (`store` namespace) |
| the runtime | every abnormal process exit (`proc/system-monitor`) | an error report (`process` namespace), one per crash site per minute |

Each piece is also separate — `attach-hatch`, `attach-store`, `attach-crashes` — and the
rest is by hand:

```brood
(log/add-backend (observe/log-backend {:min-level :info}))   ; every log line
(observe/report caught-error {:action "nightly-sync"})       ; a caught error
(observe/gauge "queue_depth" 42 {:queue "mail"})
(observe/counter "signups")
(observe/timing "render" 12.5 {:page "home"})

(observe/stats)      ; {:pending {…} :in-flight #{} :shipped n :failed n :dropped n}
(observe/flush-sync) ; drain before a shutdown
```

Several providers at once — a local copy next to the vendor
([`observe-local`](../observe-local)), or two vendors while migrating:

```brood
(observe/fan-out-provider [(observe-local/provider) (observe-appsignal/provider {…})])
```

Every entry point is one `send` to the shipper process. It batches per kind
(`:max-batch`, default 100), flushes on a timer (`:flush-ms`, default 5 s) or as soon as a
batch fills, does the HTTP in a throwaway worker (one in flight per kind), and holds at most
`:max-buffer` (default 1000) entries per kind while the provider is unreachable — past that
the oldest half is dropped and counted. A failing provider is warned about once per streak.

## Writing a provider

```brood
(defrecord my-provider (endpoint key))

(impl observe/Provider my-provider
  (ship-logs    [p context records] …)   ; → :ok | [:error message]
  (ship-metrics [p context metrics] …)
  (ship-errors  [p context reports] …))
```

`context` is `{:hostname :env :revision :app}`. Records are
`{:ts :level :message :meta}`, metrics `{:ts :name :kind :value :tags}` (`:gauge` /
`:counter` / `:timing`), reports `{:ts :level :name :message :backtrace :action :namespace :tags
:params}`. A raise inside an op counts as a failure. `(observe/memory-provider)` records
everything it is handed — the thing to test a new provider, or an app's wiring, against.

### The vocabulary against other vendors' APIs

The three shapes were checked against the public ingestion APIs of AppSignal, Sentry,
Honeybadger, Datadog, New Relic and OTLP (2026-09), so a provider for any of them is a
mapping and not a redesign. What each needs from a record, and where it comes from:

| vendor | logs | metrics | errors |
|---|---|---|---|
| AppSignal | `/logs/json` NDJSON: `timestamp` RFC 3339, `group`, `severity`, `message`, `hostname`, flat `attributes` | `/metrics/json`: `name`, `metricType` gauge/counter/timing, `value`, string `tags` | `/errors`, one per request: `timestamp` (s), `action`, `namespace`, `error {name message backtrace}`, `revision`, `tags`, `params`, `environment` |
| Sentry | envelope `log` item, ≤100 per envelope: `timestamp` (s), `trace_id` (provider generates), `level` trace…fatal, `body`, typed `attributes` | envelope metric items (beta): `count`/`gauge`/`distribution` (timing → distribution), `timestamp` | envelope `event`: `event_id` (provider generates), `timestamp`, `platform`, `level`, `transaction` (= `:action`), `server_name`, `release` (= `:revision`), `environment`, `tags`, `extra` (= `:params`), `exception.values[{type value stacktrace.frames}]` |
| Honeybadger | Insights events `/v1/events` NDJSON: `ts` + arbitrary fields | (events) | `/v1/notices`: `error {class message tags backtrace [{method}]}`, `request {component action params}`, `server {environment_name hostname revision}` |
| Datadog | `/api/v2/logs` array: `message`, `status`, `hostname`, `service` (= `:app`), `ddsource`, `ddtags`, attributes | `/api/v2/series`: `metric`, `type` count/gauge, `points [{timestamp value}]`, `tags ["k:v"]`; timing → `/api/v1/distribution_points` | error tracking via logs: `error.kind`, `error.message`, `error.stack` |
| New Relic | Log API `[{common {attributes}, logs [{timestamp message attributes}]}]` | Metric API: `name`, `type` gauge/count/summary, `value`, `timestamp`, `interval.ms` for counts; timing → summary | Event API / a log with `error.class` and `error.message` |
| OTLP/HTTP | `logRecords`: `timeUnixNano`, `severityText`/`severityNumber`, `body`, `attributes`; resource = context | `gauge`/`sum` (delta)/`histogram` data points with `timeUnixNano` | a log record with `exception.type`, `exception.message`, `exception.stacktrace` |

Two things every record carries because of this table: a metric has a `:ts` (Datadog,
New Relic and OTLP refuse a point without one; AppSignal ignores it), and an error report
has a `:level` (`:error` by default — Sentry grades events, nobody else does). The
per-vendor extras — Sentry's `event_id` and `trace_id`, Honeybadger's `notifier`,
Datadog's `ddsource` — are a provider's own business. The buffering defaults (100 per
batch, 5 s, 1000 queued, drop beyond) are the ones Sentry's SDK specification mandates.

## Development

The suite is `tests/observe_test.blsp`; `nest format` before committing.
