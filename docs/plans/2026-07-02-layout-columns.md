# layout/columns Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `layout/columns` — given rows of cells, pad each column to its per-column max visible width — so tabular lists align without every caller hand-padding in `:item->text`.

**Tech Stack:** let-go (Clojure-dialect Lisp on a Go VM); pure string layout helpers in `tiny-tui.layout`.

---

## Design

### Why / where it's used

Almost every real `select`/`multi-select` list is tabular (a name plus one or two trailing fields), and today the caller aligns columns by hand inside `:item->text` — scanning all items for the widest cell, then right-padding. That is easy to get wrong, especially with styled text, where a naive `count`/pad counts invisible ANSI escape bytes and misaligns colored cells. `columns` centralizes that once, using `style/visible-width`.

It is used by *callers* building lists (the dependency manager, a branch switcher, a process killer, file staging) — compute the aligned lines from all items, zip them onto the items, point `:item->text` at the line. It also renders static tables via `(apply layout/vstack (layout/columns rows))`. It is a layout primitive, not a table widget and not a list feature.

### Signature & behavior

```clojure
(layout/columns [["data.json" "2.5.1"]
                 ["lambdaisland/uri" "1.19.155"]])
;; => ["data.json         2.5.1"
;;     "lambdaisland/uri  1.19.155"]

(layout/columns sep rows)   ; custom separator (2-arity, mirrors hstack)
```

- Returns a **vector of aligned row strings** (one per input row).
- Each column is padded (via `layout/pad`) to the max `style/visible-width` of its cells, so styled cells align (escape codes don't count toward width).
- The **last column is not padded** — no trailing whitespace (which would smear a selected row's inverse highlight past the text).
- Separator defaults to two spaces; `(columns sep rows)` overrides it, mirroring `hstack`.
- Ragged rows tolerated (missing cells treated as `""`); `(columns [])` → `[]`.

### Algorithm

1. `(columns rows)` = `(columns "  " rows)`.
2. If `rows` is empty → `[]`.
3. `ncols` = max cell count across rows. Normalize each row to `ncols` cells with `(take ncols (concat row (repeat "")))` so ragged rows don't blow up `nth`.
4. `widths` = for each column index `c`, `(apply max 0 (map #(style/visible-width (nth % c)) norm-rows))`.
5. Each row → `(string/join sep (map-indexed (fn [c cell] (if (= c (dec ncols)) cell (pad cell (nth widths c)))) row))`.
6. Return the vector of row strings.

Uses only helpers already in the file/stdlib (`pad`, `string/join`, `style/visible-width`, `map-indexed`, `concat`, `take`, `repeat`, `apply max`).

### Key decisions

- **Lives in `tiny-tui.layout`, returns a vector of row strings** — the composable primitive; caller `vstack`s for a table or zips onto items for a list. No table widget, no `list`/`select` change.
- **Last column unpadded** — avoids trailing whitespace; correct for inverse-styled selected rows.
- **Width by `style/visible-width`** so styled cells align; **no truncation** (width-fitting stays the list widget's job).
- **Default two-space separator, optional `(columns sep rows)`** — consistent with `hstack`.
- **Left-align only in V1** — right-aligning numeric columns is an easy future add, out of scope now.

### Testing strategy

1. **Unit (`layout_test.lg`)** — even rows align to the widest cell per column; the last column has no trailing padding; a styled cell in an early column pads by visible width so the next column still aligns; custom separator; ragged rows (missing cells → `""`); `(columns [])` → `[]`; single-column input returns the cells unchanged.
2. **Example / PTY (`examples/columns_select.lg`)** — a `select` over a tabular list built with `columns`; driven on a pty to confirm the aligned rows render and a pick returns. The pure function carries most of the weight; the example is the integration check.

## File Structure

- **Modify `src/tiny_tui/layout.lg`** — add `columns` (2 arities) after `pad`/`hstack`. Pure; no forward references.
- **Modify `test/tiny_tui/layout_test.lg`** — `columns` unit tests.
- **Create `examples/columns_select.lg`** — tabular `select` demo using `columns`.
- **Modify `README.md`, `docs/ROADMAP.md`** — document and mark delivered.

---

### Task 1: `layout/columns`

**Files:**
- Modify: `src/tiny_tui/layout.lg`
- Test: `test/tiny_tui/layout_test.lg`

- [ ] **Step 1: Write failing tests**
  In `layout_test.lg`, add tests for `layout/columns`:
  - even rows: `[["a" "x"] ["abc" "y"]]` → `["a    x" "abc  y"]` (col 0 padded to 3 + two-space sep; last col unpadded).
  - last column unpadded: no row ends in a space (e.g. assert the first row equals `"a    x"`, not `"a    x "`).
  - styled cell aligns by visible width: a row whose first cell is `(style/fg :cyan "ab")` and another `"abcd"` → the second column starts at the same visible offset (assert via `style/visible-width` of the stripped prefix, or by constructing the expected string with `pad` applied to the styled cell).
  - custom separator: `(columns " | " rows)` joins with `" | "`.
  - ragged rows: `[["a"] ["a" "b"]]` → the short row's missing cell is `""` and it does not throw.
  - empty: `(columns [])` → `[]`.
  - single column: `[["a"] ["bb"]]` → `["a" "bb"]` (nothing to pad — one column is the last column).

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `layout/columns` unresolved.

- [ ] **Step 3: Implement `columns`**
  Add the 2-arity `columns` per the Algorithm, after `pad`. Docstring: aligns rows of cells to per-column max visible width; last column left unpadded; separator defaults to two spaces.

- [ ] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites).

- [ ] **Step 5: Format and commit**
  Run: `lgx fmt`
  `git commit -am "feat(layout): add columns for aligned tabular rows"`

---

### Task 2: Example and docs

**Files:**
- Create: `examples/columns_select.lg`
- Modify: `README.md`, `docs/ROADMAP.md`

- [ ] **Step 1: Write `examples/columns_select.lg`**
  A tabular `select`: a list of deps (name + version, maybe a source column), build aligned lines with `layout/columns` from all items, zip the lines onto the items (`(map (fn [d line] (assoc d :row line)) deps lines)`), and `tui/select {:items ... :item->text :row}`. Print the chosen item's name. Include the `(when-not *compiling-aot* (-main))` guard and a short header comment.

- [ ] **Step 2: Document in `README.md`**
  Add a short note near the list/layout description: `layout/columns` aligns rows of cells to per-column width (using visible width, so styled cells align) — build the aligned lines once and use them as `:item->text`. Show the zip snippet. Add `lgx run examples/columns_select.lg   # aligned tabular select` to the examples list. Use /writing-clearly.

- [ ] **Step 3: Update `docs/ROADMAP.md`**
  Mark the `layout/columns` item **Delivered** (pointing at this plan): pure helper in `tiny-tui.layout`, per-column max visible width, last column unpadded, composes with `select` by zipping lines onto items.

- [ ] **Step 4: PTY check + commit**
  Drive `examples/columns_select.lg` on a pty (per `docs/pty-verification.md`): confirm the columns visibly align in the rendered frame and a pick prints the selected name; check a cancel path. Then:
  `git commit -am "docs: add columns example and document layout/columns"`

---

## Definition of done

- `lgx test` passes, including `columns` unit tests; existing suites unchanged.
- `lgx fmt` leaves the tree clean.
- `examples/columns_select.lg` renders visibly aligned columns on a real (pty) terminal and returns a pick.
- README and `docs/ROADMAP.md` reflect the delivered `layout/columns`.
