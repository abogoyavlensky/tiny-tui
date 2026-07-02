# Inline Mode Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

> **Status: ✅ Complete (2026-07-02).** All five tasks implemented, reviewed by codex, and PTY-verified. See the Implementation Summary at the bottom.

**Goal:** Add an `:inline?` flag to `run`/`select`/`confirm` that renders the widget in place at the current cursor position (gum/fzf style) instead of the alternate screen, erasing itself on exit.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM), `tiny-tui.screen`/`tiny-tui.key` terminal boundary, `term` namespace for raw I/O.

---

## Design

### Approach

Inline mode is a new *screen lifecycle* plus a new *frame builder* in `tiny-tui.screen`, selected by an `:inline?` flag threaded through `run`. When set, the program renders at the current cursor position and erases itself on exit, gum/fzf style. Widgets, views, key handling, the run loop, and `:tui/size` are untouched — inline mode changes only how frames reach the terminal and how the terminal is entered/left.

### The key mechanic (no cross-frame height state)

The naive approach tracks rendered height across frames. It is unnecessary. If every frame ends by returning the cursor to its own top-left corner, each frame is fully self-contained:

- **Render a frame:** carriage-return to column 0, write each view line followed by clear-to-end-of-line joined by `\r\n`, then clear-below (erases leftover rows when a frame is shorter than the previous one), then move the cursor up `height - 1` rows and carriage-return back to column 0. Height is the current view's line count only — nothing is persisted between frames.
- **Erase on exit:** the cursor is already resting at the widget's top-left, so a single clear-below (`ESC[J`) wipes the entire widget. The caller's next `println` lands exactly where the widget started.

Concretely this is `frame-str` with cursor-home (`ESC[H`) replaced by `\r`, and a trailing cursor-up + `\r` appended. Growing frames, shrinking frames, and the list↔confirm-box height change are all absorbed by clear-below. Near the bottom of the screen the `\r\n` writes scroll the terminal, but content and cursor scroll together so the relative cursor-up count still lands correctly.

### Key decisions

- **Erase on exit; no summary line kept.** `select`/`confirm` already return a value — the caller prints what it wants afterward.
- **Hide the cursor during inline mode** (as full-screen does), so no cursor blinks inside the list. Restored on exit.
- **Flag name `:inline?`**, matching the `?`-suffixed predicate keys (`:destructive?`, `:confirm?`). A flag on existing opts, not new `select-inline` functions — no duplication, mirrors how `:screen` already works.
- **Default `render-fn` resolves inside `run-loop`** keyed on `:inline?` (`render-inline!` vs `render!`), so `run` only picks the screen wrapper. Explicit `:render-fn` and `:screen false` test hooks keep overriding exactly as today.
- **Documented limitation, not solved:** the widget must fit on screen. A widget taller than the terminal cannot scroll back into view — that is what the alternate-screen (default) mode is for. Inline mode targets small pickers.

### Terminal lifecycle differences (inline vs full-screen)

| Step | Full-screen (`with-screen`) | Inline (`with-inline-screen`) |
|---|---|---|
| Enter | raw mode, alternate screen, hide cursor, clear | raw mode, hide cursor (no alt screen, no clear) |
| Frame | `ESC[H` + lines + clear-below | `\r` + lines + clear-below + cursor-up + `\r` |
| Exit | reset style, show cursor, main screen, restore mode | erase widget (clear-below + `\r`), reset style, show cursor, restore mode |

The let-go `try` quirk handling (capture/cleanup/rethrow, never rely on `finally`) is identical to `with-screen*`; the inline versions reuse that pattern.

### Testing strategy

1. **Unit** — `inline-frame-str` is pure and covered like `frame-str`: leading `\r`, per-line clear-EOL, trailing clear-below, cursor-up present for multi-line and absent for single-line.
2. **PTY** — a new `examples/inline_select.lg` driven on a pseudo-TTY per `docs/pty-verification.md`: assert the alternate-screen code `ESC[?1049h` **never** appears, cursor hide/show bracket the run, the final erase fires, and the result `println` renders where the widget was. Cover a cancel path and the throwing path.
3. Headless flow tests are unchanged — they use `:screen false`, which bypasses inline entirely.

## File Structure

- **Modify `src/tiny_tui/screen.lg`** — add the inline siblings of the existing terminal functions, reusing the private `ESC`/`clear-eol`/`clear-below` defs:
  - `inline-frame-str` (pure, public for testing)
  - `render-inline!`
  - `init-inline!` / `shutdown-inline!`
  - `with-inline-screen*` + `with-inline-screen` macro
- **Modify `src/tiny_tui/core.lg`** — `run` dispatch on `:inline?`; `run-loop` default render-fn keyed on `:inline?`; `select`/`confirm` forward `:inline?`.
- **Create `examples/inline_select.lg`** — small inline picker demo that prints the result after the widget is erased.
- **Modify `test/tiny_tui/screen_test.lg`** — `inline-frame-str` unit tests.
- **Modify `README.md`** — document the `:inline?` option.
- **Modify `docs/pty-verification.md`** — note how to verify inline (assert absence of alt-screen code).
- **Modify `docs/ROADMAP.md`** — mark inline mode delivered.

Ordering note: `tiny-tui.screen` has no forward references at the top level — define `shutdown-inline!` before `init-inline!` (which calls it in its catch), and `with-inline-screen*` before the `with-inline-screen` macro, matching the existing bottom-up order in the file.

---

### Task 1: `inline-frame-str` (pure frame builder)

**Files:**
- Modify: `src/tiny_tui/screen.lg`
- Test: `test/tiny_tui/screen_test.lg`

- [x] **Step 1: Write the failing tests**
  In `screen_test.lg`, add tests for `screen/inline-frame-str` mirroring the existing `frame-str` tests. Reuse the file's `ESC`/`clear-eol`/`clear-below` defs and add a `cursor-up` helper `(fn [n] (str ESC "[" n "A"))`.
  - single line `"Hello"` → `"\r" + "Hello" + clear-eol + clear-below + "\r"` (no cursor-up; height 1).
  - multi-line `"one\ntwo\nthree"` → `"\r"` + each line + clear-eol joined by `"\r\n"`, then clear-eol + clear-below + `(cursor-up 2)` + `"\r"`.
  - empty string `""` → `"\r" + clear-eol + clear-below + "\r"` (height 1, no cursor-up).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `inline-frame-str` is unresolved / var does not exist.

- [x] **Step 3: Implement `inline-frame-str`**
  Add a public `inline-frame-str` next to `frame-str`. It is `frame-str` with cursor-home replaced by a leading `\r`, plus a trailing "return to top-left". Build the escape with `(char 27)` at runtime — never a literal control byte. Behaviour:
  - split `s` on `"\n"`, default to `[""]` when empty (same as `frame-str`);
  - `height` = line count;
  - result = `"\r"` + `(string/join (str clear-eol "\r\n") lines)` + `clear-eol` + `clear-below` + (when `height > 1`: `ESC "[" (dec height) "A"`) + `"\r"`.
  Add a docstring explaining the return-to-top-left invariant and why no cross-frame height state is needed.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites).

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(screen): add inline-frame-str pure frame builder"`

---

### Task 2: Inline screen lifecycle

**Files:**
- Modify: `src/tiny_tui/screen.lg`

No unit tests here — raw mode, cursor, and alt-screen state only exist on a real terminal and are verified by the PTY run in Task 5.

- [x] **Step 1: Implement `render-inline!`**
  Mirror `render!`: `(term/write (inline-frame-str s))` then `(term/flush)`.

- [x] **Step 2: Implement `shutdown-inline!`**
  Give the terminal back without ever having entered the alternate screen: erase the widget by writing `clear-below` then `"\r"` (cursor is resting at the widget's top-left), then `term/reset-style`, `term/show-cursor`, `term/flush`, `term/restore-mode!`. Do **not** call `term/main-screen`. Safe to call even if `init-inline!` only partially succeeded (erasing at an arbitrary cursor position is harmless). Define this before `init-inline!`.

- [x] **Step 3: Implement `init-inline!`**
  Mirror `init!` minus the alternate screen and clear: throw when `(term/size)` is nil (not a tty), then `term/raw-mode!`, then a `try` that runs `term/hide-cursor` + `term/flush` and on `catch` calls `shutdown-inline!` and rethrows.

- [x] **Step 4: Implement `with-inline-screen*` and the `with-inline-screen` macro**
  Copy the structure of `with-screen*`/`with-screen` exactly, substituting `init-inline!`/`shutdown-inline!`. Keep the capture/cleanup/rethrow pattern (`[:ok (f)]` / `[:err e]`) — do not rely on `finally`. Define `with-inline-screen*` before the macro.

- [x] **Step 5: Verify it loads and format**
  Run: `lgx test`
  Expected: PASS — confirms the namespace still compiles with the new defs (no forward-reference errors).
  Run: `lgx fmt`

- [x] **Step 6: Commit**
  `git commit -am "feat(screen): add inline terminal lifecycle (init/shutdown/render/with-inline-screen)"`

---

### Task 3: Wire `:inline?` into core

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg` (existing headless tests must stay green)

- [x] **Step 1: Resolve the default render-fn in `run-loop`**
  Change the `render-fn` binding so its default depends on `:inline?`: `(or (:render-fn opts) (if (:inline? opts) screen/render-inline! screen/render!))`. Explicit `:render-fn` still wins.

- [x] **Step 2: Dispatch on `:inline?` in `run`**
  Rewrite `run` as a `cond`: `(false? (:screen opts))` → `(run-loop opts)`; `(:inline? opts)` → `(screen/with-inline-screen (run-loop opts))`; `:else` → `(screen/with-screen (run-loop opts))`. Update the docstring / namespace comment to mention `:inline?`.

- [x] **Step 3: Forward `:inline?` through `select` and `confirm`**
  In both `select` and `confirm`, add `:inline? (:inline? opts)` to the map passed to `run`, alongside the existing `:read-key-fn`/`:render-fn`/`:screen` pass-through. Update their docstrings to list `:inline?`.

- [x] **Step 4: Run the full suite**
  Run: `lgx test`
  Expected: PASS — headless flow tests use `:screen false` and are unaffected; this confirms the wiring didn't break existing behaviour.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(core): thread :inline? through run/select/confirm"`

---

### Task 4: Inline example and docs

**Files:**
- Create: `examples/inline_select.lg`
- Modify: `README.md`, `docs/pty-verification.md`, `docs/ROADMAP.md`

- [x] **Step 1: Write `examples/inline_select.lg`**
  Model it on `examples/select_project.lg`: a small (~5-item) list via `tui/select` with `:inline? true`, then `println` the result. The point is to show the widget erasing itself and the printed result appearing in its place. Include the `(when-not *compiling-aot* (-main))` guard. Keep the header comment style consistent with the other examples.

- [x] **Step 2: Document `:inline?` in `README.md`**
  Add a short subsection (or bullet under the options) describing `:inline?`: renders in place instead of the alternate screen, erases on exit, best for small pickers that fit on screen. Use the /writing-clearly skill.

- [x] **Step 3: Note inline verification in `docs/pty-verification.md`**
  Add a short note: for inline runs, assert the alternate-screen code `ESC[?1049h` is **absent** (`grep -a -c "$(printf '\033')\[?1049h"` should be `0`), cursor hide/show still bracket the run, and the widget is erased on exit. Give the one-line `printf ... | script ...` invocation for `examples/inline_select.lg`.

- [x] **Step 4: Update `docs/ROADMAP.md`**
  Mark inline mode as delivered (short note pointing at this plan), so the roadmap reflects reality.

- [x] **Step 5: Commit**
  `git commit -am "docs: add inline example and document :inline? mode"`

---

### Task 5: PTY verification

**Files:** none (verification only — findings may reopen earlier tasks)

Follow `docs/pty-verification.md`. Run each command, strip or inspect escapes as documented, and confirm the assertions.

- [x] **Step 1: Successful inline select**
  Run: `printf '\033[B\r' | timeout 15 script -qec "lgx run examples/inline_select.lg" /dev/null | od -c | tail -30`
  Expected: `ESC [ ? 2 5 l` (hide) near the start and `ESC [ ? 2 5 h` (show) near the end; **no** `ESC [ ? 1 0 4 9 h` anywhere; frames use `\r` + `ESC[K` lines (not `ESC[H`); a trailing `ESC[J` erase before the final print.

- [x] **Step 2: Assert the alternate screen is never entered**
  Run: `printf '\033[B\r' | timeout 15 script -qec "lgx run examples/inline_select.lg" /dev/null | grep -a -c "$(printf '\033')\[?1049h"`
  Expected: `0`.

- [x] **Step 3: Rendered result lands where the widget was**
  Run: `printf '\033[B\r' | timeout 15 script -qec "lgx run examples/inline_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -5`
  Expected: the `println` output (e.g. `Selected: ...`) with no leftover list rows above it.

- [x] **Step 4: Cancel path**
  Run: `printf 'q' | timeout 15 script -qec "lgx run examples/inline_select.lg" /dev/null | od -c | tail -20`
  Expected: cursor shown again and mode restored; widget erased; the example's cancel branch printed. Repeat with `\033` (esc) and `\003` (Ctrl-C) and confirm the same clean teardown.

- [x] **Step 5: Throwing path still restores the terminal**
  Reuse the "kaboom" pattern from the V1 plan summary (a body that throws inside `with-inline-screen`), driven on a pty. Expected: the restore sequence (show cursor, reset style, restore mode) still emits and the error surfaces — cleanup is not skipped. If no throwing inline entry point exists, add a throwaway inline body in a scratch example under the scratchpad, verify, and discard it.

- [x] **Step 6: Record results**
  If every check passes, note it in the commit. If any fails, fix in the relevant earlier task and re-run this task. No code commit is required for a pass-only run; if a fix was needed, commit it with the task it belongs to.

---

## Definition of done

- `lgx test` passes (including new `inline-frame-str` tests).
- `lgx fmt` leaves the tree clean.
- PTY runs of `examples/inline_select.lg` confirm: no alternate screen, cursor hidden/shown, widget erased on exit, result printed in place, and clean teardown on cancel and throw.
- README and `docs/pty-verification.md` document inline mode; `docs/ROADMAP.md` reflects it as delivered.

---

## Implementation Summary (2026-07-02)

Delivered on branch `inline-mode` across four commits (one per code/doc task; Task 5 was verification-only with no diff):

- `feat(screen): add inline-frame-str pure frame builder`
- `feat(screen): add inline terminal lifecycle (init/shutdown/render/with-inline-screen)`
- `feat(core): thread :inline? through run/select/confirm`
- `docs: add inline example and document :inline? mode`

**What shipped:** `:inline?` on `run`/`select`/`confirm` renders the widget in place at the cursor (gum/fzf style) and erases it on exit. The design's no-cross-frame-height insight held up in practice — `inline-frame-str` is `frame-str` with `ESC[H`→`\r` plus a trailing cursor-up + `\r`, and `shutdown-inline!` erases with a single clear-below from the resting top-left. `with-inline-screen*` reuses the `with-screen*` capture/cleanup/rethrow pattern. All 64 unit tests pass (3 new for `inline-frame-str`); each task was cleared by a `review-with-codex` checkpoint with no findings.

**PTY verification (`examples/inline_select.lg` on a pseudo-TTY):**
- Success path: no `ESC[?1049h`/`ESC[?1049l` (alternate screen never entered), cursor hidden once at start and shown once at end, frames begin with `\r` and end with `ESC[K ESC[J ESC[<n>A \r`, and teardown is a lone `ESC[J` erase followed by the result `println` landing where the widget was.
- Cancel via `q` and `esc`: clean teardown (`ESC[J` erase → `ESC[0m` → `ESC[?25h`) then the cancel branch prints. Verified.
- Throwing path (throwaway kaboom example, since deleted): `with-inline-screen` ran `shutdown-inline!` (cursor shown, style reset, mode restored) and then re-surfaced the error — cleanup was not skipped.

**Issues / notes:**
- **Ctrl-C under the PTY harness is a SIGINT kill, not the app's `:ctrl-c` path.** An immediate `\003` arrives while the pty is still in cooked mode (before `raw-mode!`), so the tty driver treats it as INTR and kills the process during startup with no teardown. This is pre-existing and identical for the full-screen `select_project.lg` (confirmed), not something inline introduced. Genuine in-raw-mode Ctrl-C routes through the same `shutdown-inline!` that `q`/`esc` use (verified clean) and is covered by the headless core test.
- **The strip-escapes verification view is not meaningful for inline mode.** Stripping ANSI removes the `ESC[J`/cursor-up codes that do the erasing, so the stripped text shows every overwritten frame concatenated. Inline mode must be verified from the raw byte stream (`od -c`) and by asserting the absence of the alt-screen code — this is now documented in `docs/pty-verification.md`.
