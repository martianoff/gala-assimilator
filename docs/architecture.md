# gala-assimilator — architecture

A TUI front-end over a transport-neutral **migration engine that orchestrates AI
agents**. The engine drives a translate → verify → fix loop, per source unit,
until a project is migrated from 0 % to 100 %. The two halves are kept apart by a
stable port so a new source/target language — or a new agent backend — is added
in the engine without touching the UI, and a UI redesign never reaches into
engine internals. The first supported pair is **Go → GALA**.

## Layers (top imports down; never the reverse)

```
cmd/gala_assimilator   runnable entrypoint — picks a backend, launches the TUI (gala_tui RunWithMouse)
        │
app/ui                 gala_tui Elm app: Model / Update / View, screens (S1–S6), widgets
        │  (consumes the engine ONLY through app/engine's public EnginePort)
app/runview            UI-side projection of engine RunState → plain view structs
        │  (gala_tui-free; isolates engine sealed-type matching — see "Seam notes")
app/engine             the orchestration engine — transport-neutral, no gala_tui / app/ui imports
        ├─ acp      agent-run contract: gala-acp's generic AgentRunner[A,R,C] +
        │           RunTranscript / RunOutcome, instantiated with engine payloads
        ├─ lang     source frontend: ScanGoWorkspace reads a real Go project → WorkspaceModel
        ├─ verify   verification harness: gates a unit on its 1:1-replicated source tests
        └─ state    run state + checkpoint (save / restore a long migration)
```

(`acp` / `lang` / `verify` / `state` are concerns within the single `engine`
package, not separate sibling packages — see "Seam notes" for why.)

### Invariants

- **`app/engine` imports no `gala_tui` and no `app/ui` types.** It speaks in
  workspace-model values, the agent-run contract, verification results, and
  serializable run state — pure data. This is what lets it be unit-tested without
  a terminal and reused from a future headless/CLI mode.
- **`app/ui` reaches the engine only through its public `EnginePort`** — a small
  surface: `Scan`, `Plan`, `Run`, `Advance`, `Events`, `ActOnUnit`,
  `AnswerBlocker`, `Checkpoint`/`Resume` (+ disk save/load). No direct calls into
  engine internals.
- **The agent-run contract (`acp.*`) never crosses the port.** The engine adapts
  gala-acp's generic types to its own domain values at the boundary, so swapping
  the agent backend (mock ↔ live Claude) touches nothing in the UI.
- **Adding a language = adding a frontend/backend behind the existing port.** If a
  new language forces a UI change, the seam has leaked — fix the seam, not the UI.

## The assimilation loop (0 → 100 %)

1. **Interview** — collect every answerable decision up front (target conventions,
   unsupported-construct policy, retry budget, agent backend) so the run is
   unattended. Front-load questions; interrupt only on a genuinely new blocker.
2. **Scan** — `ScanGoWorkspace` reads the real Go project: discovers packages
   (honoring `go.mod`), enumerates `.go` files as units, classifies `*_test.go`
   as 1:1 source tests, and builds intra- and cross-package dependency edges.
3. **Plan** — order units (dependencies first) and attach each unit's real source
   text + dependency context.
4. **Translate** — the orchestrator dispatches a unit to a Translator agent over
   the agent-run contract (mock translator offline; live Claude backend via
   gala-acp's `agent` transport).
5. **Verify** — the unit reaches `Done` only when its **1:1-replicated source
   tests** pass; agent-added tests are excluded from the gate. Failure → `Failed`
   with a named reason; an untranslatable source test → flagged, never a silent
   pass.
6. **Fix / park / advance** — failures feed back for bounded retries; a unit that
   needs a human answer **parks** (a `Clarification`) while the run continues on
   other units; the answer is threaded back on a fresh assignment.
7. **Checkpoint** — run state is pure data and is persisted (atomically) after
   transitions, so a long migration survives quit/crash and resumes without
   redoing finished units.

## TUI shape (gala_tui)

Elm triple: a single `Model`, a pure `Update(Model, Msg) -> (Model, Cmd)`, a pure
`View(Model) -> Widget`. Long-running engine work runs as `Cmd[T]` (FutureCmd /
Async) and streams progress back as messages — the Update loop never blocks.
Screens are routed with `ScreenStack`. Every control is reachable by **both mouse
and keyboard**, the layout is responsive (with a minimum-size notice), and each
screen ships render + interaction tests.

Screens: **S1** Setup/Interview · **S2** Scan & Plan review · **S3** Assimilation
dashboard (split board: unit tree + live agent log + progress) · **S4** Unit/diff
review (diff + agent rationale + verify) · **S5** Blocker/clarification · **S6**
Report (+ resume entry).

## Seam notes (GALA toolchain)

- **`app/runview`** holds the projection of the engine's `RunState`/`UnitStatus`
  into plain view structs. Engine sealed-type pattern-matching lives here, in a
  gala_tui-free package, to avoid a sealed-case name collision with gala_tui's
  own `state.Done`.
- **Single `engine` package.** Engine concerns are kept in one package rather than
  sibling sub-packages because the GALA 0.49.2 build does not reliably rewrite
  intra-module self-imports; consumer→library imports (UI→engine) work fine.
- The live agent backend builds on **gala-acp** (`github.com/martianoff/gala-acp`)
  — its generic run contract is the engine-internal run machinery, and its
  `agent` subpackage provides the Claude transport.
