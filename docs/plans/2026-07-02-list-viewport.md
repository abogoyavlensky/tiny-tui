# List Viewport Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

> **Status: ✅ Complete (2026-07-02).** All five tasks implemented, reviewed by codex, and PTY-verified. See the Implementation Summary at the bottom.

**Goal:** Give `tiny-tui.list` a scroll window sized to the terminal, keeping the cursor visible with a dim `"12/200"` position indicator, so long lists no longer break the layout.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM); pure widget model (`state + msg -> [state event]`, `state -> string`); terminal size injected as `:tui/size`.

---

## Design

### Approach

Teach `tiny-tui.list` to render only a **window** of items sized to the available terminal rows, keeping the cursor visible, with a dim `"12/200"` indicator. The window is a pure function of `cursor`, a persisted scroll `offset`, and a height budget. `tiny-tui.core` computes the height budget (terminal rows − chrome) and feeds it to the list the same way it already feeds `:width`. No new namespace, no API break; a list created without a height budget renders exactly as today, so every existing test stays green.

### The mechanic — sticky (fzf-style) scrolling

fzf uses a *sticky* window: the cursor moves freely inside the visible window, and the window scrolls only when the cursor would leave it. That requires remembering where the window sits, so the list state gains an `:offset` (top visible index). One pure function is the source of truth:

- **`window [l h]`** → `{:start :end :windowed? :indicator}`. Given the list and block height `h` (nil = unbounded), compute the visible slice: keep `:offset` unless the cursor scrolled out of `[offset, offset+body)`, then scroll just enough to bring it back to an edge. Return the `"(inc cursor)/n"` indicator string when windowed, else nil.
- **`reconcile-offset [l h]`** = `(assoc l :offset (:start (window l h)))` — advances and *persists* the offset so stickiness survives across frames.

`view` calls `window` to slice and append the indicator. `core` calls `reconcile-offset` after each `list/update` to persist the scroll. Both go through the same `window`, so they never diverge. The first frame (offset 0, cursor 0) and post-resize cases are self-healing because `window` clamps the offset to keep the cursor visible.

Why reconcile in `core` and not inside `list/update`: `update` is `(l msg)` and does not know the terminal height — `core` does, from `:tui/size`. Keeping `update`'s signature also keeps every existing list test untouched.

### The `window` rule (precise)

Given `n = (count items)`, `cursor`, `offset`, block height `h`:

- If `h` is nil, or `h <= 0`, or `n <= h`: not windowed — `{:start 0 :end n :windowed? false :indicator nil}`.
- Else (windowed): reserve one row for the indicator only when `h >= 2`, so `body = (if (>= h 2) (dec h) h)`. Then:
  - `max-start = (max 0 (- n body))`
  - `o = (min offset max-start)` (clamp for shrink/resize)
  - `o = (cond (< cursor o) cursor` — scrolled above the top
    `      (> cursor (+ o (dec body))) (- cursor (dec body))` — scrolled below the bottom
    `      :else o)`
  - `start = (max 0 (min o max-start))`, `end = (min n (+ start body))`
  - `indicator = (when (>= h 2) (str (inc cursor) "/" n))`

### Key decisions

- **Sticky window with a persisted `:offset`, reconciled in `core` after `update`** — not a stateless centered window. Matches fzf; cost is one state field plus one reconcile call.
- **Height budget computed in `core`** (`terminal-rows − chrome`, clamped ≥ 1); the list widget stays layout-agnostic and receives `:height` in view-opts. Default height `nil` ⇒ show everything ⇒ back-compatible.
- **Indicator = dim `"12/200"`** (`(inc cursor)/n`, `style/dim`), shown only when windowed, as a footer line *inside* the list block, reserving one row only when `height >= 2`. A height of 1 shows a single item and no indicator.
- **Viewport lives in `list.lg`** — cohesive with the widget, reusable directly.
- **No horizontal scrolling** — existing width truncation is unchanged.

### Chrome accounting (core height budget)

`select-view` in list mode is a `vstack` of `[title ""]` (when `:title`) then `[list "" help]`. So non-list rows = `(if (:title opts) 4 2)`. The indicator row is counted *inside* the list block (reserved by `window`), not in chrome. Budget: `(max 1 (- rows chrome))`, `rows = (second (:tui/size state))`. This also bounds inline mode to the screen, which is fine.

### Testing strategy

1. **Unit (`list_test.lg`)** — `window` and `view` with `:height` cover: fits (all items, no indicator, offset 0); windowed slice + indicator content; scroll down past the bottom edge advances offset with the cursor pinned to the last visible row; scroll back up retreats offset; `height nil` unchanged; `height 1` shows one item and no indicator; the indicator is dim. Existing list tests must stay green untouched.
2. **PTY (`examples/viewport_select.lg`, ~50 items)** — on the `script` pty (reports `[0 0]` → core falls back to 24 rows) confirm only a window of items renders (not all 50), the dim `k/50` indicator shows and updates while arrowing down, and the window scrolls. Include a cancel path. Per `docs/pty-verification.md`, layout/run-loop-facing changes get a real-terminal check.

## File Structure

- **Modify `src/tiny_tui/list.lg`** — `create` gains `:offset 0`; add pure `window [l h]` and `reconcile-offset [l h]`; `view` slices to the window (absolute indices preserved for the cursor/selected-style comparison) and appends the dim indicator when windowed. Empty-list and no-height paths unchanged.
- **Modify `src/tiny_tui/core.lg`** — add private `list-height [opts state]`; `select-view` passes `:height`; `select-update` calls `reconcile-offset` on the updated list.
- **Modify `test/tiny_tui/list_test.lg`** — viewport unit tests.
- **Create `examples/viewport_select.lg`** — ~50-item `tui/select` demo.
- **Modify `README.md`** — note the list scrolls with a position indicator for long lists.
- **Modify `docs/ROADMAP.md`** — mark the viewport delivered.

Ordering note: `tiny-tui.list` has no top-level forward references — define `window` before `reconcile-offset` and before `view` (both call `window`), matching the file's existing top-down/bottom-up layout.

---

### Task 1: `window` and `reconcile-offset` (pure viewport core)

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [x] **Step 1: Write failing tests for `window`**
  In `list_test.lg`, add tests against `tlist/window` using a helper list of N items (e.g. 50 numbered strings) and a `cursor`/`offset` set directly via `assoc`:
  - `h` nil → `{:start 0 :end n :windowed? false :indicator nil}`.
  - `n <= h` (e.g. 3 items, h 10) → not windowed, whole range, nil indicator.
  - windowed, cursor 0, h 10 → `start 0`, `end 9` (body 9 with indicator), `windowed? true`, `indicator "1/50"`.
  - cursor past the bottom edge (e.g. cursor 20, offset 0, h 10) → `start (- 20 8)` = 12, `end 21`, `indicator "21/50"`.
  - cursor above offset (e.g. cursor 5, offset 30, h 10) → `start 5`.
  - `h 1` → windowed, `body 1`, one item, `indicator nil`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `tlist/window` unresolved.

- [x] **Step 3: Implement `window` and `reconcile-offset`**
  Add pure `window [l h]` per the rule in the Design, returning `{:start :end :windowed? :indicator}`. Add `reconcile-offset [l h]` = `(assoc l :offset (:start (window l h)))`. Add `:offset 0` to the map returned by `create`. Docstring `window` with the sticky invariant. Define `window` before `reconcile-offset`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites, including untouched existing list tests).

- [x] **Step 5: Add `reconcile-offset` tests**
  Assert `reconcile-offset` persists `:offset` equal to `(:start (window ...))` for a windowed case and for `h` nil (offset 0). Run: `lgx test` → PASS.

- [x] **Step 6: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): add pure viewport window + offset reconcile"`

---

### Task 2: Windowed `view` with indicator

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [x] **Step 1: Write failing tests for windowed `view`**
  Add `view` tests with `:height`:
  - list of 5 items, `:height 3` → renders items `[start..end)` per `window` (cursor 0 → first 2 items + a third? recall body = h-1 = 2 when windowed) with the dim indicator line `(style/dim "1/5")` appended as the last line. Assert exact string via `string/join "\n"` including marker/`style/inverse` on the cursor row and blanks on others.
  - after moving the cursor down enough to scroll, the rendered slice shifts and the indicator updates (build the list via `reconcile-offset` to mimic what core does, since `view` reads `:offset`).
  - `:height` large enough to fit all items → no indicator, identical to today's output.
  - `:height 1` → single item row, no indicator.
  - empty list with `:height` → still the empty-text string.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `view` ignores `:height` / no indicator yet.

- [x] **Step 3: Implement windowed `view`**
  In `view`, compute `(window l (:height view-opts))`. Render only items `start..end`, keeping absolute index `i` for the cursor comparison and selected-style (iterate the subvector with an index base of `start`). When `:indicator` is present, append `(style/dim indicator)` as a trailing line. Preserve existing marker/blank/truncation logic and the empty-list path. `nil` height ⇒ `window` returns the full range ⇒ unchanged output.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): render only the visible window with a position indicator"`

---

### Task 3: Wire the height budget through `core`

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg` (existing headless tests must stay green)

- [x] **Step 1: Add `list-height` and thread `:height`**
  Add private `list-height [opts state]` = `(max 1 (- (second (:tui/size state)) (if (:title opts) 4 2)))`. In `select-view` (list mode), pass `:height (list-height opts state)` alongside `:width` to `tlist/view`. In `select-update`, after `tlist/update`, replace the updated list with `(tlist/reconcile-offset <updated-list> (list-height opts state))` before assoc-ing it back into `state` — so the persisted offset stays in sync with the terminal height. Confirm mode and the action/confirm branches keep working (offset lives in list state and survives the round-trip).

- [x] **Step 2: Run the full suite**
  Run: `lgx test`
  Expected: PASS — headless tests inject `[80 24]`, so `list-height` is large and lists still render fully; existing behavior is unchanged.

- [x] **Step 3: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(core): size the list viewport to the terminal in select"`

---

### Task 4: Example and docs

**Files:**
- Create: `examples/viewport_select.lg`
- Modify: `README.md`, `docs/ROADMAP.md`

- [x] **Step 1: Write `examples/viewport_select.lg`**
  Model on `examples/select_project.lg`: build ~50 items (e.g. `(map (fn [i] {:id i :name (str "branch-" i)}) (range 1 51))`), call `tui/select` with a `:title`, `:item->text :name`, then `println` the result. Include the `(when-not *compiling-aot* (-main))` guard and a short header comment. Keep it full-screen (the default) so the scroll window is obvious.

- [x] **Step 2: Document in `README.md`**
  Add a short note (near the list/`select` description) that long lists scroll to a window sized to the terminal, keeping the selection visible, with a dim position indicator. Use /writing-clearly.

- [x] **Step 3: Update `docs/ROADMAP.md`**
  Mark item 2 (viewport) as **Delivered**, pointing at this plan, noting the sticky-offset-in-`core` approach.

- [x] **Step 4: Add the example to the README examples list**
  Add `lgx run examples/viewport_select.lg   # long list with a scroll window` to the examples block.

- [x] **Step 5: Commit**
  `git commit -am "docs: add viewport example and document list scrolling"`

---

### Task 5: PTY verification

**Files:** none (verification only — findings may reopen earlier tasks)

Follow `docs/pty-verification.md`. The `script` pty reports `[0 0]` → core falls back to 24 rows, so a 50-item list must window.

- [x] **Step 1: Initial frame shows a window, not all 50**
  Run: `printf 'q' | timeout 15 script -qec "lgx run examples/viewport_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | grep -c "branch-"`
  Expected: fewer than 50 (roughly `24 - chrome - 1`); definitely not 50. Also visually confirm `1/50` appears (grep for `/50`).

- [x] **Step 2: Indicator present and dim**
  Run: `printf 'q' | timeout 15 script -qec "lgx run examples/viewport_select.lg" /dev/null | grep -a "$(printf '\033')\[2m"`
  Expected: at least one dim (`ESC[2m`) run, and the stripped output contains `1/50`.

- [x] **Step 3: Scrolling down moves the window and updates the indicator**
  Run (25 downs then quit): `printf '%.0s\033[B' $(seq 1 25) | { cat; printf 'q'; } | timeout 15 script -qec "lgx run examples/viewport_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | grep -oE "[0-9]+/50" | tail -3`
  Expected: the indicator's final value reflects a scrolled position (e.g. `26/50`), and earlier frames show smaller numbers — the window advanced. (If constructing the repeated-down byte string is awkward, use `printf '\033[B\033[B…q'` with 25 explicit sequences.)

- [x] **Step 4: Enter selects the item under the (scrolled) cursor**
  Drive several downs then `\r`, and assert the printed `Selected` line names the branch matching the indicator position — confirms cursor/selection stays correct while scrolled.

- [x] **Step 5: Cancel path**
  Run: `printf 'q' | timeout 15 script -qec "lgx run examples/viewport_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -2`
  Expected: the example's cancel branch prints; terminal restored (no leftover raw-mode artifacts).

- [x] **Step 6: Record results**
  If everything passes, note it. If any check fails, fix in the relevant earlier task and re-run. No commit needed for a pass-only run.

---

## Definition of done

- `lgx test` passes, including new `window`/`view` viewport tests; existing list/core tests unchanged.
- `lgx fmt` leaves the tree clean.
- A 50-item `tui/select` renders a scrolling window with a dim `k/50` indicator on a real (pty) terminal; the cursor stays visible while scrolling and selection is correct.
- README and `docs/ROADMAP.md` reflect the delivered viewport.

---

## Implementation Summary (2026-07-02)

Delivered on branch `viewport` across four commits (Task 5 was verification-only with no diff):

- `feat(list): add pure viewport window + offset reconcile`
- `feat(list): render only the visible window with a position indicator`
- `feat(core): size the list viewport to the terminal in select`
- `docs: add viewport example and document list scrolling`

**What shipped:** `tiny-tui.list/window` computes a sticky, fzf-style visible slice from `cursor`, a persisted `:offset`, and a height budget; `reconcile-offset` persists the offset. `list/view` renders only that slice (absolute indices preserved for the cursor row) and appends a dim `"cursor/total"` indicator when windowed. `core` sizes the budget to the terminal (`rows − chrome`, clamped ≥ 1), passes `:height` to the list, and reconciles the offset after each `list/update`. Widgets stay pure; a list with no height budget renders in full, so all pre-existing tests were untouched. 76 unit tests pass (12 new). Each task cleared a `review-with-codex` checkpoint.

**PTY verification (`examples/viewport_select.lg`, 50 items, on a 24-row pty):**
- First frame renders 19 rows (24 − 4 chrome − 1 indicator), not 50, with a `1/50` indicator.
- The indicator is dim (`ESC[2m1/50ESC[0m`).
- 25 down-arrows advance the indicator `1/50` → `26/50` across 26 frames — the window scrolls.
- Enter after 25 downs selects `branch-26`, matching the `26/50` position — selection stays correct while scrolled.
- `q` cancels cleanly: `Cancelled.` prints, alternate screen left once, cursor restored once.

**Issues / notes:**
- **The one substantive codex finding was the "not-yet-wired" gap after Task 2** — `select` didn't pass `:height` or reconcile offsets yet. That was Task 3 by design (the plan sequences the wiring after the rendering), and Task 3 resolved it; the final codex pass on the core wiring was clean.
- **No new gotchas.** The sticky-offset-in-`core` approach behaved exactly as designed; the reconcile-after-update call keeps the persisted offset in sync with the live terminal height, and the pty's `[0 0]`→`[80 24]` fallback made the windowing directly observable.
