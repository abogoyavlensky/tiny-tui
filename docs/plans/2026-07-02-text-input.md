# Text Input Widget Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

> **Status: ✅ Complete (2026-07-02).** All six tasks implemented, reviewed by codex, and PTY-verified. See the Implementation Summary at the bottom.

**Goal:** Add a single-line text input widget (`tiny-tui.input`) and a `tui/input` helper, so tools can prompt for a string — a name, a search query, a confirmation phrase — with an optional validator and inline error.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM); pure widget model (`state + msg -> [state event]`, `state -> string`); terminal I/O confined to `screen`/`key`.

---

## Design

### Approach

A new pure widget `tiny-tui.input` modeled directly on `tiny-tui.confirm` (state + msg → `[state event]`, state → string), plus a `tui/input` helper in `tiny-tui.core` built on `run` — the same layering `confirm` uses. It reuses the existing key vocabulary (`tiny-tui.key` already parses `:left :right :home :end :backspace :delete`, printables as one-character strings) and the merged inline mode. Single-line only; multiline editing is out of scope.

### State & messages

`create` returns `{:text :cursor :placeholder :validate :error}` (`:text` seeded from `:value`, cursor at end). `update [s msg]` returns `[new-state event]`:

- **`(string? msg)`** → insert the string at the cursor, advance the cursor by its length, clear `:error`. This single rule captures every printable. *Critical:* only `:esc` cancels — `"q"` and every other letter are inserted as text, unlike list/confirm.
- `:backspace` → delete the char before the cursor (no-op at start), cursor − 1, clear error.
- `:delete` → delete the char at the cursor (forward delete, no-op at end), clear error.
- `:left` / `:right` → move the cursor, clamped to `[0, len]`.
- `:home` / `:end` → cursor to 0 / `len`.
- `:enter` → run `:validate` on the text (when set): if it returns a non-nil error string, store it in `:error` and emit **no** event (`[state' nil]`); otherwise emit `{:type :submit :text text}`.
- `:esc` → `{:type :cancel}`.
- anything else (`:up`, `:down`, `:tab`, ctrl keys) → ignored, `[s nil]`.

`:ctrl-c` and end-of-input never reach the widget as edit keys — the run loop handles them (returns `:ctrl-c` / `nil`), which `tui/input` maps to cancel.

Edit math (with `subs`): insert at `cursor` = `(str (subs text 0 cursor) s (subs text cursor))`, `cursor += (count s)`. Backspace = `(str (subs text 0 (dec cursor)) (subs text cursor))`. Delete = `(str (subs text 0 cursor) (subs text (inc cursor)))`.

### Rendering the cursor

The global terminal cursor is hidden (`screen/init!`), so the widget draws its own **block cursor** via inverse video. `field-str`:

- Non-empty text: `before + inverse(char-at-cursor) + after`, where the "char at cursor" is `" "` (an inverse space) when the cursor is at end-of-text.
- Empty text: a leading `inverse(" ")` block followed by the dim `:placeholder` (when set).

`input/view` stacks `title` (bold, + blank) when set, then the field, then `["" (style/fg :red error)]` when `:error` is set — mirroring how `confirm/view` composes title/message/list. `core`'s `tui/input` adds the help bar below, exactly like `confirm-view` does.

### The `tui/input` helper (core)

Mirror `confirm`: build a `run` map with `:init {:input (tinput/create opts)}`, an `:update` that threads `tinput/update`, a `:view` that stacks `tinput/view` + `""` + `(help/view input-bindings)`, and forwards `:inline?` and the test hooks (`:read-key-fn :render-fn :screen`). `input-bindings` = `[{:key "enter" :label "submit"} {:key "esc" :label "cancel"}]`. Unwrap the result: `{:type :submit :text t}` → return `t`; anything else (cancel, `:ctrl-c`, nil) → return `nil`. An empty but submitted field returns `""`, distinct from `nil`.

### Key decisions

- **Validate on submit, not per keystroke.** Enter runs `:validate`; on error the field stays open with a red inline message that clears on the next edit. Simplest and correct; avoids running a possibly-expensive check on every key.
- **Only `:esc` cancels; all printables (including `"q"`) insert.** Correctness-critical difference from the other widgets.
- **`tui/input` returns the string on submit, `nil` on cancel** (mirrors `confirm` returning a bool); empty submit returns `""`.
- **Include `:delete` (forward) alongside `:backspace`** — the key already parses, so it is free.
- **Support `:inline?`** — a one-line prompt is the ideal inline use case; pass it through like `select`/`confirm`.
- **No horizontal scrolling** (documented limitation): text wider than the terminal is not windowed in V1.

### Testing strategy

1. **Unit (`input_test.lg`)** — cover editing, cursor motion, the anti-quit rule, submit/validate/cancel, and view snapshots (placeholder, mid-string block cursor, end-of-text trailing block, error line). Pure, snapshot-style like the other widget tests.
2. **Headless (`core_test.lg`)** — drive `tui/input` with `:screen false` and scripted keys: a typed string submits and is returned; esc/ctrl-c returns nil; validate blocks, then a corrected value submits.
3. **PTY (`examples/input_name.lg`)** — type a name and submit (result printed), and an esc-cancel path; confirm raw-mode entry/teardown and that the inverse block cursor renders. Per `docs/pty-verification.md`.

## File Structure

- **Create `src/tiny_tui/input.lg`** — `create`, `update`, `view` plus private `field-str`. Pure; requires `layout`, `style` (no terminal access). Order defs top-down with no forward references.
- **Modify `src/tiny_tui/core.lg`** — require `tiny-tui.input :as tinput`; add private `input-bindings` and the public `input` helper next to `confirm`.
- **Create `test/tiny_tui/input_test.lg`** — widget unit tests.
- **Modify `test/tiny_tui/core_test.lg`** — headless `tui/input` flow tests.
- **Create `examples/input_name.lg`** — `tui/input` demo.
- **Modify `README.md`** — document `tui/input`.
- **Modify `docs/ROADMAP.md`** — mark item 1 (text input) delivered.

---

### Task 1: `input` widget — editing and cursor motion

**Files:**
- Create: `src/tiny_tui/input.lg`
- Test: `test/tiny_tui/input_test.lg`

- [x] **Step 1: Write failing tests for create + editing**
  New `input_test.lg` (ns `tiny-tui.input-test`, require `tiny-tui.input :as tinput`). Test:
  - `create` with no `:value` → `:text ""`, `:cursor 0`; with `:value "hi"` → `:text "hi"`, `:cursor 2`.
  - insert: from empty, `update` with `"a"` then `"b"` → text `"ab"`, cursor 2; insert mid-string (cursor moved left first) inserts at the cursor.
  - `:backspace` at cursor 0 → unchanged; mid/end → removes the preceding char, cursor − 1.
  - `:delete` at end → unchanged; mid → removes the char at the cursor, cursor unchanged.
  - `:left`/`:right` clamp at `0`/`len`; `:home` → 0; `:end` → len.
  - **anti-quit:** `"q"` inserts a literal `q` (text `"q"`), returns no event.
  - moving/editing returns `[state nil]` (no event).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `tinput/*` unresolved.

- [x] **Step 3: Implement `create` and `update` (edit/motion only)**
  Implement `create` (seed `:text`/`:cursor`/`:placeholder`/`:validate`/`:error nil`) and `update` for the string-insert rule and `:backspace :delete :left :right :home :end`, plus the `[s nil]` fallback. Leave `:enter`/`:esc` for Task 2 (they can fall through to the ignore branch for now, but structure `update` as a `cond` ready for them).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites).

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(input): add text input widget editing and cursor motion"`

---

### Task 2: Submit, validate, cancel

**Files:**
- Modify: `src/tiny_tui/input.lg`
- Test: `test/tiny_tui/input_test.lg`

- [x] **Step 1: Write failing tests for submit/validate/cancel**
  Add tests:
  - `:enter` with no `:validate` → `{:type :submit :text "<text>"}`.
  - `:enter` with `:validate` returning a string (e.g. empty-check) → no event (`nil`), state `:error` set to that string.
  - after a blocked submit, an edit (`"x"` or `:backspace`) clears `:error`.
  - `:enter` with `:validate` that returns `nil` for the current text → submits.
  - `:esc` → `{:type :cancel}`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — enter/esc not handled yet.

- [x] **Step 3: Implement `:enter`/`:esc` and error-clearing**
  Add the `:enter` branch (run `:validate`; set `:error` + no event on failure, else `{:type :submit :text text}`) and `:esc` → `{:type :cancel}`. Ensure every edit/motion branch clears `:error` (set it to nil on text change; simplest is to clear on all edit keys).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(input): submit with optional validation and cancel"`

---

### Task 3: `view` with block cursor, placeholder, and error line

**Files:**
- Modify: `src/tiny_tui/input.lg`
- Test: `test/tiny_tui/input_test.lg`

- [x] **Step 1: Write failing tests for `view`**
  Assert exact strings via `layout/vstack` / `string/join`, using `style/inverse`, `style/dim`, `style/bold`, `style/fg`:
  - empty text, placeholder `"name"`, no title → `(str (style/inverse " ") (style/dim "name"))`.
  - text `"ab"`, cursor 1, no title → `(str "a" (style/inverse "b"))` (cursor block on `b`, nothing after).
  - text `"ab"`, cursor 2 (end) → `(str "ab" (style/inverse " "))` (trailing block).
  - with `:title "Name"` → bold title + blank line above the field.
  - with `:error "required"` → a `(style/fg :red "required")` line (preceded by a blank) below the field.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `view` missing / not composing these.

- [x] **Step 3: Implement `field-str` and `view`**
  Add private `field-str` (block-cursor logic above) and `view` (vstack of optional title, field, optional error line), matching `confirm/view`'s composition style. No terminal access.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(input): render field with a block cursor, placeholder, and error line"`

---

### Task 4: `tui/input` helper in core

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg`

- [x] **Step 1: Write failing headless tests**
  In `core_test.lg`, add `tui/input` flows using the scripted-key pattern already in the file (`:screen false`, `:read-key-fn`, `:render-fn`):
  - typing `"h" "i" :enter` → returns `"hi"`.
  - `:esc` (or `:ctrl-c`) → returns `nil`.
  - `:validate` that rejects empty: `:enter` on empty stays (no return yet), then typing `"x" :enter` → returns `"x"`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `tui/input` unresolved.

- [x] **Step 3: Implement `input` + `input-bindings`**
  Require `tiny-tui.input :as tinput` in `core`. Add private `input-bindings` and the public `input` helper (mirror `confirm`): build the `run` map, forward `:inline?` and the test hooks, and unwrap `{:type :submit :text t}` → `t`, else `nil`. Docstring lists `:title :placeholder :value :validate :inline?` and the test hooks.

- [x] **Step 4: Run the full suite**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(core): add tui/input helper"`

---

### Task 5: Example and docs

**Files:**
- Create: `examples/input_name.lg`
- Modify: `README.md`, `docs/ROADMAP.md`

- [x] **Step 1: Write `examples/input_name.lg`**
  Model on `examples/select_project.lg`: call `tui/input` with a `:title`, `:placeholder`, and a `:validate` that rejects empty (e.g. returns `"Name can't be empty"`), then `println` the result or a cancel message. Include the `(when-not *compiling-aot* (-main))` guard and a short header comment.

- [x] **Step 2: Document `tui/input` in `README.md`**
  Add a short section near `confirm`: prompt for a line of text, returns the string or nil on cancel, with `:placeholder` and `:validate`. Show the empty-check example. Add `lgx run examples/input_name.lg   # tui/input` to the examples list. Use /writing-clearly.

- [x] **Step 3: Update `docs/ROADMAP.md`**
  Mark item 1 (text input) **Delivered**, pointing at this plan; note single-line, validate-on-submit, block cursor, `:inline?` supported.

- [x] **Step 4: Commit**
  `git commit -am "docs: add input example and document tui/input"`

---

### Task 6: PTY verification

**Files:** none (verification only — findings may reopen earlier tasks)

Follow `docs/pty-verification.md`.

- [x] **Step 1: Type and submit**
  Run: `printf 'Ada\r' | timeout 15 script -qec "lgx run examples/input_name.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -3`
  Expected: the printed result contains `Ada` (the submitted text).

- [x] **Step 2: Editing works (backspace)**
  Run: `printf 'Adaa\177\r' | timeout 15 script -qec "lgx run examples/input_name.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -3`
  (`\177` = backspace) Expected: result is `Ada`, confirming the last `a` was deleted.

- [x] **Step 3: Block cursor renders**
  Run: `printf 'Ada' | timeout 15 script -qec "lgx run examples/input_name.lg" /dev/null | grep -a -c "$(printf '\033')\[7m"`
  Expected: ≥ 1 — the inverse (`ESC[7m`) block cursor is present. (No trailing `\r`, so input stays open until stdin closes.)

- [x] **Step 4: Validation blocks empty submit**
  Run: `printf '\r' | timeout 15 script -qec "lgx run examples/input_name.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r'`
  Expected: the red error text appears; then stdin closes (end-of-input) and the run ends via cancel — confirm no crash and the terminal is restored (`ESC[?25h`, `ESC[?1049l` present when run without stripping).

- [x] **Step 5: Cancel path**
  Run: `printf '\033' | timeout 15 script -qec "lgx run examples/input_name.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -2`
  Expected: the cancel branch prints; terminal restored.

- [x] **Step 6: Record results**
  Note the outcome. If any check fails, fix in the relevant earlier task and re-run. No commit needed for a pass-only run.

---

## Definition of done

- `lgx test` passes, including new `input` widget and `tui/input` headless tests; existing suites unchanged.
- `lgx fmt` leaves the tree clean.
- On a real (pty) terminal, `tui/input` accepts and edits text, renders a visible block cursor, blocks an invalid submit with a red message, submits a valid value, and cancels cleanly on esc.
- README and `docs/ROADMAP.md` reflect the delivered text input.

---

## Implementation Summary (2026-07-02)

Delivered on branch `input` across five commits (Task 6 was verification-only with no diff):

- `feat(input): add text input widget editing and cursor motion`
- `feat(input): submit with optional validation and cancel`
- `feat(input): render field with a block cursor, placeholder, and error line`
- `feat(core): add tui/input helper`
- `docs: add input example and document tui/input`

**What shipped:** `tiny-tui.input` — a pure single-line text field (`create`/`update`/`view`) modeled on `confirm` — plus the `tui/input` helper in `core`. Editing (insert / `:backspace` / `:delete`), cursor motion (`:left :right :home :end`, clamped), submit-with-`:validate` (blocks with a red inline error that clears on the next edit), and `:esc`/`:ctrl-c` cancel. The widget draws its own inverse-video block cursor since the real cursor stays hidden. `tui/input` returns the entered string on submit or `nil` on cancel, and forwards `:inline?` and the test hooks. 98 unit + headless tests pass (17 new). Each task cleared a `review-with-codex` checkpoint with no findings.

**PTY verification (`examples/input_name.lg`):**
- Type `Ada` + enter → `Hello, Ada!`.
- `Adaa` + backspace + enter → `Hello, Ada!` (edit applied).
- The inverse block cursor (`ESC[7m`) renders while typing.
- Empty submit shows the red `Name can't be empty` error (`ESC[31m`) and correctly keeps the field open; a following esc cancels cleanly (`ESC[?25h` + `ESC[?1049l` + `Cancelled.`).
- Esc alone cancels cleanly.

**Issues / notes:**
- **Caught a real bug via a failing view test:** the first `create` dropped `:title` (never stored it), so `view` rendered no title. Fixed by adding `:title (:title opts)` to `create` — the snapshot test made it obvious.
- **`str`-shadowing avoided:** the insert helper's parameter was renamed from `str` to `ins` so it doesn't shadow the `str` function it calls.
- **PTY EOF nuance:** the `script` pty does not propagate the pipe's EOF to the slave, so an input left open (blocked empty submit, or typing without a terminating key) hangs until `timeout` kills it — the documented "missed key blocks forever" gotcha. Verify blocked/edit paths by sending a terminating key (esc/enter), not by relying on stdin closing. The widget behavior (staying open after a rejected submit) is correct.
- **Known limitation (documented):** no horizontal scrolling for text wider than the terminal in V1.
