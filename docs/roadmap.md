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
- **Iterative translate → verify → fix loop (run-to-fixpoint).** A unit is no
  longer a single forward pass. On a verify **fail** it re-enters `Translating`
  with the verify diagnostics fed back as added context (threaded onto the
  assignment's `PriorFailure`), then re-verifies — repeating until it passes
  (→ `Done`) or a **configurable per-unit cap** (default 3, `DefaultMaxAttempts()`)
  is reached, at which point it settles `Failed` carrying the last diagnostics: a
  visible failure, never a silent give-up or an infinite spin (per the product
  invariant, a visible failure beats a wrong-but-green translation). The **run**
  loops to a fixpoint over all units; `IsComplete` is precisely "no unit in
  Queued / Translating / Verifying". Attempt count is tracked per unit
  (`UnitRun.Attempts`, round-tripped through the checkpoint) and the loop turning
  is streamed as `EvRetry` events. All in the engine, behind `EnginePort` (no UI
  types). Proven in `app/engine/loop_test.gala`: fail-then-pass → `Done` at attempt
  2; never-passes → `Failed` at the cap with diagnostics; the run reaches the
  completion fixpoint.
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
- **Real toolchain verify gate — mechanism landed.** `RealVerifier`
  (`app/engine/realverify.gala`) implements the `Verifier` contract against the
  actual toolchain: per unit it materializes the emitted target + its 1:1 source
  tests into an **isolated** scratch GALA module under `os.TempDir()`, runs
  `gala build` then `gala test`, and maps the result — compile error → `VerifyFail`
  with the build diagnostics, red test → `VerifyFail` with the test output, all
  green → `VerifyPass`. The actual toolchain stderr becomes the failure reason, so
  the fix loop re-translates against **real** errors. The process runner is an
  injected port (`ToolchainRunner`): tests drive the green / compile-error / test-
  failure mappings with a deterministic fake (`app/engine/realverify_test.gala`),
  so the suite spawns no `gala` and never risks the nested-`gala test` workspace-
  contention bug. `MockVerifier` stays the default; the real gate is opt-in
  (`ClaudeEngineRealVerify` / `SelectBackend "claude-verify"`). End-to-end
  validation is the remaining step — see below.
- **Blocker park / answer.** A unit that needs clarification parks as `Blocked`;
  the user answers it on S5 and the unit re-queues and continues. Driven through
  the real run machinery in `app/ui/s5_test.gala` via `ScriptedBlockerEngine`.
- **Checkpoint / restore.** Run state is pure data and is checkpointed after each
  transition; a migration can be stopped and restored. (`app/engine/persist.gala`,
  `app/ui/checkpoint.gala`.)
- **Multi-package 0 → 100% end-to-end demo (offline, deterministic).** One test
  stitches the whole real arc over a two-package fixture — `ScanGoWorkspace` →
  `GoEngine().Run` → `RunToCompletion` to a fixpoint → `Integrate` per package to
  disk — and asserts the seam holds: ≥ 2 packages with the cross-package edge, every
  unit in *every* package reaching `Done` (0% → 100%), and per-package integrated
  output (`gala.mod` with the `_gala`-mapped module + `.go` → `.gala` units).
  `TestMultiPackageEndToEnd` (`app/engine/e2e_test.gala`), runbook in
  [docs/demo.md](demo.md). Offline against `MockTranslator`/`MockVerifier`, so it
  proves the wiring and structure — not translation quality.
- **Live-backend smoke harness.** `cmd/smoke` is the one place the live Claude
  translator is driven against the real `claude` CLI end to end: select a live
  backend, scan a tiny bundled fixture, run the loop to its fixpoint, print a report,
  and exit non-zero unless every unit reached `Done`. It is a `cmd` (not a
  `_test.gala`), so `gala build ./...` compiles it but the offline suite never
  invokes it; the report/predicate it leans on (`SmokeReport` / `SmokeSucceeded`) are
  covered offline in `app/engine/smoke_report_test.gala`. Runbook + `claude` /
  `claude-verify` selector in [docs/smoke.md](smoke.md).
- **Real verify gate exercised for real + fix loop recovery.** A deterministic
  `ReferenceTranslator` (`app/engine/reference_translator.gala`) emits real,
  compilable, semantics-preserving GALA for a curated `Add` fixture — wired behind
  the same translator seam a live backend uses. A runnable harness (`cmd/verifygate`)
  drives it through the **real** `gala` toolchain gate and proves three things end to
  end: the golden translation clears `gala build`/`gala test` (`VerifyPass`); a
  deliberately wrong translation is rejected with the **real** toolchain diagnostics
  (`VerifyFail`); and the **fix loop** re-translates against those diagnostics and the
  unit reaches `Done`. An offline golden test pins the reference translation so drift
  fails the suite without spawning `gala`; the real-toolchain run is kept out of the
  default suite (the nested-`gala test` contention bug). Runbook in
  [docs/verify-gate.md](verify-gate.md). Proves the *gate wiring* on one fixture —
  broad translation quality across real projects (live backend) remains open.

## In progress / gaps

- **Disk-backed resume UI.** Checkpoint/restore exists as engine state; the
  launch-screen **Resume vs. Start fresh** choice — reading a checkpoint off disk at
  startup — is not yet wired through the UI.
- **Per-package commits during a live migration.** The offline demo above proves
  the multi-package arc end to end and integrates per-package output to disk
  (`docs/demo.md`); what remains is committing per package *as it completes* during a
  real run, demonstrated start to finish against a live backend.
- **Live backend run at scale.** The live translator is wired and now exercised by a
  runtime smoke (`cmd/smoke`, above) that drives one tiny unit through the real
  `claude` CLI to `Done`. What remains is running it over a *non-trivial*
  multi-package project — the smoke proves the live loop *finishes*, not that a real
  project migrates 0 → 100% against the live backend. The CI suite stays offline
  against `MockEngine` by design.
- **Real verify gate — end-to-end validation pending the live backend.** The
  mechanism has landed (see Done: `RealVerifier`, `app/engine/realverify.gala`): it
  materializes a scratch module and runs `gala build`/`gala test`, mapping the
  result onto `VerifyResult` with the real diagnostics. What remains is exercising
  it **for real**, which is gated on the backend emitting real GALA — under the
  default `MockTranslator` a unit's target is the placeholder `// mock translation`,
  which won't compile, so the real gate stays opt-in (not the default) until the
  live-backend round. Open sub-items for that round: confirm the scratch-module
  layout (the provisional `gala.mod` / flat-file shape in `realverify.gala`) builds
  a real translated unit, and resolve how a unit's dependencies are supplied to the
  scratch build.
- **Translation quality vs. the live backend not yet validated end-to-end.** The
  live Claude backend *does* emit real GALA, but actual Go → GALA output quality
  has not been validated through the full pipeline. The current proof is of the
  **pipeline**, not of the translation.
- **Surfacing the loop in the run view (the loop's UI follow-up).** The fix loop now
  has a genuine real-failure signal end to end: the `ReferenceTranslator` emits real
  GALA, the real gate produces a true red-test `VerifyFail`, and the loop recovers to
  `Done` against it (`cmd/verifygate`, above). What remains is **surfacing the loop**
  in the TUI — per-unit attempt counts and the `EvRetry` activity-log lines in the run
  view — so a reviewer can watch a unit retry and converge.

## Definition of done for the MVP

A reviewer can check the MVP is complete against this list:

- [ ] A real multi-package Go project migrates **0 → 100%** end-to-end against the
      **live Claude backend** (not just the mock).
- [x] The **real verify gate runs `gala build` / `gala test`** per unit —
      `RealVerifier` (`app/engine/realverify.gala`) materializes a scratch module,
      runs the toolchain, and maps green/red onto `VerifyResult` with the real
      diagnostics. *(Mechanism landed; the box below tracks running it for real.)*
- [x] The real gate is **exercised end-to-end against a non-mock backend**: the
      deterministic `ReferenceTranslator` emits real GALA and `cmd/verifygate` drives
      it through the real `gala build`/`gala test` to a genuine `VerifyPass` — the
      unit clears the gate only because its 1:1 source test actually passes (not the
      fake runner). *(One curated fixture; the live backend at scale stays open.)*
- [ ] Emitted GALA **passes the source's own tests on a non-trivial project** — the
      migration is demonstrably semantics-preserving at corpus scale. *(Proven on the
      curated `Add` fixture via `cmd/verifygate` — see [docs/verify-gate.md](verify-gate.md);
      a non-trivial corpus via the live backend remains.)*
- [x] The **fix loop runs against the real verify gate**: a genuine red-test
      `VerifyFail` carries the real diagnostics, the loop re-translates against them,
      and the unit reaches `Done` — driven through `RunToCompletion` over the real
      gate in `cmd/verifygate` (not only scripted/test verifiers).
- [ ] The TUI **surfaces the loop**: per-unit attempt counts and `EvRetry`
      activity-log lines render on S3/S4.
- [ ] A unit with no faithful target is **flagged** (unsupported-construct /
      unverified), never silently passed or guessed — per the product invariant in
      [CLAUDE.md](../CLAUDE.md).
- [ ] **Checkpoint/restore works across a process restart** through the UI: an
      interrupted run resumes from the dependency frontier and never re-translates
      finished units.
- [x] A **multi-package 0 → 100% demo** is reproducible from documented commands —
      `TestMultiPackageEndToEnd` + the [docs/demo.md](demo.md) runbook drive real
      scan → loop → per-package integrate offline. *(Per-package **commits** during a
      live run remain — tracked in the gaps above.)*
- [x] A **live-backend smoke** drives the real `claude` CLI end to end — `cmd/smoke`
      scans a fixture, runs the loop to its fixpoint, and exits non-zero unless every
      unit reached `Done` ([docs/smoke.md](smoke.md)); kept out of the offline suite.
- [ ] The **S1–S6 screens** remain green under the UI harness
      ([testing-ui.md](testing-ui.md)) — mouse + keyboard, wide + narrow.
- [ ] `gala test` is green across the workspace and the layering invariant holds
      (engine imports no `gala_tui`/`app/ui`; UI touches the engine only through
      `EnginePort`).
