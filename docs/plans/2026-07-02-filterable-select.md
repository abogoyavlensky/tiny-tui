# Filterable Select Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `:filterable? true` to the list/select: typing narrows the list, arrows navigate the matches, enter selects — the fzf interaction, composed from the existing list + viewport.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM); pure widget model (`state + msg -> [state event]`, `state -> string`); terminal I/O confined to `screen`/`key`.

---

## Design

### Approach

Add filtering **inside the list widget**, gated by `:filterable? true`. When on, typing builds a query string that narrows the visible items, arrows navigate the matches, and enter selects the highlighted match. Filtering is derived state — "filter state is just another field" — so it composes with the existing viewport for free and stays snapshot-testable. Crucially it requires **no logic change in `core/select`**: `select` already passes the full opts map to `tlist/create` and renders `(help/view (tlist/bindings ...))`, so `:filterable?`/`:filter-fn` flow through and the help line adapts automatically.

### The indirection: `:filtered` = original indices

The list gains `:filterable? :query "" :filter-fn :filtered`. `:filtered` is a vector of **original item indices** currently matching the query (identity `[0..n)` when not filtering, so it never changes for non-filterable lists). Everything that reads items for display, selection, or navigation goes through it:

- `selected-item` / `selected-index` map the cursor through `:filtered`, so a select event's `:index` stays an index into the **original** `:items` (stable identity), and `:item` is the matched item.
- `move`, `window`, and `view` operate on `(count (:filtered l))` and slice `:filtered`.
- Because `create` seeds `:filtered` to identity for non-filterable lists, **every existing list test passes unchanged**.
- A query edit recomputes `:filtered` and resets `:cursor`/`:offset` to 0 (top), matching fzf.

### Matching

Default: **case-insensitive substring** on `(item->text item)` via `string/lower-case` + `string/includes?` (both confirmed available). Overridable with `:filter-fn (fn [query item] boolean)`. An empty query matches everything.

Precisely, `matches? [l query item]`:
- empty query → true;
- `:filter-fn` set → `((:filter-fn l) query item)`;
- else → `(string/includes? (string/lower-case ((:item->text l) item)) (string/lower-case query))`.

`refilter [l]` = `(assoc l :filtered (vec (filter-matching-indices l)) :cursor 0 :offset 0)`.

### Key routing when filterable (in `list/update`)

Branch `update` on `:filterable?`. When filterable:
- `(string? msg)` → append to `:query`, `refilter`, no event. (**`"q"` types; it does not quit.**)
- `:backspace` → drop the last query char, `refilter`, no event (no-op on empty query).
- `:up` → `move -1`; `:down` → `move +1`; `:enter` → select the matched item (or nil event when no matches); `:esc` → `{:type :cancel}`.
- anything else ignored.

When not filterable: the current `update` code, untouched (q/esc cancel, action keys fire).

### Rendering (`list/view`)

- `item-budget [l h]` = `(if (:filterable? l) (max 1 (dec h)) h)` — reserve one row for the query line when filterable. `view` and `reconcile-offset` both call `window` with this budget; `window` itself is unchanged (still takes the item budget and reads `(count (:filtered l))`).
- When filterable, the first rendered line is the query line: `(str "> " query (style/inverse " "))` — prompt + query + a static block caret (consistent with the input widget's caret).
- Body: when `(seq (:filtered l))`, slice `:filtered` to the window and render each match (marker / `selected-style` / truncate), comparing the **filtered position** to `:cursor`; when filterable with zero matches, a dim `"No matches"`.
- Trailing viewport indicator when windowed (position within the filtered set), as today.
- Non-filterable output is byte-for-byte unchanged (no query line; empty list still shows `:empty-text`).

### `bindings`

When `:filterable?`, return `[{:key "↑/↓" :label "navigate"} {:key "enter" :label "select"} {:key "esc" :label "cancel"}]` — no `q quit`, no action entries (their keys now type). Otherwise the current bindings.

### Key decisions

- **Filtering lives in the list widget via `:filtered` indices; `core/select` gets no logic change** (opts pass through; `bindings`/help adapt).
- **Case-insensitive substring default, `:filter-fn` to override** (predictable; fuzzy is caller opt-in).
- **Only `:esc` cancels; letter-key `:actions` are shadowed while filtering** — typing owns the keyboard. Documented limitation: `:filterable?` + letter `:actions` don't combine.
- **A query line `> query▮` with a static inverse caret** above the results, reserving one row; no matches shows a dim `No matches`.
- **Cursor/offset reset to top on every query change**; the indicator counts matches.

### Testing strategy

1. **Unit (`list_test.lg`)** — `matches?`/filter behavior, typing narrows + resets cursor, backspace widens, enter selects with the correct original `:index`, `"q"` types / `:esc` cancels, no-match state, filterable `bindings`, and the viewport still windows the filtered set. Existing tests stay green.
2. **Headless (`core_test.lg`)** — `tui/select {:filterable? true …}` with scripted keys narrows and returns the right item; a `:filter-fn` case.
3. **PTY (`examples/filter_select.lg`, ~50 items)** — type to narrow live, arrow + enter selects, `q` types (not quit), esc cancels. Per `docs/pty-verification.md`.

## File Structure

- **Modify `src/tiny_tui/list.lg`** — `create` (add `:filterable? :query :filter-fn :filtered`); private `matches?`, `refilter`, `item-budget`; update `selected-index`/`selected-item`/`move`/`select-event`/`action-event`/`update`/`bindings`/`window`/`reconcile-offset`/`view` to go through `:filtered`. Requires `string` (already required). No forward references.
- **Modify `src/tiny_tui/core.lg`** — docstring note on `select` for `:filterable?`/`:filter-fn`; no behavioral change.
- **Modify `test/tiny_tui/list_test.lg`** — filtering unit tests.
- **Modify `test/tiny_tui/core_test.lg`** — headless filterable `select` tests.
- **Create `examples/filter_select.lg`** — filterable picker demo.
- **Modify `README.md`, `docs/ROADMAP.md`** — document and mark delivered.

Ordering note: define `matches?`/`refilter` before `create` uses `refilter`, or have `create` build `:filtered` inline via the same helper — keep bottom-up with no forward references.

---

### Task 1: Filter core — `matches?`, `refilter`, and `create` fields

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [ ] **Step 1: Write failing tests**
  In `list_test.lg`, add tests for the filter primitives (require `tiny-tui.list`):
  - `create` with `:filterable? true` seeds `:query ""`, `:filtered` = all indices `[0..n)`; without it, `:filtered` still = `[0..n)`.
  - `matches?`-driven `refilter`: a helper list of named items; set `:query` then `refilter`; assert `:filtered` holds the original indices of the substring matches (case-insensitive), and `:cursor`/`:offset` reset to 0.
  - custom `:filter-fn` (e.g. prefix match) is used instead of the default.

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — new vars/fields absent.

- [ ] **Step 3: Implement**
  Add `:filterable? (boolean (:filterable? opts))`, `:query ""`, `:filter-fn (:filter-fn opts)`, and `:filtered (vec (range (count items)))` to `create`. Add private `matches? [l query item]` and `refilter [l]` (recompute `:filtered` from matching original indices; reset cursor/offset). Use `string/lower-case`/`string/includes?` for the default. Have `create` produce the identity `:filtered` (query empty) directly.

- [ ] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites, existing list tests untouched).

- [ ] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): add filter matching and refilter core"`

---

### Task 2: Route keys and selection through the filter

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [ ] **Step 1: Write failing tests**
  - typing narrows: on a filterable list, `update` with printable chars sets `:query` and shrinks `:filtered`; `"q"` types (no cancel event).
  - `:backspace` widens the query.
  - `:enter` after narrowing returns `{:type :select :item <match> :index <original-index>}` (assert the index is into the original items, not the filtered position).
  - `:esc` returns `{:type :cancel}`.
  - `:up`/`:down` move within matches and clamp.
  - no-match: after a query with zero matches, `selected-item`/`selected-index` are nil and `:enter` yields no event.
  - non-filterable list: `"q"` still cancels, action keys still fire (existing behavior intact).

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [ ] **Step 3: Implement**
  Update `selected-index`/`selected-item` to map the cursor through `:filtered` (guard on `(seq (:filtered l))`). Update `move` to clamp against `(count (:filtered l))`, and `select-event`/`action-event` to use the mapped item/index and guard on `:filtered`. Branch `update` on `:filterable?` per the Design (printables → query, backspace, up/down/enter/esc). Leave the non-filterable branch as the current code.

- [ ] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [ ] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): narrow on typing, select the matched item"`

---

### Task 3: Windowed filterable `view`, `bindings`, and budget

**Files:**
- Modify: `src/tiny_tui/list.lg`
- Test: `test/tiny_tui/list_test.lg`

- [ ] **Step 1: Write failing tests**
  - `window`/viewport operate on the filtered count: narrow a 50-item list to a handful, assert the window/indicator reflect the match count.
  - `view` with `:filterable? true`: query line `(str "> " query (style/inverse " "))` first, then the matched rows (cursor row inverted), then indicator when windowed; assert exact strings.
  - `view` no-match → query line + dim `"No matches"`.
  - `bindings` when filterable → navigate / enter select / esc cancel (no `q`, no actions).
  - a non-filterable `view` snapshot is unchanged (regression guard).

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [ ] **Step 3: Implement**
  Change `window` to read `(count (:filtered l))`. Add private `item-budget [l h]`. Have `view` and `reconcile-offset` call `window` with `(item-budget l h)`. In `view`, prepend the query line when filterable, render the filtered slice (compare filtered position to `:cursor`, render `(nth items (nth filtered pos))`), show dim `"No matches"` when empty-and-filterable, append the indicator. Make `bindings` filter-aware. Keep the non-filterable output identical.

- [ ] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [ ] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(list): render the filter query line, matches, and adaptive help"`

---

### Task 4: Wire through `select` and headless tests

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg`

- [ ] **Step 1: Write failing headless tests**
  In `core_test.lg`, add `tui/select` filterable flows (scripted keys, `:screen false`):
  - `{:filterable? true …}` + typing that narrows to one item + `:enter` → returns that item with its original `:index`.
  - a `:filter-fn` case.
  - `"q"` types (does not cancel) while filterable.

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL (or reveal a wiring gap).

- [ ] **Step 3: Implement / confirm wiring**
  `select` should already pass `:filterable?`/`:filter-fn` through `(tlist/create opts)` and render adaptive `bindings` — confirm and, if a gap exists, close it minimally. Add a docstring note to `select` listing `:filterable?` and `:filter-fn`.

- [ ] **Step 4: Run the full suite**
  Run: `lgx test`
  Expected: PASS.

- [ ] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(core): document and cover filterable select"`

---

### Task 5: Example and docs

**Files:**
- Create: `examples/filter_select.lg`
- Modify: `README.md`, `docs/ROADMAP.md`

- [ ] **Step 1: Write `examples/filter_select.lg`**
  Model on `examples/viewport_select.lg`: ~50 named items, `tui/select` with `:title`, `:item->text :name`, `:filterable? true`; `println` the result or a cancel message. Include the `(when-not *compiling-aot* (-main))` guard and a short header comment.

- [ ] **Step 2: Document in `README.md`**
  Add a short note near `select`: `:filterable? true` lets the user type to narrow the list (case-insensitive substring; `:filter-fn` to override), arrows navigate matches, enter selects. Note that while filtering, letter keys type (so letter `:actions` don't apply). Add `lgx run examples/filter_select.lg   # type-to-filter select` to the examples list. Use /writing-clearly.

- [ ] **Step 3: Update `docs/ROADMAP.md`**
  Mark item 3 (filtering in select) **Delivered**, pointing at this plan; note substring default, `:filter-fn` override, filtering lives in the list widget, composes with the viewport.

- [ ] **Step 4: Commit**
  `git commit -am "docs: add filter example and document :filterable? select"`

---

### Task 6: PTY verification

**Files:** none (verification only — findings may reopen earlier tasks)

Follow `docs/pty-verification.md`. The `script` pty falls back to 24 rows.

- [ ] **Step 1: Typing narrows the list**
  Run: `printf 'branch-1\r' | timeout 15 script -qec "lgx run examples/filter_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -3`
  Expected: the result names a `branch-1…` item (the query narrowed and enter selected the top match).

- [ ] **Step 2: Query line renders with the caret**
  Run: `printf 'bra' | timeout 15 script -qec "lgx run examples/filter_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | grep -a "> bra"`
  Expected: a `> bra` prompt line is present. (No trailing key, so it stays open until timeout — acceptable, we only inspect the frame.)

- [ ] **Step 3: `q` types instead of quitting**
  Run: `printf 'q\033' | timeout 15 script -qec "lgx run examples/filter_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | grep -a "> q"`
  Expected: a `> q` prompt line appears (the `q` was typed into the query), and the following esc cancels cleanly.

- [ ] **Step 4: No-match state**
  Run: `printf 'zzzzz' | timeout 15 script -qec "lgx run examples/filter_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | grep -a "No matches"`
  Expected: `No matches` shows.

- [ ] **Step 5: Cancel path**
  Run: `printf '\033' | timeout 15 script -qec "lgx run examples/filter_select.lg" /dev/null | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -2`
  Expected: the cancel branch prints; terminal restored (check `ESC[?25h`, `ESC[?1049l` present without stripping).

- [ ] **Step 6: Record results**
  Note the outcome. Fix-and-re-run on any failure. No commit needed for a pass-only run.

---

## Definition of done

- `lgx test` passes, including filtering unit + headless tests; existing list/core suites unchanged.
- `lgx fmt` leaves the tree clean.
- On a real (pty) terminal, `tui/select {:filterable? true}` narrows as you type, shows a `> query` line, selects the highlighted match, types `q` rather than quitting, shows `No matches`, and cancels cleanly on esc.
- README and `docs/ROADMAP.md` reflect the delivered filtering.
