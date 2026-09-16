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

(log/add-backend (observe/log-backend {:min-level :info}))   ; every log line
(observe/attach-hatch)                                       ; hatch request timings + crashes

(observe/report caught-error {:action "nightly-sync"})       ; a caught error
(observe/gauge "queue_depth" 42 {:queue "mail"})
(observe/counter "signups")
(observe/timing "render" 12.5 {:page "home"})

(observe/stats)      ; {:pending {…} :in-flight #{} :shipped n :failed n :dropped n}
(observe/flush-sync) ; drain before a shutdown
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
`{:ts :level :message :meta}`, metrics `{:name :kind :value :tags}` (`:gauge` /
`:counter` / `:timing`), reports `{:ts :name :message :backtrace :action :namespace :tags
:params}`. A raise inside an op counts as a failure. `(observe/memory-provider)` records
everything it is handed — the thing to test a new provider, or an app's wiring, against.

## Development

The suite is `tests/observe_test.blsp`; `nest format` before committing.
