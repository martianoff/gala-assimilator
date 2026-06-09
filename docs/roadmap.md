# Roadmap — MVP status

A committed view of where the migration tool stands and what's left for the MVP.
The MVP target is **Go → GALA**: emitted GALA transpiles back to Go, which gives
an objective verification signal.

This roadmap respects the layering invariant in
[architecture.md](architecture.md): the engine is transport-neutral (no
`gala_tui`, no `app/ui`) and the UI reaches it only through `EnginePort`. Every
item below is either behind that port (engine) or above it (UI) — adding a
language or backend never crosses the seam.

## Done

- **Real Go scan.** A real Go project is scanned into a workspace model —
  multi-package, with cross-package dependencies — not a fixture stub.
- **The 0 → 100% orchestration loop.** Scan → plan → run climbs the full set of
  units to completion. Proven end-to-end in `app/ui/update_test.gala`
  (`TestRunDrivesToCompletion`: every unit reaches `Done`, nothing left
  actionable).
- **S1–S6 screens, mouse + keyboard, responsive.** The interview (S1), plan (S2),
  dashboard (S3), diff/review (S4), blocker (S5), and summary/resume (S6) screens
  are tested through real input and at wide/narrow widths. See
  [testing-ui.md](testing-ui.md) for the harness; the mouse/keyboard parity is
  pinned in `app/ui/s1_interaction_test.gala`, `s3_test.gala`, `s4_test.gala`,
  and `s5_test.gala`.
- **Verify gate on 1:1 source tests.** The correctness signal is the *source's
  own* tests, replicated 1:1 (`UnitKind` `SourceTest`). Agent-added tests
  (`AddedTest`) are excluded before they can reach the verifier, so an agent can
  never validate its own output. The gate is a port (`Verifier` in
  `app/engine/verify.gala`) with a sealed `VerifyResult`
  (pass / fail / unsupported), and a unit with no source-test coverage is flagged
  unverified rather than silently passed.
- **Blocker park / answer.** A unit that needs clarification parks as `Blocked`;
  the user answers it on S5 and the unit re-queues and continues. Driven through
  the real run machinery in `app/ui/s5_test.gala` via `ScriptedBlockerEngine`.
- **Checkpoint / restore.** Run state is pure data and is checkpointed after each
  transition; a migration can be stopped and restored. (`app/engine/persist.gala`,
  `app/ui/checkpoint.gala`.)

## In progress / gaps

- **Disk-backed resume UI.** Checkpoint/restore exists as engine state; the S6
  launch-time **Resume vs. Start fresh** choice reading a checkpoint off disk is
  not yet wired through the UI.
- **End-to-end multi-package demo with per-package commits.** A full migration of
  a real multi-package project, committing per package as it completes, is not yet
  demonstrated start to finish.
- **Live Claude backend runtime smoke test.** The live translator (via `gala-acp`)
  is wired and selectable on S1, but there is no runtime smoke test exercising it
  against the real Claude CLI/API. All current tests run offline against
  `MockEngine`.
- **The real verify gate is still deferred under the mock translator.** This is
  the most important open item. The gate's *contract* is done, but the real
  `gala build` / `gala test` invocation is **not wired** — see the `DEFERRED`
  header in `app/engine/verify.gala`. The real verifier will, per unit,
  materialize the emitted target plus its 1:1-replicated source tests into a
  scratch module, run `gala test`, and map green/red onto `VerifyResult`. That
  work is gated on the default backend emitting real GALA: under the deterministic
  `MockTranslator` a unit's target text is the placeholder `// mock translation`,
  which there is nothing to compile. Until then `MockVerifier` (a deterministic
  pass) stands in, and swapping in the real verifier is a port substitution —
  the gate itself does not change.
- **Translation quality vs. the live backend not yet validated end-to-end.** The
  live Claude backend *does* emit real GALA, but actual Go → GALA output quality
  has not been validated through the full pipeline. The current proof is of the
  **pipeline**, not of the translation.

## Definition of done for the MVP

A reviewer can check the MVP is complete against this list:

- [ ] A real multi-package Go project migrates **0 → 100%** end-to-end against the
      **live Claude backend** (not just the mock).
- [ ] The **real verify gate runs `gala build` / `gala test`** per unit — the
      `DEFERRED` path in `app/engine/verify.gala` is wired and a unit only reaches
      `Done` when its 1:1 source tests genuinely pass on the emitted target.
- [ ] Emitted GALA **transpiles back to Go and passes the source's own tests** —
      the migration is demonstrably semantics-preserving on a non-trivial project.
- [ ] A unit with no faithful target is **flagged** (unsupported-construct /
      unverified), never silently passed or guessed — per the product invariant in
      [CLAUDE.md](../CLAUDE.md).
- [ ] **Checkpoint/restore works across a process restart** through the UI: an
      interrupted run resumes from the dependency frontier and never re-translates
      finished units.
- [ ] A **multi-package demo with per-package commits** is reproducible from the
      README's documented commands.
- [ ] The **S1–S6 screens** remain green under the UI harness
      ([testing-ui.md](testing-ui.md)) — mouse + keyboard, wide + narrow.
- [ ] `gala test` is green across the workspace and the layering invariant holds
      (engine imports no `gala_tui`/`app/ui`; UI touches the engine only through
      `EnginePort`).
