# Live-backend smoke (`cmd/smoke`)

`cmd/smoke` is the one place the **live Claude translator** is driven against the
real `claude` CLI, end to end: it selects a live backend, scans a tiny bundled Go
fixture, runs the translate → verify → fix loop to its fixpoint, prints a report, and
exits non-zero unless every unit reached `Done`.

It is **deliberately excluded from `gala test`.** The whole test suite is offline and
deterministic against `MockEngine`; this smoke needs `claude` on `PATH` and valid
credentials, so it lives as a `cmd` (its own `main`), not a `_test.gala`. `gala build
./...` compiles it, but nothing in the suite ever invokes it — `gala test` stays green
with no live backend present. The engine-side rendering and pass/fail predicate it
leans on (`SmokeReport` / `SmokeSucceeded`) *are* covered offline, in
`app/engine/smoke_report_test.gala`.

## Prerequisites

- The `claude` CLI on `PATH`, authenticated (the live backend opens a real `claude`
  session per unit). If `claude` is absent or unauthenticated, the smoke does **not**
  panic: the unit settles `failed:` with the transport/auth error in the report and
  the process exits non-zero.
- A working `gala` toolchain (to build/run the cmd). For the `claude-verify` backend,
  `gala` must also be on `PATH` — the real verify gate shells out to it.

## Run it

```sh
# default: live translator + deterministic verify gate (MockVerifier)
gala run ./cmd/smoke

# or build then run the binary
gala build -o gala_smoke ./cmd/smoke
./gala_smoke
```

### Backend selector

The backend is chosen by the `SMOKE_BACKEND` environment variable (live backends
only — the smoke can never silently fall back to an offline mock):

| `SMOKE_BACKEND`        | backend                      | what runs                                                    |
| ---------------------- | ---------------------------- | ------------------------------------------------------------ |
| unset / `claude`       | `ClaudeEngine()`             | live `claude` translation + deterministic (always-pass) gate |
| `claude-verify`        | `ClaudeEngineRealVerify()`   | live `claude` translation + **real** `gala` toolchain gate   |

```sh
# real toolchain verify gate (gala build/test over the emitted GALA)
SMOKE_BACKEND=claude-verify gala run ./cmd/smoke
```

## The fixture

A one-function Go package (`func Add(a, b int) int`) bundled in the cmd as source
strings and materialized to a fresh temp dir at run time, then scanned — so the smoke
is self-contained and runnable from any directory (the built binary has no `testdata`
staged). Tiny on purpose: the smoke proves the live loop *finishes* on real input, not
test parity.

## Expected output

A concise report, then an exit code that is `0` only if every unit reached `Done`:

```
=== gala-assimilator live smoke ===
backend: claude
units:   1

  add  [done]  attempts=1
    --- emitted GALA ---
    <the GALA the model emitted for Add>

phase transitions:
    add  -> translating
    add  reading: reading source unit
    add  emitting: translating via claude
    add  -> verifying
    add  -> done

result: 1/1 units done — COMPLETE
```

If a unit stalls — a clarification (`blocked: …`), an abort, verify exhaustion, or a
missing `claude` CLI (`failed: failed to open claude session: …`) — the report shows
that terminal status, the footer reads `INCOMPLETE`, and the process exits non-zero.
Nothing is swallowed.
