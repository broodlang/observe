# observe — guidance for Claude

The vendor-neutral shipper: one registered process (`:observe`) that batches log records,
metrics and error reports and hands each batch to a `Provider` in a worker. Read
`src/observe.blsp`'s header for the design and the reasons; `README.md` for the API.

- **Nothing here may block a caller.** Every public entry point is a cast; the HTTP runs
  in a spawned worker; buffers are bounded. Keep it that way.
- **The shipper's own warnings must never re-enter the shipper.** They carry
  `{:observe true}` and the log backend refuses them — a log backend that ships a log
  about failing to ship is a loop.
- **Tests that start a shipper are `:isolated`** (`tests/observe_test.blsp`): the process
  is registered by name, so two in parallel would stop each other. Pure helpers stay in
  parallel groups.
- A `defrecord` in this package has the id `:observe/observe/<name>`; an `impl` names it
  bare. An ability from another module (`log/LogBackend`) must be imported bare with
  `(:use log :only [LogBackend])` before `impl` — a qualified name there registers under
  a name the logger never dispatches on.
- Run the one test file (the full suite is blocked on this machine) and `nest format`
  before committing. No trailing `!`, no abbreviations.
