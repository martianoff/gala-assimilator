# Testing the TUI

This is how we test the `app/ui` Elm app — both the rendered layout and real
input (keyboard + mouse). Every example below is lifted from a test that lives in
the tree; no API here is aspirational. The seam rules from
[architecture.md](architecture.md) still hold in tests: the UI drives the engine
only through `EnginePort`, and tests assert on the `Model` / `View`, never on
engine internals.

## Two test modes

There are two complementary ways to exercise the UI, and most screens use both.

| Mode | Entry point | What it proves |
|------|-------------|----------------|
| **Render / snapshot** | `Snapshot(view(model), cols, rows)` → `string` | The `View` lays out correctly at a given terminal size. |
| **Interaction** | `harness.NewHarnessFull[Model, Msg](...)` | Real key/mouse input flows through `inputToMsg` → `Update` and moves the `Model`. |

Both are plain `gala test` tests — no terminal, no goroutines, fully
deterministic.

### Conventions (from CLAUDE.md)

- **Internal tests only.** Every UI test file is `package ui` (same package as the
  code under test). External `package main` test files next to library sources are
  invalid Go layout and `gala test` rejects them.
- **`gala test` runs the whole workspace.** There is no working `-run` filter, so
  don't write tests that assume one. Keep each test self-contained and fast.
- **Tests never touch the live backend.** UI tests build their `Program` with a
  factory that always returns the offline `MockEngine` (see `mockFactory` below),
  so a run is deterministic regardless of which backend the model has selected.

## Render / snapshot tests

`Snapshot(view(model), cols, rows)` renders the `View` to a fixed-size string
buffer. Wrap it with `str.S(...)` to get the string helpers (`.Contains`,
`.Lines`, `.IndexOf`), then assert on the visible text. Running the same model at
a wide and a narrow width is how we pin the responsive layout.

From `s3_test.gala` — the same mid-run model rendered at 110 and at 72 columns:

```gala
func TestS3WideRendersSplitBoard(t T) T {
    val m = midRunModel(110, 24, 5)
    val out = str.S(Snapshot(view(m), 110, 24))
    val t1 = IsTrue(t,  out.Contains("Assimilation in progress"))
    val t2 = IsTrue(t1, out.Contains("Units"))      // left pane title
    val t3 = IsTrue(t2, out.Contains("Activity"))   // right pane title
    return IsTrue(t3, out.Contains("core/value"))   // a unit row in the tree
}

func TestS3NarrowStacksBoard(t T) T {
    val m = midRunModel(72, 24, 5)
    val out = str.S(Snapshot(view(m), 72, 24))
    // ... same panes, stacked instead of split
    return IsTrue(t, out.Contains("core/"))
}
```

`midRunModel` (in the same file) shows the idiom for getting a model into a
specific state for a snapshot: build a real Mock run, then advance it with
`StepAll` before rendering (see [Driving a full run](#driving-a-full-run-scanplanruntick)
below).

S4 snapshots assert the diff body, the rationale, the verify badge, and the action
bar all render — at two widths (`s4_test.gala`, `TestS4WideRendersDiffRationaleVerify`
and `TestS4NarrowRendersStacked`).

## Interaction tests

An interaction test builds a full, mouse-aware harness around the program:

```gala
func s1Harness() harness.Harness[Model, Msg] {
    val program = NewProgram(mockFactory(), 110, 20)
    return harness.NewHarnessFull[Model, Msg](program, (e) => inputToMsg(e), 110, 20)
}
```

`NewHarnessFull` takes the `Program`, the input adapter `inputToMsg` (the same one
the real app uses to turn an input event into a `Msg`), and the terminal size. It
returns a `harness.Harness[Model, Msg]`; calling `.Start()` returns a
`harness.Session[Model, Msg]` you then drive. Every drive method returns a new
`Session`, so calls chain.

### The interaction API

All of these appear in the test sources cited:

| Call | Effect | Seen in |
|------|--------|---------|
| `.Start()` | Initialize the program, return the first `Session`. | `s1_interaction_test.gala` |
| `.Press(PlainKey(...))` | Inject a keystroke. Keys: `Tab()`, `ArrowRight()`, `Enter()`, `Char('y')`. | `s1_interaction_test.gala`, `s5_test.gala` |
| `.Click(x, y)` | Inject a mouse click at a cell. | `s1_interaction_test.gala` |
| `.FindRow(substr)` | `Option[int]` — the row index whose rendered text contains `substr`. | `s1_interaction_test.gala` |
| `.SendMsg(msg)` | Inject a `Msg` directly (async results, ticks). | `s3_test.gala`, `s5_test.gala` |
| `.Model` | Read the current `Model` for assertions. | all interaction tests |
| `.Text()` | The current rendered frame as a `string`. | `s1_interaction_test.gala` |
| `.Contains(substr)` | Convenience: does the current frame contain `substr`? | `s5_test.gala` |

#### Keyboard

`s1_interaction_test.gala`, `TestKeyboardCyclesTargetLanguage` — Tab to the
target-language cycler, then `ArrowRight` to cycle it, then assert both the model
field and the rendered view:

```gala
val s = s1Harness().Start()
    .Press(PlainKey(Tab()))         // browse → target
    .Press(PlainKey(ArrowRight()))  // cycle target forward
val t1 = Eq(t, s.Model.TargetLang(), "TypeScript")
val out = str.S(Snapshot(view(s.Model), 110, 20))
return IsTrue(t1, out.Contains("‹ TypeScript ▾ ›"))
```

Character input is `Char(...)`. In `s5_test.gala` an answer is typed one key at a
time and submitted with `Enter`:

```gala
val s2 = s
    .SendMsg(MOpenBlocker())
    .Press(PlainKey(Char('y')))
    .Press(PlainKey(Char('e')))
    .Press(PlainKey(Char('s')))
    .Press(PlainKey(Enter()))
```

#### Mouse: the `FindRow` + `IndexOf` → `Click` pattern

A click needs a cell `(x, y)`. We don't hard-code coordinates — they'd break the
moment layout shifts. Instead we locate the target by its rendered text:

1. `.FindRow(substr)` → the **row** `y` containing the target (an `Option[int]`).
2. `.Text()` → grab that row's string and `.IndexOf(substr)` → the **column** `x`.
3. `.Click(x + 1, y)` — the `+1` lands the click inside the control rather than on
   its leading glyph/edge.

The real S1 "skip" example (`s1_interaction_test.gala`,
`TestMouseSelectsSkipPolicy`) — selecting the Skip policy with the mouse, no
keyboard touched:

```gala
val s0 = s1Harness().Start()
val rowOpt = s0.FindRow("skip")
return rowOpt match {
    case Some(y) => {
        val rowText = str.S(s0.Text()).Lines().Get(y)
        val colOpt = rowText.IndexOf("skip")
        return colOpt match {
            case Some(x) => {
                val s1 = s0.Click(x + 1, y)
                val t1 = IsTrue(pre, UnsupportedEq(s1.Model.Unsupported, SkipUnit()))
                val out = str.S(Snapshot(view(s1.Model), 110, 20))
                return IsTrue(t1, out.Contains("(•) skip"))   // view marks it selected
            }
            case None() => IsTrue(pre, false)
        }
    }
    case None() => IsTrue(pre, false)
}
```

Note the assertion checks **both** the model (`Unsupported` is now `SkipUnit()`)
and the view (the radio renders `(•) skip`) — a click is only proven correct when
it moves the state *and* the state is reflected back to the user. The same shape
drives `TestMouseClickScanStartsScan`, which finds `"scan project"`, clicks it,
and asserts `s1.Model.Scanning`.

`None()` arms are deliberate failures: if the text isn't on screen, the test fails
rather than silently passing.

#### Injecting messages with `SendMsg`

Async engine results and timer ticks normally arrive as `Cmd` completions. In a
test you inject them directly with `.SendMsg(...)` so the sequence stays
deterministic. `s3_test.gala`, `TestRunGrowsLogAndDoneCount` starts a run by
hand-feeding the scan/plan/run messages, then ticks it and watches the activity
log and done-count climb:

```gala
val s0 = s3Harness().Start()
    .SendMsg(MScanPlanned(Workspace = ws, Plan = plan))
    .SendMsg(MRunStarted(State = st))
val before = logLenOf(s0.Model)
val s1 = s0
    .SendMsg(MRunTick()).SendMsg(MRunTick()).SendMsg(MRunTick())
    .SendMsg(MRunTick()).SendMsg(MRunTick())
val after = logLenOf(s1.Model)
val t2 = IsTrue(t1, after > before)   // activity log grew
return IsTrue(t2, doneOf(s1.Model) > doneOf(s0.Model))
```

## Driving a full run (scan → plan → run → tick)

To drive the whole assimilation loop in a test, get a real `MockEngine` workspace,
run it through the port, then advance the run with ticks. Reducer-style tests do
this with `StepAll` (no harness, pure message sequences); interaction tests do it
with `.SendMsg` (shown above). The port calls are identical:

```gala
val port = engine.MockEngine()
val ws   = port.Scan(engine.ScanRequest(Root = "/x"))
val plan = port.Plan(ws)
val st   = port.Run(plan)
val setup = ArrayOf[Msg](
    MScanPlanned(Workspace = ws, Plan = plan),
    MRunStarted(State = st),
)
```

### The tick helper

Ticks come from `update_test.gala`:

```gala
// each StepAll-applied tick = one Advance, since the self-rescheduling
// AfterDelay future doesn't fire under StepAll.
func ticks(n int) Array[Msg] = ArrayTabulate[Msg](n, (i int) => MRunTick())
```

So one `MRunTick` = one unit of progress. `setup.AppendAll(ticks(n))` advances the
run `n` steps. The two boundary tests pin both ends of the loop:

```gala
// update_test.gala — a few ticks finish the first unit
func TestTicksIncreaseDoneCount(t T) T {
    val (m0, _) = StepAll(prog(), setup)
    val (m1, _) = StepAll(prog(), setup.AppendAll(ticks(3)))
    return IsTrue(t, doneOf(m1) > doneOf(m0))
}

// update_test.gala — enough ticks drive every unit to Done
func TestRunDrivesToCompletion(t T) T {
    val (m1, _) = StepAll(prog(), setup.AppendAll(ticks(40)))
    return m1.Run match {
        case Some(rs) => {
            val t1 = Eq(t, engine.CountByDone(rs), rs.Units.Length())  // all done
            return IsFalse(t1, runview.AnyActionable(rs))              // nothing left
        }
        case None() => IsTrue(t, false)
    }
}
```

`doneOf` and `mockFactory` are the shared helpers in `update_test.gala`:

```gala
func mockFactory() func(string) engine.EnginePort = (name string) => engine.MockEngine()
func prog() Program[Model, Msg] = NewProgram(mockFactory(), 110, 30)
func doneOf(m Model) int = m.Run match {
    case Some(rs) => engine.CountByDone(rs)
    case None()   => 0
}
```

`StepAll` starts from a `Program`'s initial model; to continue a sequence from an
existing model, `update_test.gala` provides `StepAll2(m, msgs)`, which rebuilds a
`Program` around the given model. The S4 reducer tests use it to drive
accept/skip from a mid-run model (`s4_test.gala`).

## Driving a blocker to completion (S5)

The `MockEngine` always succeeds, so blocker handling can't be tested through it.
`s5_test.gala` swaps in `engine.ScriptedBlockerEngine("Is x nullable?")` via a
local factory — every unit it translates yields a clarification, so the first unit
reaches `Blocked` through the real run machinery. The interaction test then opens
the blocker, types an answer, submits, and asserts the unit leaves `Blocked` and
the run is actionable again:

```gala
val s = h.Start()
    .SendMsg(MScanPlanned(Workspace = ws, Plan = plan))
    .SendMsg(MRunStarted(State = st))
    .SendMsg(MRunTick()).SendMsg(MRunTick()).SendMsg(MRunTick())
val t1 = IsTrue(t, blockedAt(s.Model, 0))            // unit 0 is blocked
// ... open blocker, type "yes", Enter ...
val t2 = IsFalse(t1, blockedAt(s2.Model, 0))         // no longer blocked
return IsTrue(t3, anyActionableOf(s2.Model))         // re-queued → run continues
```

Like the UI-test rule above, the scripted factory **never** returns the live
Claude backend — blocker tests stay fully offline.

## Quick reference

- **Render a layout:** `str.S(Snapshot(view(model), cols, rows)).Contains(...)`.
- **Build a harness:** `harness.NewHarnessFull[Model, Msg](NewProgram(mockFactory(), c, r), (e) => inputToMsg(e), c, r)`.
- **Keyboard:** `.Press(PlainKey(Tab() | ArrowRight() | Enter() | Char('x')))`.
- **Mouse:** `.FindRow(substr)` → `Some(y)`; row `.IndexOf(substr)` → `Some(x)`; `.Click(x + 1, y)`.
- **Inject async / ticks:** `.SendMsg(MScanPlanned(...))`, `.SendMsg(MRunStarted(...))`, `.SendMsg(MRunTick())`.
- **Advance a run in a reducer test:** `StepAll(prog(), setup.AppendAll(ticks(n)))`.
- **Assert:** `.Model` for state, `str.S(.Text())` / `.Contains(...)` for the frame.
