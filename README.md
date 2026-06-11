# gala-assimilator

A terminal UI that **migrates a project from one programming language to another
by orchestrating AI agents** — driving a translate → verify → fix loop, per source
file, until the project reaches 100 % migrated. It is written in
[GALA](https://github.com/martianoff/gala-tui) (which transpiles to Go) and built
on the `gala_tui` Elm-architecture framework.

The first supported pair is **Go → GALA**, chosen because emitted GALA transpiles
back to Go, giving an objective verification signal.

> **Status: MVP in progress.** The full pipeline runs end to end on real Go input
> — scan → plan → run climbs 0 → 100 % in the TUI, with diff review, blocker
> handling, a verify gate, and checkpoint/restore. Translation runs against a
> deterministic **mock** translator by default; a **live Claude backend**
> (via `gala-acp`) is wired and selectable. See [Status](#status) for specifics.

## The product invariant

A migration must **preserve semantics**. A translation that compiles but silently
changes behavior is the worst possible output. So:

- A unit is only `Done` when its **own tests, replicated 1:1 from the source**,
  pass. Tests the agent adds for extra coverage **never** count toward
  correctness — that would let an agent validate its own output.
- A construct with no faithful target is **flagged**, never guessed.

## Build, test, run

You need the **GALA toolchain** (`gala`) on your `PATH`. Run the commands from the
**repository root** — that's where `gala.mod` lives, and GALA resolves dependencies
from it (no Bazel needed for the inner loop).

```sh
# build (build the entry point explicitly — see the note below)
gala build -o gala_assimilator ./cmd/gala_assimilator

# test (whole workspace)
gala test

# run the TUI
./gala_assimilator
```

> Build the **entry point** (`./cmd/gala_assimilator`), not the whole-workspace
> pattern. `gala build ./...` can fail with `no Go files in …/gen` on some platforms
> (notably Windows); the explicit entry-point build above is the canonical one and
> links the full UI.

Two extra runnable harnesses (outside `gala test`, documented in `docs/`):

```sh
# real verify gate over a curated fixture (real `gala build`/`gala test`)
gala run ./cmd/verifygate          # docs/verify-gate.md

# live Claude backend smoke (needs the `claude` CLI authenticated)
gala run ./cmd/smoke               # docs/smoke.md
```

In the TUI: pick the **source project** and **agent backend** on the **setup**
screen → review the scanned **package/unit plan** → start, and watch the **run
dashboard** climb 0 → 100 %. Open a unit to see its **diff**, the agent's
rationale, and verify results; answer any parked **blockers**; then review the
**summary** and resume an interrupted run from there.

## How it works

```
cmd → app/ui (gala_tui)  →  app/engine (orchestrator behind a stable EnginePort)
                              ├─ agent-run contract (gala-acp): mock / live Claude
                              ├─ Go frontend: scan a real project → workspace model
                              ├─ verify: gate each unit on its 1:1 source tests
                              └─ state: checkpoint / restore a long migration
```

The engine is **transport-neutral** (no UI/terminal imports) and the UI talks to
it **only through `EnginePort`** — so a new language or agent backend is added in
the engine without touching the UI. See **[docs/architecture.md](docs/architecture.md)**
for the layering, the assimilation loop, and the seam invariants.

## Agent backends

Selectable on the setup screen:

- **Mock** — deterministic, offline. The default for tests and demos; no network.
- **Go (real scan)** — reads the real Go project and runs it through the mock
  translator (proves the pipeline on real input).
- **Claude** — the live translator via `gala-acp`'s `agent` transport (needs the
  Claude CLI/API at runtime).

## Save & restore

A migration can be long, so run state is pure data and is checkpointed
(atomically) after each transition. On launch, an existing checkpoint offers a
**Resume** vs **Start fresh** choice; resuming continues from the dependency
frontier and never re-translates finished units.

## Status

Working: real Go scanning (multi-package, cross-package deps), the orchestration
loop 0 → 100 %, all screens (setup → plan → run dashboard → unit/diff → blockers →
summary; mouse + keyboard, responsive, tested), the verify gate on 1:1 source
tests, blocker park/answer, and the run dashboard surfacing the fix loop (attempt
counts, retry log lines) and correctness flags (unsupported / unverified).

Proven offline and reproducibly:

- a multi-package Go project driven **0 → 100 %** end to end and integrated
  per-package to disk ([docs/demo.md](docs/demo.md));
- the **real verify gate** running `gala build`/`gala test` over a curated
  semantics-preserving fixture — a true pass, a real diagnostic-carrying failure,
  and a fix-loop recovery ([docs/verify-gate.md](docs/verify-gate.md));
- **disk-backed resume across a process restart** — Resume vs. Start fresh at
  launch, resuming from the dependency frontier without re-translating finished
  units;
- a runnable **live-backend smoke** that drives the real Claude CLI end to end
  ([docs/smoke.md](docs/smoke.md)).

In progress / next: a live Claude run at **project scale** (the smoke proves the
live loop on one unit), per-package commits during a live migration, and
**translation-quality** validation across a real corpus. The mock translator emits
placeholder output, so the offline proof is of the **pipeline**, not of real
Go → GALA translation quality — see [docs/roadmap.md](docs/roadmap.md) for the
detailed checklist.

## Contributing

GALA conventions and project guardrails are in **[CLAUDE.md](CLAUDE.md)**. In
short: functional, immutable-by-default, sealed types + exhaustive matching; the
engine stays transport-neutral; every supported construct ships with a fixture
that proves its output.
