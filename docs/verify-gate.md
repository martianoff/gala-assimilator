# Running the real verify gate

The verify gate (`RealVerifier`, `app/engine/realverify.gala`) is the engine's
correctness check: per unit it materializes the emitted GALA translation plus its
1:1 source test into an **isolated scratch module under `os.TempDir()`**, runs
`gala build` then `gala test` over it, and maps the result onto a `VerifyResult`. A
red build or test becomes a `VerifyFail` carrying the **real toolchain stderr**, so
the fix loop re-translates against genuine diagnostics rather than a synthetic
message.

The offline suite exercises the gate's *mapping* through an injected fake runner
(`realverify_test.gala`) — green/red/diagnostics, no real process. What the fake
cannot prove is that a translation **actually compiles and its test actually
passes**. That proof needs the real `gala` toolchain, which we deliberately keep
out of `gala test`.

## Why it is a separate command, not a test

Invoking the real `gala` toolchain from *inside* `gala test` risks the
workspace-contention bug: concurrent `gala build`/`gala test` invocations share a
CWD-local gen/deps workspace and corrupt each other (the symptom is spurious
`go.sum` / "no Go files in gen" errors). So the real-toolchain proof lives in a
standalone, runnable command — `cmd/verifygate` — that never runs during the suite.
`RealVerifier` always builds and tests in its own `os.TempDir()` scratch module
(never the project workspace), so the harness's `gala` invocations are isolated.

## What the harness proves

`cmd/verifygate` drives one curated, semantics-preserving fixture
(`ReferenceTranslator` / `referenceFixture` in `app/engine/reference_translator.gala`)
through the real gate and asserts three things end to end:

1. **A real `VerifyPass`** — the golden GALA translation of a tiny Go `Add` unit
   clears `gala build` *and* `gala test` (its 1:1 test `Add(2, 3) == 5` passes).
2. **A real, diagnostic-carrying `VerifyFail`** — a deliberately wrong translation
   (`Add` as subtraction) compiles but fails its test, and the gate's `VerifyFail`
   carries the actual toolchain output (`expected 5, got -1`, `--- FAIL: TestAdd`).
3. **A real fix-loop recovery** — driven through the engine's actual
   `RunToCompletion` loop against the real gate: the first attempt emits the wrong
   translation (red), `retryOrFail` re-translates with the real diagnostics fed
   back as a `PriorFailure`, the second attempt emits the golden translation
   (green), and the unit settles `Done`.

The fixture is the product-invariant fixture CLAUDE.md asks for: a translation
proven semantics-preserving by building and running its replicated source test.

## The fixture

| Piece | Value |
|-------|-------|
| Go unit under migration | `func Add(a, b int) int { return a + b }` |
| Golden GALA translation | `func Add(a int, b int) int = a + b` |
| 1:1 source test (GALA) | `func TestAdd(t T) T = Eq(t, Add(2, 3), 5)` |
| Wrong translation (drives the fail + fix loop) | `func Add(a int, b int) int = a - b` |

The golden translation is pinned by an **offline** test
(`reference_translator_test.gala`, `TestReferenceTranslatorEmitsGoldenGala`): if the
reference translation drifts, the default `gala test` suite fails — no toolchain
needed.

## Run it

One command, from the repo root:

```sh
gala run ./cmd/verifygate
```

Expected output (the failure diagnostics are the *real* `gala test` output for the
wrong translation — their presence is the point):

```
reference verify-gate proofs:
  [PASS] golden GALA cleared the real gate (VerifyPass)
  [PASS] wrong GALA rejected with real diagnostics (VerifyFail)
  [PASS] fix loop re-translated against diagnostics and reached Done
--- failure diagnostics (from the real toolchain) ---
gala test failed:
...
    ERROR: expected 5, got -1
--- FAIL: TestAdd (0.000s)
...
verify-gate harness: ALL PROOFS HELD
```

The harness **exits non-zero** if any proof fails, so it is usable as a gate in a
script or a non-default CI job. No credentials or network are involved — only the
local `gala` toolchain.

## What this validates (and what it doesn't)

It validates that the **real gate wiring works**: a real translation flows through
`RunToCompletion` → the real `gala build`/`gala test` → `VerifyResult`, that a real
red result carries real diagnostics, and that the fix loop recovers against them.

It does **not** validate translation quality at scale — it is one hand-written
golden fixture, not a corpus. Broad semantics-preservation across real projects
(driven by the live Claude backend) remains the open end-to-end validation item.
