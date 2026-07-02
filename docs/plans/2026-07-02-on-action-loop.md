# :on-action — Stay in the Select Loop

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

> **Status: ✅ Complete (2026-07-02).** All four tasks implemented, codex-reviewed (no findings), and PTY-verified. See the Implementation Summary at the bottom.

**Goal:** Add an `:on-action` handler to `tui/select` so an action (e.g. delete) updates the list in place with a status message and keeps browsing, instead of returning and exiting.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM); pure widget model; `select` is the app-level orchestrator over the pure `list`/`confirm` widgets.

---

## Design

### Approach

Add an optional `:on-action` handler to `tui/select`. When an **action** fires (directly, or after its confirmation is accepted), instead of returning and exiting the loop, `select` calls the caller's handler with the action event, swaps in any items it returns (keeping the cursor and filter query), shows a transient status line, and keeps looping. `:select` (enter) and `:cancel` still exit. This lives entirely in `select`'s orchestration (`select-update`/`select-view`); the `list` and `confirm` widgets stay pure, and the handler is the caller's code (it may be impure — e.g. update the caller's own atom).

### The handler contract

`:on-action (fn [action-event] -> {:items new-items :status "Removed X"})`, both keys optional:
- `:items` present → the list is rebuilt with them in place (cursor preserved, filter re-applied); absent → items unchanged.
- `:status` → a one-line message shown until the next keypress, then cleared.
- Returning `nil` is treated as `{}` (no change).

### The mechanic

- **New pure `tlist/set-items [l items]`** — replaces `:items`, re-runs the current `:query` filter over the new items (reusing `matches?`), and **clamps the cursor to the new count** (repeated deletes stay put rather than jumping to top). Offset is reconciled by the caller afterward. Non-filterable lists (identity `:filtered`) just get the cursor clamped.
- **`apply-on-action [opts state action-event]`** in `select-update`, routed to by both the direct-action branch and the confirmed-action branch:
  - handler set → run it (`(or (handler event) {})`); if the result `contains?` `:items`, `set-items` then `reconcile-offset`; set `:status` from the result; reset to `:list` mode (clear `:confirm`/`:pending-action`); return `[state' nil]` (continue).
  - no handler → return `[state' action-event]` (exit — byte-identical to today).
- **Status cleared at the top of every `select-update`** (`(assoc state :status nil)` before processing), so a status set by an action survives exactly until the next key.
- **`select-view`** renders `:status` on its own line between the list and the help bar (as-is — the caller styles the string); **`list-height` subtracts that row** (status line + its blank) when a status is present so the layout never overflows.

### Concretely, the refactor of `select-update`

```
(defn- select-update [opts state msg]
  (let [state (assoc state :status nil)]           ; clear the transient status
    (if (= :confirm (:mode state))
      (let [r (tconfirm/update (:confirm state) msg) event (second r)]
        (cond
          (nil? event) [(assoc state :confirm (first r)) nil]
          (:value event) (apply-on-action opts state
                                           (assoc (:pending-action state) :confirmed? true))
          :else [(assoc state :mode :list :confirm nil :pending-action nil) nil]))
      (let [r (tlist/update (:list state) msg) event (second r)
            state (assoc state :list (tlist/reconcile-offset (first r) (list-height opts state)))
            action (when (= :action (:type event)) (action-config opts event))]
        (cond
          (and action (needs-confirm? action))
          [(assoc state :mode :confirm :confirm (confirm-for-action action event)
                        :pending-action event) nil]
          (= :action (:type event)) (apply-on-action opts state event)
          :else [state event])))))
```

`apply-on-action` always resets `:mode`/`:confirm`/`:pending-action` (harmless on the direct path where they are already nil/`:list`).

### Key decisions

- **`:on-action` intercepts only `:type :action` events; `:select` (enter) and `:cancel` still exit.** Enter returns the picked item, q/esc returns `:cancel`. Model an in-loop primary action as an action.
- **Cursor preserved (clamped) across `set-items`**, not reset to top — the right feel for delete-and-keep-browsing. Viewport re-windows to keep it visible.
- **Status rendered as-is** (caller styles it); transient, cleared on the next key.
- **No `:quit` signal from the handler** (YAGNI) — actions always continue; the user quits with enter/esc/q.
- **`select`'s return value is unchanged** — the `:select`/`:cancel` event describing how the user left; callers track item state in their handler closure.
- **Without `:on-action`, behavior is byte-identical to today** — `apply-on-action` with no handler returns the action event and exits.

### Testing strategy

1. **Unit (`list_test.lg`)** — `set-items`: cursor preserved and clamped to the new count; filter query re-applied to the new items; empty-result handling; non-filterable path (identity filtered, cursor clamped).
2. **Headless (`core_test.lg`)** — `select {:on-action …}` with scripted keys and an atom: an action removes an item and the loop continues (does not exit); the handler receives the (confirmed) action event; items update in place; `:status` set then cleared on the next key; a later enter/esc returns normally. Also confirm the no-handler path still returns the action event.
3. **PTY (`examples/deps_manager.lg`)** — delete-in-place: press delete → item disappears, status shows, keep browsing, quit cleanly. Per `docs/pty-verification.md`.

## File Structure

- **Modify `src/tiny_tui/list.lg`** — add pure `set-items [l items]` (reuses `matches?`; define after `refilter`). No forward references.
- **Modify `src/tiny_tui/core.lg`** — `apply-on-action` helper (define before `select-update`); refactor `select-update` to route both action paths through it and clear `:status` each update; render the status line in `select-view`; subtract the status row in `list-height`; document `:on-action` on `select`.
- **Modify `test/tiny_tui/list_test.lg`** — `set-items` unit tests.
- **Modify `test/tiny_tui/core_test.lg`** — headless `:on-action` flow tests.
- **Create `examples/deps_manager.lg`** — canonical stay-in-the-loop demo.
- **Modify `examples/filter_select.lg`** — add an `:on-action` so its existing `:ctrl-d` delete works in place (today it would throw on confirm — the `case` has no `:action` clause).
- **Modify `README.md`, `docs/ROADMAP.md`** — document and mark delivered.

---

### Task 1: `tlist/set-items` (pure in-place item swap)

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [x] **Step 1: Write failing tests**
  In `list_test.lg`, test `tlist/set-items`:
  - non-filterable list at cursor 2 of 5; `set-items` to a 3-item vector → `:items` replaced, `:filtered` identity `[0 1 2]`, cursor clamped to 2.
  - cursor preserved when still in range: cursor 1 of 5, `set-items` to 4 items → cursor stays 1.
  - filterable list with an active query → `set-items` re-applies the query over the new items (assert `:filtered` holds the matching new indices) and clamps the cursor.
  - `set-items` to `[]` → `:filtered []`, cursor 0.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `tlist/set-items` unresolved.

- [x] **Step 3: Implement `set-items`**
  Add `set-items [l items]`: `(let [items (vec items) base (assoc l :items items) filtered (vec (filter #(matches? base (:query base) (nth items %)) (range (count items)))) n (count filtered) cursor (if (zero? n) 0 (min (:cursor l) (dec n)))] (assoc base :filtered filtered :cursor cursor :offset 0))`. Docstring: replaces items, re-applies the current filter, preserves/clamps the cursor. Define after `refilter`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites; existing list tests untouched).

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): add set-items for in-place item updates"`

---

### Task 2: `:on-action` in `select` (core)

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg`

- [x] **Step 1: Write failing headless tests**
  In `core_test.lg`, add `:on-action` flows (scripted keys, `:screen false`, an atom for item state and to record handler calls):
  - a non-destructive action fires `:on-action`; the handler returns `{:items fewer :status "Removed X"}`; the loop continues; a following `:enter`/`"q"` returns normally; assert the handler ran and (via a captured render frame or the atom) that items updated and the status appeared then cleared.
  - a destructive (confirmed) action: `[<key> "y"]` routes the confirmed event (with `:confirmed? true`) to `:on-action`.
  - no `:on-action`: the action still returns the action event (regression — exits as before).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [x] **Step 3: Implement**
  Add `apply-on-action` (per Design) before `select-update`; refactor `select-update` to clear `:status` at the top and route both action paths through `apply-on-action`. In `select-view`, render `:status` between the list block and the help bar (its own line + a blank) when present. In `list-height`, subtract 2 more rows when `(:status state)`. Add `:on-action` to `select`'s docstring.

- [x] **Step 4: Run the full suite**
  Run: `lgx test`
  Expected: PASS — existing select/confirm/filter tests unchanged (no handler ⇒ old behavior).

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(core): add :on-action to keep select looping with a status line"`

---

### Task 3: Examples, docs

**Files:**
- Create: `examples/deps_manager.lg`
- Modify: `examples/filter_select.lg`, `README.md`, `docs/ROADMAP.md`

- [x] **Step 1: Write `examples/deps_manager.lg`**
  A canonical stay-in-the-loop demo: a list of deps held in an atom, a `d` delete action (`:destructive? true` with a confirm message), and `:on-action` that removes the item from the atom and returns `{:items @deps :status (str "Removed " name)}`. `case` on the final result handles `:select`/`:cancel`. Include the `(when-not *compiling-aot* (-main))` guard and a short header comment.

- [x] **Step 2: Fix `examples/filter_select.lg`**
  It has a `:ctrl-d` destructive delete but no `:action`/`:on-action` handling (throws on confirm). Hold `branches` in an atom and add `:on-action` that removes the deleted branch and returns `{:items @branches :status (str "Removed " name)}`, so it demonstrates filter + delete-in-place. Keep the `case` on `:select`/`:cancel`.

- [x] **Step 3: Document in `README.md`**
  Add a short section: `:on-action` receives the (confirmed) action event and returns `{:items … :status …}` to update the list in place and keep browsing; enter/esc still exit. Note the widget stays pure — the handler is the caller's code. Add `lgx run examples/deps_manager.lg   # delete-in-place with :on-action` to the examples list. Use /writing-clearly.

- [x] **Step 4: Update `docs/ROADMAP.md`**
  Mark the "Staying in the loop" item **Delivered** (pointing at this plan): `:on-action` on `select`, in-place item updates via `tlist/set-items`, transient status line; enter/esc still exit; handler is caller code so widgets stay pure.

- [x] **Step 5: Commit**
  `git commit -am "docs: add deps-manager example and document :on-action"`

---

### Task 4: PTY verification

**Files:** none (verification only — findings may reopen earlier tasks)

Follow `docs/pty-verification.md`.

- [x] **Step 1: Delete updates the list in place + status**
  Drive `examples/deps_manager.lg`: move to an item, press the delete key, confirm (`y`), then quit (`q`). Run with the escape-stripping filter and assert: the deleted item's name no longer appears in the final frame, and the status text (e.g. `Removed <name>`) appears. Command modelled on the deps_actions example line in `docs/pty-verification.md`.

- [x] **Step 2: Loop continues (does not exit on the action)**
  After the delete+confirm, send more navigation then `q`; assert the run only ends at `q` (the program printed its `:cancel` branch), proving the action didn't exit the loop.

- [x] **Step 3: Status clears on the next key**
  Capture frames (`:render` via od or the stripped stream): the status shows in the frame right after the action and is gone after the next keypress. If asserting per-frame is awkward on the pty, verify via the headless test in Task 2 instead and note it here.

- [x] **Step 4: filter_select delete-in-place works (no throw)**
  Drive `examples/filter_select.lg`: type to narrow, `Ctrl-D`, `y`, then `q`. Assert it does not crash (no error/stack trace in output) and the status appears — confirming the latent `case` throw is fixed.

- [x] **Step 5: Cancel path**
  `printf 'q' …` — the program prints its cancel branch and the terminal is restored (`ESC[?25h`, `ESC[?1049l`).

- [x] **Step 6: Record results**
  Note the outcome. Fix-and-re-run on any failure. No commit needed for a pass-only run.

---

## Definition of done

- `lgx test` passes, including `set-items` and `:on-action` tests; existing suites unchanged (no-handler path identical to today).
- `lgx fmt` leaves the tree clean.
- On a real (pty) terminal, a delete action updates the list in place, shows a status that clears on the next key, keeps the loop alive, and quits cleanly; `filter_select` no longer throws on its `:ctrl-d` delete.
- README and `docs/ROADMAP.md` reflect the delivered `:on-action`.

---

## Implementation Summary (2026-07-02)

Delivered on branch `on-action` across three commits (Task 4 was verification-only with no diff):

- `feat(list): add set-items for in-place item updates`
- `feat(core): add :on-action to keep select looping with a status line`
- `docs: add deps-manager example and document :on-action`

**What shipped:** `:on-action` on `tui/select`. When an action fires (directly or after confirmation), `apply-on-action` runs the caller's handler, swaps in any returned `:items` via the new pure `tlist/set-items` (which re-applies the current filter query and clamps the cursor so repeated deletes stay put), reconciles the viewport, sets a transient `:status`, and returns to `:list` mode with a nil event — so the loop continues. `:status` is cleared at the top of every `select-update`, so it survives exactly until the next key. `select-view` renders the status between the list and help; `list-height` subtracts that row so the layout never overflows. Enter and cancel still exit, and — crucially — with no handler the action path is byte-identical to before (`apply-on-action` returns the action event). Both the direct-action and confirmed-action branches route through the one helper. 138 tests pass (8 new); each task cleared a `review-with-codex` checkpoint with no findings.

**PTY verification (`examples/deps_manager.lg`, letter-key delete):**
- `↓ d y q`: `Removed lambdaisland/uri` (green) shows and the run ends at `Done. Remaining: 3` — item removed in place, loop continued, quit cleanly.
- `↓ d y ↓ q`: the status renders in exactly **one** frame — confirming it clears on the next key (matches the headless test).
- `q` alone: `Done. Remaining: 4`, terminal restored (`ESC[?25h`, `ESC[?1049l`).

**Issues / notes:**
- **`examples/filter_select.lg` had a latent throw** (a user-added `:ctrl-d` destructive delete with no `:action`/`:on-action` clause — the `case` would throw on confirm). Fixed by adding an `:on-action` that deletes the branch in place. PTY-verified it no longer crashes.
- **Ctrl-D isn't deliverable under the `script` pty** (it's the terminal's VEOF/EOF character), so `filter_select`'s `:ctrl-d` delete couldn't be driven end-to-end there. The full stack (filterable + control-key action + confirm + `:on-action` delete-in-place + status) was instead verified with a *deliverable* control key (Ctrl-O) in a throwaway copy → `Deleted branch-5`. The committed `:ctrl-d` binding is correct; only the pty's VEOF handling hid it. On a real terminal in raw mode, Ctrl-D delivers normally.
- **Enter still exits with the selected item** (by design) — `:on-action` only intercepts `:action` events. To do something in-loop on a key, model it as an action.
