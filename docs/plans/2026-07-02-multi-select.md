# Multi-Select Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

> **Status: ✅ Complete (2026-07-02).** All five tasks implemented, codex-reviewed (one extra fix folded in), and PTY-verified. See the Implementation Summary at the bottom.

**Goal:** Add `:multi? true` to the list/select — space toggles a checkbox on the current row, enter submits the checked set — for picking deps to upgrade, files to stage, tests to run.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM); pure widget model; `select` is the app-level orchestrator over the pure `list` widget.

---

## Design

### Approach

Add a `:multi?` capability to the list widget: **space toggles a checkbox** on the current row, **enter submits the checked set**. The list gains a `:selected` set and a checkbox column; nav, viewport, filtering, and actions are unchanged and compose. Like filtering, this lives almost entirely in `tiny-tui.list` — `select` already passes opts through to `tlist/create` and renders `(tlist/bindings …)`, so `:multi?` flows through and the help/view adapt automatically. A thin `tui/multi-select` helper unwraps the result to the chosen items.

### State & the `:selected` set

`create` gains `:multi? (boolean (:multi? opts))` and `:selected #{}`. **`:selected` holds original item indices** (stable across filtering, consistent with `:filtered`), not items. Toggling the highlighted row `disj`/`conj`s its `selected-index` (the original index). On enter, a submit event maps those indices back to items:

```
{:type :submit :items [chosen items in index order] :indices [sorted indices]}
```

let-go provides `conj`/`disj`/`contains?`/`sort` on sets (confirmed).

### Key routing (in `list/update`)

- **Space (`" "`) toggles** the current row's checkbox (no-op on an empty list). In the filterable branch it is claimed *before* the `(string? msg)` "printables type" branch — so other characters still narrow the query, but a literal space can't be typed into it.
- **Enter submits** — emits `(submit-event l)` instead of `(select-event l)` when `:multi?`.
- Everything else is unchanged. Non-`:multi?` behavior is byte-identical to today.

### Rendering (`list/view`)

When `:multi?`, prepend a checkbox column `[x] ` / `[ ] ` after the cursor marker/blank, based on `(contains? (:selected l) orig-idx)` where `orig-idx = (nth (:filtered l) pos)`. So a row reads `› [x] item` (highlighted, checked) or `  [ ] item`. Truncation's `max-w` budget subtracts the checkbox width (4) as well as the marker width. Plain ASCII, not styled (V1). Non-`:multi?` output is unchanged.

### `bindings`

When `:multi?`, the nav entries become `navigate · space toggle · enter submit` (instead of `navigate · enter select`); the action and quit/cancel tail is unchanged (so it still adapts for `:filterable?`).

### `tui/multi-select` (core)

```
(defn multi-select [opts]
  (let [result (select (assoc opts :multi? true))]
    (if (= :submit (:type result)) (:items result) nil)))
```

Returns the chosen **items vector** — `nil` on cancel, `[]` on an empty submit (distinct, like `input`'s `""` vs `nil`). `select` itself needs no logic change: a `:submit` event isn't an `:action`, so it flows straight through `select-update` to `select`'s return, and `:multi?` reaches `tlist/create` via the opts map.

### Key decisions

- **Multi-select lives in the list widget; `select` gets no logic change** — opts flow through, help/view adapt; new `tui/multi-select` helper only.
- **`:selected` holds original indices** (stable under filtering); submit event returns `:items` + `:indices`; `multi-select` returns the items vector (`nil` cancel, `[]` empty submit).
- **Space toggles / enter submits the checked set, empty allowed** — no "if none checked, pick highlighted" magic. In filterable+multi, space is the toggle (can't be typed into the query).
- **Checkbox `[x] `/`[ ] `, plain, not configurable in V1.**
- **No toggle-all / range-select / pre-seeded `:selected`** in V1.

### Testing strategy

1. **Unit (`list_test.lg`)** — toggle adds/removes the current original index; selection survives filtering (stable indices across a query change); enter emits `:submit` with items+indices in index order; empty submit → `:items []`; `view` renders the checkbox column with cursor-highlight and check state independent; `bindings` for multi (and multi+filterable); space toggles while other keys still filter.
2. **Headless (`core_test.lg`)** — `tui/multi-select` returns the chosen items vector; `[]` on empty submit; `nil` on cancel; the underlying `:submit` event shape via `select`.
3. **PTY (`examples/multi_select.lg`)** — toggle a couple of rows with space, enter, see the chosen set printed; cancel path. Per `docs/pty-verification.md` (space byte is a literal space; enter is `\r`).

## File Structure

- **Modify `src/tiny_tui/list.lg`** — `create` (`:multi?`, `:selected #{}`); private `checkbox`/marker defs; private `toggle` and `submit-event`; weave space-toggle + enter-submit into both `update` branches; checkbox column + budget in `view`; `:multi?` branch in `bindings`. No forward references (define `toggle`/`submit-event` before `update`; they can go near `select-event`).
- **Modify `src/tiny_tui/core.lg`** — add `tui/multi-select`; note `:multi?` on `select`'s docstring.
- **Modify `test/tiny_tui/list_test.lg`** — multi-select unit tests.
- **Modify `test/tiny_tui/core_test.lg`** — `tui/multi-select` headless tests.
- **Create `examples/multi_select.lg`** — multi-select demo.
- **Modify `README.md`, `docs/ROADMAP.md`** — document and mark delivered.

---

### Task 1: List multi-select state, toggle, submit

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [x] **Step 1: Write failing tests**
  In `list_test.lg`:
  - `create` with `:multi? true` → `:multi? true`, `:selected #{}`.
  - space toggles the current row: on a fresh multi list, `update` with `" "` adds index 0 to `:selected`; again removes it.
  - toggle then navigate then toggle → `:selected` holds both original indices.
  - selection is stable across filtering: multi + filterable, toggle a row, type a query that narrows, assert the toggled original index is still in `:selected`.
  - enter emits `{:type :submit :items [...] :indices [...]}` with items in index order (toggle indices 2 then 0 → `:indices [0 2]`, matching items).
  - enter with nothing selected → `{:type :submit :items [] :indices []}`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `:multi?`/toggle/submit absent.

- [x] **Step 3: Implement**
  Add `:multi?`/`:selected #{}` to `create`. Add private `toggle [l]` (`disj`/`conj` `(selected-index l)` in `:selected`; no-op when nil) and `submit-event [l]` (`(let [idxs (sort (:selected l))] {:type :submit :items (vec (map #(nth (:items l) %) idxs)) :indices (vec idxs)})`), defined before `update`. In both `update` branches: add a leading `(and (:multi? l) (= msg " ")) [(toggle l) nil]` clause (in the filterable branch, before `(string? msg)`); change the `:enter` clause to `(if (:multi? l) (submit-event l) (select-event l))`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites; existing list tests untouched).

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): multi-select state — space toggles, enter submits"`

---

### Task 2: Checkbox column in `view` and `bindings`

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [x] **Step 1: Write failing tests**
  - `view` for a multi list of `["a" "b"]` with index 0 toggled: row 0 (cursor, checked) = `(str "› " "[x] " (style/inverse "a"))`; row 1 = `(str "  " "[ ] " "b")`. Assert the exact joined string.
  - a non-cursor checked row renders `[x]` without the inverse cursor style (toggle index 1, move cursor to 0).
  - width truncation accounts for the checkbox: a multi list with `:width` small truncates the text using the reduced budget (marker 2 + checkbox 4).
  - `bindings` for `:multi?` → `[navigate, space toggle, enter submit, q quit]`; for `:multi? + :filterable?` → `[navigate, space toggle, enter submit, esc cancel]`.
  - a non-multi `view`/`bindings` snapshot is unchanged (regression guard).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — no checkbox column / multi bindings.

- [x] **Step 3: Implement**
  Add private `checked "[x] "` / `unchecked "[ ] "` defs. In `view`, when `:multi?`, compute the checkbox per row from `(contains? (:selected l) (nth fset pos))` and insert it after the marker/blank (`(str marker checkbox styled-text)` / `(str blank checkbox text)`); reduce `max-w` by the checkbox width when multi. In `bindings`, build the nav entries as `navigate · space toggle · enter submit` when `:multi?`, else the current `navigate · enter select`; keep the existing action + quit/cancel tail.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): render the multi-select checkbox column and adaptive help"`

---

### Task 3: `tui/multi-select` and headless tests

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg`

- [x] **Step 1: Write failing headless tests**
  In `core_test.lg` (scripted keys, `:screen false`):
  - `tui/multi-select` with items; toggle two rows (`" " :down " " :enter`) → returns the two chosen items in index order.
  - enter with nothing toggled → returns `[]`.
  - `:esc`/`:ctrl-c` → returns `nil`.
  - via `tui/select {:multi? true}`, `:enter` returns a `{:type :submit :items … :indices …}` event.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `tui/multi-select` unresolved.

- [x] **Step 3: Implement**
  Add `tui/multi-select` (wrap `select` with `:multi? true`, unwrap `:submit` → `:items`, else nil) with a docstring (returns the chosen items vector; `nil` on cancel, `[]` on empty submit; opts as `select` plus `:filterable?` composes). Add a `:multi?` note to `select`'s docstring. Confirm no other core change is needed (the `:submit` event flows through `select-update`).

- [x] **Step 4: Run the full suite**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(core): add tui/multi-select"`

---

### Task 4: Example and docs

**Files:**
- Create: `examples/multi_select.lg`
- Modify: `README.md`, `docs/ROADMAP.md`

- [x] **Step 1: Write `examples/multi_select.lg`**
  A `tui/multi-select` demo: a list of items (e.g. deps or files), `:title`, `:item->text`; print the chosen set (or a cancel/none message). Include the `(when-not *compiling-aot* (-main))` guard and a short header comment.

- [x] **Step 2: Document in `README.md`**
  Add a short section: `tui/multi-select` (or `:multi? true`) — space toggles a checkbox, enter submits the set; returns the chosen items vector (nil on cancel). Note it composes with `:filterable?`. Add `lgx run examples/multi_select.lg   # multi-select with checkboxes` to the examples list. Use /writing-clearly.

- [x] **Step 3: Update `docs/ROADMAP.md`**
  Mark the "Multi-select" item **Delivered** (pointing at this plan): `:multi?` on the list, `:selected` set of indices + checkbox column, `tui/multi-select` returns the items; composes with filtering/viewport.

- [x] **Step 4: Commit**
  `git commit -am "docs: add multi-select example and document :multi?"`

---

### Task 5: PTY verification

**Files:** none (verification only — findings may reopen earlier tasks)

Follow `docs/pty-verification.md`. Space is a literal space byte; enter is `\r`.

- [x] **Step 1: Toggle and submit**
  Drive `examples/multi_select.lg`: `space`, `down`, `space`, `enter` (toggle rows 0 and 1, submit). Run with the escape-stripping filter and assert the printed chosen set names both toggled items.

- [x] **Step 2: Checkbox column renders**
  With a partial drive (e.g. `space` then no terminator, or `space` + `esc`), assert `[x]` and `[ ]` appear in the stripped frame output.

- [x] **Step 3: Empty submit**
  `enter` immediately → the example prints its empty/none branch (returned `[]`).

- [x] **Step 4: Cancel path**
  `printf 'q'` (non-filterable) or `\033` — the example prints its cancel branch; terminal restored (`ESC[?25h`, `ESC[?1049l`).

- [x] **Step 5: Record results**
  Note the outcome. Fix-and-re-run on any failure. No commit needed for a pass-only run.

---

## Definition of done

- `lgx test` passes, including multi-select unit + headless tests; existing suites unchanged (non-multi path identical).
- `lgx fmt` leaves the tree clean.
- On a real (pty) terminal, space toggles a visible checkbox, enter submits the chosen set, an empty submit returns `[]`, and cancel returns cleanly.
- README and `docs/ROADMAP.md` reflect the delivered multi-select.

---

## Implementation Summary (2026-07-02)

Delivered on branch `multiselect` across four commits (Task 5 was verification-only with no diff):

- `feat(list): multi-select state — space toggles, enter submits`
- `feat(list): render the multi-select checkbox column and adaptive help`
- `feat(core): add tui/multi-select`
- `docs: add multi-select example and document :multi?`

**What shipped:** `:multi?` on the list widget. `create` adds `:multi?` and a `:selected` set of **original indices** (stable across filtering). Space toggles the highlighted row's index (`toggle`), enter emits `{:type :submit :items … :indices …}` (`submit-event`, index-ordered); both are woven into the filterable and non-filterable `update` branches (space claimed before filter-typing). `view` prepends a `[x] `/`[ ] ` checkbox column (width-budget-aware for truncation); `bindings` shows `navigate · space toggle · enter submit`. `tui/multi-select` wraps `select` with `:multi? true` and returns the chosen items vector — `nil` on cancel, `[]` on empty submit. **`select` needed no logic change** (the `:submit` event flows straight through, `:multi?` reaches `tlist/create` via opts) — the same architecture win as filtering. 154 tests pass (18 new); non-`:multi?` behavior is byte-identical.

**PTY verification (`examples/multi_select.lg`):** the frames show the checkbox toggling live (`[ ] src/core.lg` → `[x] src/core.lg`); space/down/space/enter staged both files; enter with nothing checked printed `Nothing staged.` (returned `[]`); `q` printed `Cancelled.` (nil) with the terminal restored (`ESC[?25h`, `ESC[?1049l`).

**Issues / notes:**
- **One extra codex finding, fixed in Task 1:** `set-items` (used by `:on-action`) left stale `:selected` indices pointing into the old item vector — a wrong-item or out-of-bounds `nth` risk if multi-select and `:on-action` are combined. `set-items` now resets `:selected` to `#{}` on item replacement (with a locking test). Codex's other two findings (render the checkbox, fix the help line) were Task 2's scope and cleared there.
- **Filterable + multi tradeoff (as designed):** space is the toggle, so a literal space can't be typed into the filter query. Other printables still narrow.
- **Deliberately out of V1:** toggle-all, range-select, pre-seeded `:selected`, and configurable checkbox glyphs — all easy additive follow-ups.
