# Demo: a multi-package project, 0% → 100%, integrated to disk

This is a reproducible, copy-pasteable walkthrough of the assimilator driving a
**real multi-package Go project** all the way through the pipeline — real scan →
the bounded translate → verify → fix loop to a fixpoint → per-package output
written to disk. It runs **offline and deterministically** (the `MockTranslator` +
`MockVerifier` behind `GoEngine()`), so it is the demo you can run in CI without a
backend or credentials.

The seam rules from [architecture.md](architecture.md) hold throughout: everything
below goes through the engine's `EnginePort`; no `gala_tui` / `app/ui` types and no
`gala-acp` run-contract types appear.

## The fixture

The checked-in reference project is `app/engine/testdata/simpleproj/` — two
packages with a cross-package edge and a 1:1 source test:

```
simpleproj/
├── go.mod                 module example.com/simpleproj
├── main.go                package main — imports example.com/simpleproj/mathx
└── mathx/
    ├── add.go             package mathx — func Add(a, b int) int
    └── add_test.go        package mathx — TestAdd (a SourceTest, counts toward parity)
```

That is the smallest project that exercises *every* structural feature the
pipeline cares about: more than one package, a real `import` edge between them
(`.` → `mathx`), and both unit kinds discovered from disk (a `SourceUnit`,
`add.go`, and a `SourceTest`, `add_test.go`).

> **Why a temp-dir copy in the test.** `gala test` transpiles into a temporary
> build workspace and runs the test binary from there — it does **not** stage
> `testdata/`. So the end-to-end test materializes an identical project into a
> fresh `os.TempDir()` directory (`materializeFixture`, `goscan_test.gala`) and
> scans *that*. The checked-in `testdata/simpleproj/` tree is the human-readable
> mirror and a runnable target for the built binary.

## Run it

The demo is the single end-to-end test `TestMultiPackageEndToEnd` in
`app/engine/e2e_test.gala`. The whole suite is offline, so just:

```sh
gala test
```

Expected (engine line, names elided):

```
ok  gala-build-workspace/gen/app/engine   0.0XXs
```

To build the real binary (the UI that consumes the same port):

```sh
gala build -o gala_assimilator ./cmd/gala_assimilator
```

> On Windows the whole-workspace `gala build ./...` *pattern* can report
> "no Go files in …/gen"; build the entry point explicitly (the command above) —
> that is the canonical build and it links the full UI.

## What the demo proves, stage by stage

The test walks the exact public pipeline. Each step below is one call through
`GoEngine()`'s `EnginePort`.

### 1. Real scan → multi-package shape

```gala
val port  = GoEngine()
val model = port.Scan(ScanRequest(Root = src))   // ScanGoWorkspace over real files
```

Asserted from real disk IO:

- `model.Packages.Length() >= 2` — the module-root package (`.`, holding `main.go`)
  and `mathx`.
- a cross-package edge `. → mathx`, recovered from `main.go`'s `import` of
  `example.com/simpleproj/mathx`.
- **both** unit kinds discovered: a `SourceUnit` (`mathx/add`) and a `SourceTest`
  (`mathx/add_test`).

### 2. The bounded loop to a fixpoint: 0% → 100%

```gala
val plan  = port.Plan(model)
val start = port.Run(plan)
val final = RunToCompletion(port, start)
```

- Before advancing, `CountByDone(start) == 0` — **0% done**.
- `RunToCompletion` advances the run until `IsComplete` (no unit left in a
  transient `Queued` / `Translating` / `Verifying` phase). Each unit marches
  `Queued → Translating → Verifying → Done` through the *real* run machinery (the
  same `advance` the UI ticks).
- After the loop, `IsComplete(final)` holds and `CountByDone(final)` equals the
  unit total (`main` + `mathx/add` + `mathx/add_test` = **3**) — **100% done**.

The demo additionally checks **per-package** completeness: every unit in *every*
package reached `Done`. This is stronger than the aggregate count — it fails even
if the total were 100% but a single package had a unit settle anywhere but `Done`.

### 3. Integrate → per-package output on disk

```gala
val out = fs.MkdirTemp("", "gala_e2e").GetOrElse("")
port.Integrate(final, model, out)        // integrateProject — structure-preserving
```

The emitted tree mirrors the source packages. The mapping is the single
source → target seam `targetModulePath` (append `_gala`) and `targetUnitPath`
(`.go` → `.gala`):

```
<out>/
├── gala.mod               module example.com/simpleproj_gala
├── main.gala              (from main.go)
└── mathx/
    ├── gala.mod           module example.com/simpleproj/mathx_gala
    ├── add.gala           (from mathx/add.go)
    └── add_test.gala      (from mathx/add_test.go)
```

The **per-package proof**: the test asserts *both* packages emitted output —
the module-root `gala.mod` + `main.gala`, **and** `mathx/gala.mod` +
`mathx/add.gala` + `mathx/add_test.gala`. A missing package's output fails the
test.

> With the deterministic `MockTranslator`, each emitted `.gala` body is the
> placeholder `// mock translation` — the demo proves the **wiring and structure**
> (every package scanned, run to `Done`, and written to the right target path),
> not translation quality. Real GALA output flows through this exact same
> `integrateProject` step unchanged once a live backend drives the run; see the
> live-backend smoke runbook for that path.

## At a glance

| Stage | Call | Asserted |
|-------|------|----------|
| Scan | `port.Scan` | ≥ 2 packages, cross-edge `. → mathx`, both unit kinds |
| Plan + Run | `port.Plan` → `port.Run` | `CountByDone == 0` (0%) |
| Loop | `RunToCompletion` | `IsComplete`; every unit in every package `Done` (100%) |
| Integrate | `port.Integrate` | per-package `gala.mod` (`_gala` mapping) + `.go`→`.gala` units, **both** packages |
