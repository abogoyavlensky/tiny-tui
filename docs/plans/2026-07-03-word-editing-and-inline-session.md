# Word Editing & Inline Session Implementation Plan — ✅ COMPLETED

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Status:** All 8 tasks complete. See the Implementation Summary at the end.

**Goal:** Make word-wise cursor motion and deletion work in the input widget on macOS (and everywhere else), and add `with-inline-session` so a whole multi-widget flow — including prompts nested inside `:on-action` handlers — runs inline in one raw-mode span.

**Tech Stack:** let-go (`.lg`), lgx tasks (`lgx test`, `lgx fmt`), pty verification via `script` (see `docs/pty-verification.md`).

---

## Design

### Problem 1: word jump / word delete in the input widget

macOS terminals (Terminal.app, iTerm2 "Option as Esc+") send Option+Left as
`ESC b`, Option+Right as `ESC f`, and Option+Backspace as `ESC DEL` (`0x1b
0x7f`). let-go's key tokenizer keeps only `ESC [` (CSI) and `ESC O` (SS3)
sequences together; `ESC b` is emitted as a bare `ESC` token with `b` left in
the buffer. Today Option+Left inside an input therefore **cancels the input**
(`:esc`) and then inserts a stray `b`. Other terminals (Ghostty, kitty,
Ctrl+Arrow on Linux) send CSI forms like `ESC [1;3D` — those arrive as one
token but are simply unmapped, and the input widget has no word operations at
all.

**Fix — entirely inside tiny-tui, no let-go changes:**

1. **`key/read` lookahead.** After `term/read-key` returns a bare `ESC`,
   check `term/key-pending?`. Bytes already buffered mean the terminal sent
   an Alt chord in one burst: read the next token and translate the pair via
   a new pure `parse-alt`. A human pressing Esc alone never has bytes pending
   at that instant, so plain `:esc` keeps working. `parse` and `parse-alt`
   stay pure and unit-testable; only `read` gains the tiny impure lookahead.
   If the second read returns nil (EOF race), fall back to `:esc`.

2. **New messages** (semantic keywords, mirroring `:up`/`:left` style):

   | Message | Produced by |
   |---|---|
   | `:word-left` | `ESC b`, CSI `[1;3D`, CSI `[1;5D`, Alt chord + `:left` (`ESC ESC [ D`) |
   | `:word-right` | `ESC f`, CSI `[1;3C`, CSI `[1;5C`, Alt chord + `:right` |
   | `:backspace-word` | `ESC DEL`, and `:ctrl-w` inside widgets |
   | `:delete-word` | `ESC d`, CSI `[3;3~` |

   `parse-alt` works by parsing the inner token first, then mapping through
   an alt table: `{"b" :word-left, "f" :word-right, "d" :delete-word,
   :left :word-left, :right :word-right, :backspace :backspace-word,
   :esc :esc}`. Any *other* Alt+printable becomes an inert `:alt-<char>`
   keyword — crucially never `:esc`, so a stray Option-key can't cancel a
   half-typed input. Alt + an unmapped keyword (e.g. Alt+Enter) falls back to
   the inner parse (`:enter`).

3. **Input widget word operations** (pure). Word chars = ASCII letters and
   digits, so `/foo/bar-baz` jumps per path segment, matching macOS
   text-field behavior. (Non-ASCII letters count as separators — an accepted
   v1 limitation, noted in a code comment.)
   - `:word-left` → move to previous word start: skip non-word chars
     leftward, then word chars leftward.
   - `:word-right` → move to next word end: skip non-word chars rightward,
     then word chars rightward.
   - `:backspace-word` (also `:ctrl-w`) → delete from previous word start to
     the cursor.
   - `:delete-word` → delete from the cursor to the next word end.
   All clear `:error` like the existing editing ops. The help line stays
   unchanged (short); the README documents the bindings.

4. **Filterable list bonus:** the filter query is append-only, so word
   motion doesn't apply, but `:backspace-word` drops the query's trailing
   word (trailing non-word chars plus the word before them) and refilters.

### Problem 2: inline mode for the full cycle

Every widget (`select`, `multi-select`, `input`, `confirm`) already accepts
`:inline?`, but per-widget `:inline?` has two real weaknesses:

1. Each widget enters/exits raw mode independently — visible churn (cursor
   flash, mode toggles) between chained prompts.
2. A `run` nested inside a select's `:on-action` handler would tear down the
   outer terminal state — which is why the `:returns?` exit-and-re-enter
   dance exists.

**Fix — `with-inline-session`:**

- `screen` gains a private session flag (an atom) with a public reader
  `session?`, an `erase-widget!` (write `clear-below` + `\r`, flush), and
  `with-inline-session*`/`with-inline-session` — siblings of
  `with-inline-screen*` using the same capture/cleanup/rethrow pattern
  (never try/finally; see the let-go quirk). Enter: set the flag, then
  `init-inline!` (unset the flag before rethrowing if init throws). Exit:
  unset the flag, then `shutdown-inline!`. A nested `with-inline-session`
  just runs its body (the outer session owns the terminal).
- `core/run` checks the session before anything else:
  ```
  :screen false        -> bare run-loop (testing hook, unchanged, wins first)
  (screen/session?)    -> run-loop with render-inline!, then erase-widget!
  :inline? true        -> with-inline-screen (unchanged; one-widget sugar)
  otherwise            -> with-screen (unchanged)
  ```
  Inside a session a widget skips init/shutdown, renders inline, and erases
  its own frame when its loop ends. Chained widgets replace each other in
  place; a widget called from an `:on-action` handler renders over the
  select's frame, erases itself, and the select repaints on its next loop
  pass — no `:returns?` needed (though `:returns?` keeps working).
- `core` re-exports the macro (same pattern as screen's own macros) so apps
  write `tui/with-inline-session`.
- Documented gotcha: `println` *inside* a session runs in raw mode, so `\n`
  doesn't return the carriage — print results after the session (widgets
  erase themselves anyway).

### Testing strategy

Per `docs/pty-verification.md`, in order: `lgx test` for all pure parts
(parse-alt, word ops, filter word-delete, session branch semantics via
`:screen false` flow tests); then pty runs for everything touching `key/read`,
`screen`, and the run loop — word-key bytes end-to-end, the session example's
chained and nested flows, cancel paths (q/esc/Ctrl-C), and the throwing path.

New pty key bytes: `\033b` (Option+Left), `\033f` (Option+Right), `\033\177`
(Option+Backspace), `\033d` (Alt+d), `\033[1;3D`/`\033[1;5D` (CSI variants),
`\027` (Ctrl-W).

## File Structure

- Modify: `src/tiny_tui/key.lg` — CSI word entries in `special`, `parse-alt`, lookahead in `read`.
- Modify: `src/tiny_tui/input.lg` — word motion/deletion; docstring.
- Modify: `src/tiny_tui/list.lg` — `:backspace-word` in the filter branch.
- Modify: `src/tiny_tui/screen.lg` — session flag, `session?`, `erase-widget!`, `with-inline-session*` + macro.
- Modify: `src/tiny_tui/core.lg` — session branch in `run`, re-exported `with-inline-session`, docstrings.
- Create: `examples/inline_flow.lg` — chained select → input → confirm, plus a nested prompt from `:on-action`.
- Modify: `test/tiny_tui/key_test.lg`, `test/tiny_tui/input_test.lg`, `test/tiny_tui/list_test.lg`, `test/tiny_tui/core_test.lg`.
- Modify: `README.md`, `docs/pty-verification.md`.

Remember the let-go gotchas: no top-level forward references (order
bottom-up or `declare`), control bytes via `(char 27)`/`(char 127)` only,
capture/cleanup/rethrow instead of try/finally.

## Tasks

### Task 1: Key layer — word messages and Alt-chord parsing

**Files:**
- Modify: `src/tiny_tui/key.lg`
- Test: `test/tiny_tui/key_test.lg`

- [x] **Step 1: Write failing tests**
  In `key_test.lg`, following the existing `(str ESC ...)` style:
  - `key/parse`: `(str ESC "[1;3D")` and `(str ESC "[1;5D")` → `:word-left`;
    `"[1;3C"`/`"[1;5C"` → `:word-right`; `"[3;3~"` → `:delete-word`.
  - `key/parse-alt` (new public pure fn taking the *inner* token):
    `"b"` → `:word-left`, `"f"` → `:word-right`, `"d"` → `:delete-word`,
    `(str (char 127))` → `:backspace-word`, `(str ESC "[D")` → `:word-left`,
    `(str ESC "[C")` → `:word-right`, `ESC` → `:esc`,
    `"x"` → `:alt-x`, `(str (char 13))` → `:enter` (unmapped keyword falls
    through).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — unknown `parse-alt`, CSI entries unmapped.

- [x] **Step 3: Implement**
  - Add the five CSI pairs to the `special` map's pair list.
  - Add a private `alt-special` map and `parse-alt`: parse the inner token,
    look the result up in `alt-special` (string and keyword keys), else
    `:alt-<char>` for a 1-char string, else the parsed inner value.
  - Rewrite `read`: when the first token equals `ESC` and
    `(term/key-pending?)` is true, read the next token and `parse-alt` it
    (nil second token → `:esc`); otherwise `parse` as before. Update the ns
    docstring to explain the burst-lookahead rule (same-burst ESC+key =
    Alt chord; lone ESC = escape).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -m "Add word-motion key messages and Alt-chord parsing"`
  Codex review (P2s addressed): added Ctrl+Delete `[3;5~` alongside Alt+Delete
  `[3;3~`; `read` now falls back to `:esc` when the pending byte is only a
  SIGWINCH resize wake (BEL) so Esc-during-resize still cancels.

### Task 2: Input widget — word motion and word deletion

**Files:**
- Modify: `src/tiny_tui/input.lg`
- Test: `test/tiny_tui/input_test.lg`

- [x] **Step 1: Write failing tests**
  Using states built via `input/create` + `:value` (cursor starts at end):
  - `:word-left` on `"foo bar"` cursor 7 → cursor 4; again → 0; on
    `".agents/skills"` from the end → 8 (path segments); at 0 → stays 0.
  - `:word-right` mirrors: from 0 on `"foo bar"` → 3, then 7; at end stays.
  - `:backspace-word` on `"foo bar"` cursor 7 → text `"foo "`, cursor 4;
    on `"foo bar-"` cursor 8 → text `"foo "` (trailing `-` consumed with the
    word); at 0 → no-op; clears a pending `:error`.
  - `:ctrl-w` behaves exactly like `:backspace-word`.
  - `:delete-word` on `"foo bar"` cursor 0 → text `" bar"`, cursor 0; at end
    → no-op.
  - All return a nil event.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — messages fall through the `:else` no-op branch.

- [x] **Step 3: Implement**
  Private `word-char?` (ASCII alnum via int ranges; comment the non-ASCII
  limitation), `prev-word-start`, `next-word-end` (skip separators, then
  word chars), then wire `:word-left`/`:word-right` through `move` and
  `:backspace-word`/`:ctrl-w`/`:delete-word` as region deletions clearing
  `:error`. Keep helpers above `update` (no forward references). Update the
  `update` docstring.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -m "Add word motion and word deletion to input widget"`
  Codex review: clean, no findings.

### Task 3: List filter — delete trailing query word

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [x] **Step 1: Write failing test**
  On a filterable list with query `"foo bar"`, `:backspace-word` (and
  `:ctrl-w`) → query `"foo "` and the list refiltered; with query `""` → a
  no-op that still returns nil event.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [x] **Step 3: Implement**
  In the filter branch next to `:backspace`, handle `:backspace-word` and
  `:ctrl-w`: drop trailing word chars and the separators after them (reuse
  the same word-boundary rule as input — a tiny local helper is fine, the
  query has no cursor), then `refilter`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -m "Support word delete in list filter query"`
  Codex review (P3 addressed): added `:backspace-word`/`:ctrl-w` to
  `reserved-keys` so `bindings` no longer advertises a filter action bound to
  a key the word-delete branch now consumes (guarded by a new test).

### Task 4: PTY verification of word editing

**Files:** none (verification only; fix regressions where found)

- [x] **Step 1: Verify word keys end-to-end on a pty**
  Using `examples/input_name.lg` per `docs/pty-verification.md`, assert the
  final printed result reflects the edit (strip escapes with the sed filter):
  - Option+Backspace: type a two-word value, `\033\177`, enter → second word
    gone.
  - Ctrl-W same: `\027`.
  - Option+Left + `:delete-word`: `\033b\033d` at end of `"foo bar"` then
    enter → `"foo "`.
  - CSI variant: `\033[1;5D\033d` behaves the same.
  Verified: all four printed `Hello, foo !`. Also confirmed Option+Right
  (`\033f`) at end + `:delete-word` is a no-op (`Hello, foo bar!`). Keys were
  gated behind a leading `sleep` so raw mode is active before they arrive,
  with each ESC-chord kept in one `printf` burst so the `key-pending?`
  lookahead sees both bytes.

- [x] **Step 2: Verify Esc still cancels**
  A lone `\033` (followed by no bytes for the widget to read — end the
  printf input there) must cancel the input, and `q`/Ctrl-C paths on
  `examples/inline_select.lg` must still work.
  Verified: lone ESC and type-then-ESC both printed `Cancelled.`; Alt+x
  (`\033x`) is inert (printed `Hello, hi!` — never cancels, never inserts);
  on `inline_select.lg` q / Ctrl-C / esc all printed `No scoops today.` with
  the cursor restored (`ESC[?25h` present, `ESC[?1049` count 0), and
  down+enter selected chocolate.

- [x] **Step 3: Commit any fixes**
  No regressions found — the Task 1 `read` BEL/esc guard already handles the
  Esc path, so nothing new to commit here.

### Task 5: Screen — inline session primitives

**Files:**
- Modify: `src/tiny_tui/screen.lg`

- [x] **Step 1: Implement session primitives**
  In the inline section of `screen.lg`, ordered bottom-up:
  - Private `session-active?` atom (false), public `(session?)` reader.
  - `erase-widget!` — write `clear-below` + `"\r"`, flush (the cursor rests
    at the frame's top-left, so this wipes the widget; same trick
    `shutdown-inline!` uses).
  - `with-inline-session*` — if `(session?)` already true, just call `f`
    (outer session owns the terminal). Otherwise set the flag, `init-inline!`
    (on throw: unset the flag, rethrow), run `f` with the
    capture/cleanup/rethrow pattern, unset the flag, `shutdown-inline!`,
    return/rethrow. Do **not** use try/finally (let-go quirk).
  - `with-inline-session` macro, same shape as `with-inline-screen`.
  Sanity-checked off-tty: `session?` defaults false; the macro expands to
  `(with-inline-session* (fn [] (do ...)))`.

- [x] **Step 2: Run full test suite**
  Run: `lgx test`
  Expected: PASS (no behavior change yet).

- [x] **Step 3: Format and commit**
  Run: `lgx fmt`
  `git commit -m "Add inline session primitives to screen"`
  Codex review: clean, no findings.

### Task 6: Core — session-aware run and nested-prompt flow test

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg`

- [x] **Step 1: Write failing headless flow test**
  In `core_test.lg` (`:screen false`, scripted `:read-key-fn`s): a
  `tui/select` whose `:on-action` handler itself calls `tui/input` with its
  own scripted keys and returns `{:status ...}` from the input's result —
  assert the outer select keeps looping with the new status and the nested
  input's submitted text comes through. This locks in the nested-prompt
  pattern the session enables on a real terminal.

- [x] **Step 2: Run test**
  Run: `lgx test`
  Expected: PASS already with `:screen false` (each run is bare) — this test
  guards the pattern; if it fails, the design assumption is wrong: stop and
  re-check.
  Passed immediately (nested input submitted `alpha!`, outer status folded
  in), confirming the data-flow assumption.

- [x] **Step 3: Implement session branch**
  - In `run`, between the `:screen false` and `:inline?` branches: when
    `(screen/session?)`, call `run-loop` forcing inline rendering (default
    `render-fn` to `screen/render-inline!`), then `screen/erase-widget!`
    before returning the result.
  - Re-export the macro: `with-inline-session` in `core` expanding to
    `tiny-tui.screen/with-inline-session*` (same style as screen's macros).
  - Docstrings: `run` (the four branches), and a line in `select`/`input`/
    `confirm`/`multi-select` pointing at `with-inline-session` for chained
    flows; note the raw-mode `println` gotcha in the macro's docstring.

- [x] **Step 4: Run full test suite**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -m "Add with-inline-session for chained and nested inline widgets"`
  Codex review: clean, no findings.

### Task 7: Inline flow example + pty verification of the session

**Files:**
- Create: `examples/inline_flow.lg`

- [x] **Step 1: Write the example**
  Modeled on `examples/inline_select.lg`: inside `tui/with-inline-session`,
  a select over a few items with one `:on-action` action (`r` = rename)
  whose handler calls `tui/input` (prefilled with the item) and returns
  `{:items ... :status ...}`, then on enter a `tui/confirm`; print the
  outcome *after* the session. Header comment explains what to look for
  (widgets replacing each other in place, no alternate screen).

- [x] **Step 2: PTY-verify the session**
  Per the inline checklist in `docs/pty-verification.md`:
  - Happy path (navigate, rename via nested input, enter, confirm `y`):
    `ESC[?25l`/`ESC[?25h` once each, **zero** `ESC[?1049h`, frames begin
    with `\r`, widgets erased (`ESC[J`), final println lands where the
    widgets sat.
  - Cancel paths: esc inside the nested input (outer select keeps running),
    q at the select, Ctrl-C at each stage — cursor restored every time.
  - Throwing path: a temporary variant whose handler throws must still show
    the cursor and restore cooked mode before the error surfaces.
  Verified all three. Happy path: `Applied. Selected: write docs!` with
  hide/show-cursor 1 each, alt-screen 0, `ESC[J` ×5. Cancel paths: esc in the
  nested input left the outer select running (status `Rename cancelled`) then
  `q` → `No task chosen.`; q / Ctrl-C at select and Ctrl-C at confirm all ended
  gracefully — hide=show=1, alt=0 every time. Throwing path (temp
  `inline_flow_throw.lg`, since deleted): `ESC[?25h` present, alt=0, `kaboom`
  surfaced, post-session code never ran. Note: the nested-input esc test needs
  a delay before the following key, else the burst-lookahead reads ESC+key as
  an Alt chord instead of a cancel.

- [x] **Step 3: Format and commit**
  Run: `lgx fmt`
  `git commit -m "Add inline flow example"`
  Codex review: clean (ran its own pty smoke test).

### Task 8: Documentation

**Files:**
- Modify: `README.md`
- Modify: `docs/pty-verification.md`

- [x] **Step 1: Update docs**
  - README: word-editing key bindings for the input widget (Option/Alt +
    arrows, Option+Backspace, Ctrl-W, Alt+d), and an "Inline sessions"
    subsection with a short `with-inline-session` snippet, the nested
    `:on-action` prompt pattern, and the raw-mode `println` gotcha.
  - `docs/pty-verification.md`: add the new key bytes to the table
    (`\033b`, `\033f`, `\033d`, `\033\177`, `\033[1;3D`, `\027`) and a
    session paragraph under the inline section (what a healthy session run
    shows: one hide/show cursor pair around the whole flow, per-widget
    `ESC[J` erasures, no `1049`).
  Also added the run word messages (`:word-left` etc.), an `inline_flow.lg`
  examples-list entry, and a scripting caveat (gate keys behind a `sleep`; an
  ESC with a byte right after is an Alt chord, not a cancel).

- [x] **Step 2: Final full check**
  Run: `lgx fmt && lgx test`
  Expected: clean format, all tests PASS.
  Result: clean format; 202 tests, 410 assertions, 0 failures.

- [x] **Step 3: Commit**
  `git commit -m "Document word editing keys and inline sessions"`
  Codex review (P2 fixed): the inline-session README snippet mixed
  `:item->text :name` (map items) with string handling; switched it to string
  items so it stays consistent with `examples/inline_flow.lg`.

## Out of scope / follow-ups

- Updating skl and wtr to the new tiny-tui tag (pass nothing — word editing
  just works; optionally wrap their flows in `with-inline-session` and drop
  wtr's `:returns?` dance). Separate work in those repos after a release.
- Unicode-aware word boundaries.
- Changing let-go's tokenizer to emit Alt chords as single tokens (the
  lookahead makes it unnecessary).

## Implementation Summary

All eight tasks landed; final suite **202 tests, 410 assertions, 0 failures**.
Seven commits (one per task, verification tasks folded into their code):

- `f5fb1e1` word-motion key messages + Alt-chord parsing (`key.lg`)
- `a60e338` word motion / deletion in the input widget (`input.lg`)
- `9a0b966` word delete in the list filter query (`list.lg`)
- `c65b845` inline session primitives (`screen.lg`)
- `c1284fc` `with-inline-session` + session-aware `run` (`core.lg`)
- `5785d23` `examples/inline_flow.lg`
- `c4ba5af` README + pty-verification docs

**What shipped**

1. *Word editing.* `key/read` gained an impure burst-lookahead: a bare `ESC`
   with bytes still buffered (`term/key-pending?`) is an Alt/Option chord,
   translated by the new pure `parse-alt`; a lone `ESC` stays `:esc`. New
   messages `:word-left/:word-right/:backspace-word/:delete-word` come from
   both macOS `ESC b/f/d/DEL` and CSI `[1;3/;5…` forms. The input widget moves
   and deletes by word (ASCII letter/digit boundaries, so path segments count),
   and the filterable list drops its trailing query word on
   `:backspace-word`/`:ctrl-w`.
2. *Inline sessions.* `screen` got a private `session-active?` atom with a
   `session?` reader, `erase-widget!`, and `with-inline-session*` + macro
   (capture/cleanup/rethrow, no try/finally). `core/run` now has four ordered
   branches (`:screen false` → session → `:inline?` → full screen); inside a
   session a widget renders inline and erases its own frame, so chained prompts
   and prompts nested in an `:on-action` handler share one raw-mode span.
   `core` re-exports `with-inline-session`.

**Issues found and fixed (all via the codex review checkpoint)**

- *Task 1:* mapped Ctrl+Delete `[3;5~` alongside Alt+Delete `[3;3~`, and made
  `read` fall back to `:esc` when the only pending byte is a SIGWINCH resize
  wake (BEL) — so Esc-during-resize still cancels.
- *Task 3:* added `:backspace-word`/`:ctrl-w` to the list's `reserved-keys` so
  `bindings` no longer advertises a filter action bound to a now-consumed key.
- *Task 8:* fixed an internally inconsistent README snippet (map items vs.
  string handling).

**Verification.** Beyond `lgx test`, every terminal-touching change was driven
on a real pty: word keys end-to-end (Option+Backspace / Ctrl-W / word-jump +
delete-word / CSI variants), Esc-still-cancels and the stray-Option-is-inert
property, and the full session example — happy path (one hide/show-cursor pair,
zero alternate-screen), all cancel paths (esc in the nested prompt leaves the
outer select running; q / Ctrl-C restore the cursor), and the throwing path
(cleanup runs before the error surfaces). One gotcha worth carrying forward:
scripted pty keys must be gated behind a `sleep` (raw mode first), and an `ESC`
meant as a cancel must not be immediately followed by another byte, or the
lookahead reads it as an Alt chord.
