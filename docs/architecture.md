# gala-assimilator — architecture

A TUI front-end over a transport-neutral migration engine. The two halves are
kept apart by a stable port so a new source/target language is added in the
engine without touching the UI, and a UI redesign never reaches into engine
internals.

## Layers (top imports down; never the reverse)

```
cmd/gala_assimilator      runnable entrypoint — wires engine + UI, calls gala_tui RunFull
        │
app/ui                    gala_tui Elm app: Model / Update / View, screens, widgets
        │  (consumes the engine ONLY through app/engine's public port)
app/engine                migration engine — transport-neutral, no gala_tui imports
        ├─ frontend/       source-language readers:  source text → AST → IR
        ├─ ir/             language-neutral intermediate representation + passes
        └─ backend/        target-language emitters:  IR → target source
```

### Invariants

- **`app/engine` imports no `gala_tui` and no `app/ui` types.** It speaks in IR,
  diagnostics, and migration-result data — pure values. This is what lets it be
  unit-tested without a terminal and reused from a future headless/CLI mode.
- **`app/ui` reaches the engine only through its public port** (a small surface:
  "scan project", "plan migration", "run migration", "stream progress/diags").
  No direct calls into `frontend`/`ir`/`backend` internals from the UI.
- **Adding a language = adding a frontend and/or backend behind the existing
  port.** If a new language forces a UI change, the seam has leaked — fix the
  seam, not the UI.

## The migration pipeline

1. **Scan** — discover source files, detect the source language, build a work plan.
2. **Frontend** — parse each source file → AST → lower to the shared IR.
3. **IR passes** — normalize, resolve, annotate unsupported constructs.
4. **Backend** — emit target-language source from the IR.
5. **Report** — per-file diffs, unsupported-construct flags, success/fail summary
   the UI renders for review before anything is written.

## TUI shape (gala_tui)

Elm triple: a single `Model`, a pure `Update(Model, Msg) -> (Model, Cmd)`, a pure
`View(Model) -> Widget`. Long-running engine work runs as `Cmd[T]` (FutureCmd /
Async) and streams progress back as messages — the Update loop never blocks.
Screens are routed with `ScreenStack`; review uses the diff/datatable widgets.
