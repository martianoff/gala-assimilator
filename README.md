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

GALA dependencies are resolved via `gala.mod` (no Bazel needed for the inner loop).

```sh
# build
gala build -o gala_assimilator ./cmd/gala_assimilator

# test (whole workspace)
gala test

# run the TUI
./gala_assimilator
```

In the TUI: pick the **source project** and **agent backend** on S1 → review the
scanned **package/unit plan** on S2 → start, and watch the **assimilation
dashboard** (S3) climb 0 → 100 %. Open a unit (S4) to see its diff, the agent's
rationale, and verify results; answer any parked blockers (S5); review the
summary and resume an interrupted run from S6.

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

Selectable on S1:

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
loop 0 → 100 %, the S1–S6 screens (mouse + keyboard, responsive, tested), the
verify gate on 1:1 source tests, blocker park/answer, and checkpoint/restore.

In progress / next: disk-backed resume UI, an end-to-end multi-package demo with
per-package commits, and a runtime smoke test of the live Claude backend.

Translation quality against the live backend is not yet validated end to end; the
mock translator emits placeholder output, so the current proof is of the
**pipeline**, not of real Go → GALA output.

## Contributing

GALA conventions and project guardrails are in **[CLAUDE.md](CLAUDE.md)**. In
short: functional, immutable-by-default, sealed types + exhaustive matching; the
engine stays transport-neutral; every supported construct ships with a fixture
that proves its output.
