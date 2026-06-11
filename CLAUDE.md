# gala-assimilator — Claude Code instructions

A TUI tool, written in **GALA** (transpiles to Go) and built on the
[`gala_tui`](https://github.com/martianoff/gala-tui) Elm-architecture framework,
that migrates a project/library from one language to another by **orchestrating
AI agents** through a translate → verify → fix loop (0 % → 100 %). The first
supported pair is **Go → GALA**; the architecture is built so a new language
frontend/backend — or a new agent backend — plugs in behind stable ports.

## Must read

- **[docs/architecture.md](docs/architecture.md)** — component map + the layering
  invariant. The migration engine is transport-neutral and sits behind a port
  (`EnginePort`); the TUI consumes that port. Engine internals (the agent-run
  contract, the Go frontend, the verify gate, run state) MUST NOT leak into the
  `app/ui` layer, and the UI's widget/model types MUST NOT leak into the engine.
  In particular the `gala-acp` run-contract types never cross the port. Skim it
  before editing across that seam.

## How this project is built and tested

- Language: **GALA** (`.gala` sources transpile to `.gen.go`). Functional,
  Elm-architecture, immutable-by-default. Model / Update / View triple via
  `gala_tui`'s `Program[M, T]`, `Cmd[T]`, `Sub[T]`.
- Build:  `gala build -o gala_assimilator ./cmd/gala_assimilator`
- Test:   `gala test`  (runs the whole workspace; there is no working `-run` filter)
- Deps are resolved by the GALA dep system via `gala.mod` (`gala_tui`, `uuid`,
  etc.) — no bazel required for the inner build/test loop.

## GALA conventions

- **Functional over imperative.** Sealed types + exhaustive matching; pure data;
  no hidden mutation. Side effects are `Cmd[T]` values, never inline IO.
- **`Copy(...)` re-invokes the receiver per unmodified field.** `f().Copy(x = …)`
  re-runs `f()` for every field it doesn't change (~80× on a big model). Bind the
  receiver to a `val` first, then `.Copy(...)`.
- **No `val _ = expr`.** GALA accepts bare expression statements; don't wrap a
  discarded call in `val _ =`.
- **Internal tests** (`package <name>` matching the lib package) are the standard.
  External `package main` test files next to library sources are invalid Go layout
  and `gala test` rejects them.

## Migration correctness — the product invariant

This tool's whole value is that a migration **preserves semantics**. A
translation that compiles but silently changes behavior is the worst possible
output — worse than a visible failure.

- Never emit a translation you can't justify as semantics-preserving. When a
  source construct has no faithful target, surface it (a flagged TODO / an
  unsupported-construct report), don't guess.
- Every supported construct needs a fixture that proves the round-trip /
  reference output. Correctness changes ship with the fixture that demonstrates
  them.

## NEVER commit private artifacts

This repo is (or will be) public. Readers don't share our home directory, git
history, or local debug logs. These MUST NOT appear anywhere in the source tree
(code, comments, tests, docs, PR bodies, commit-message bodies):

- Absolute home paths (`C:\Users\<name>\…`, `/home/<name>/…`) → use `/workdir/…`,
  `<repo>/…`, `os.TempDir()`, `$HOME`.
- Personal usernames / GitHub handles (except in published repo URLs) → drop or `<user>`.
- PR numbers / commit SHAs / dated "audit caught…" provenance in comments → drop;
  describe the bug shape directly. Git blame links to the PR if anyone needs it.
- Real session IDs / issue IDs → placeholders.
- Live team-member names in comments/PR text → use the role (`the TL`, `an engineer`,
  `the QA member`). Test fixture names are fine in test code.

Read every comment back as a stranger: does every path/number/date/name resolve?
If not, drop or rewrite.
