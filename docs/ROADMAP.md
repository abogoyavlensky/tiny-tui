Good moment for this — V1 proves the model works; the question is what turns it from a demo into something you'd reach for weekly. Here's my read, organized by what actually unlocks real tools rather than by what's fun to build.

The three gaps that block real apps today

1. Text input. This is the biggest one. Almost every practical flow eventually needs a string from the user: a search query, a new name, a commit message, "type the project name to confirm deletion". A single-line input widget fits the existing model perfectly — state is {:text "" :cursor 0}, messages are printables/:backspace/:left/:right/:home/:end, enter emits {:type :submit :text ...} — plus a tui/input {:title ... :placeholder ... :validate ...} helper. Deliberately single-line; multiline editing is legmacs's job. A :validate fn with an inline error line covers the common "can't be empty / must be unique" cases cheaply. **Delivered** (`docs/plans/2026-07-02-text-input.md`): `tiny-tui.input` + `tui/input`, single-line, validate-on-submit with an inline red error, a self-drawn inverse block cursor (the real cursor stays hidden), and `:inline?` supported. Only :esc cancels — every printable including "q" is text. No horizontal scrolling yet.

2. A viewport for the list. Right now the list renders every item, so 200 git branches on a 40-row terminal breaks the layout. A scroll window that keeps the cursor visible (plus a dim "12/200" indicator) is table stakes for real data. I'd do a viewport, not pagination — it's what people expect from fzf/fuzzy finders, and it's simpler to reason about. :tui/size is already injected, so the plumbing exists. **Delivered** (`docs/plans/2026-07-02-list-viewport.md`): `tiny-tui.list/window` computes a sticky fzf-style slice, `core` sizes it to the terminal (rows − chrome) and persists the scroll offset via `reconcile-offset`. Widgets stay pure; a list with no height budget still renders in full.

3. Filtering in select. Once input and viewport exist, this is the killer combination: :filterable? true, typing narrows the list, enter selects. fzf built an entire ecosystem on just this interaction. For lists over ~10 items it's the difference between a toy and a daily driver. It's also the feature that most rewards the pure-widget architecture: filter state is just another field, and the whole thing stays snapshot-testable. **Delivered** (`docs/plans/2026-07-02-filterable-select.md`): filtering lives entirely in the list widget via a `:filtered` vector of original indices (so `core/select` needed no logic change — opts pass through, help adapts); case-insensitive substring by default with a `:filter-fn` override; composes with the viewport. While filtering, letter keys type; actions bound to control keys (`:key :ctrl-d`, shown as `^d`) still fire, while plain-letter actions are shadowed.

The structural question worth deciding early

Staying in the loop. Today select returns on any action — so a dependency manager that deletes an item has to re-launch select to show the updated list, losing cursor position and flickering. Real tools want: delete → list updates in place → status line says "Removed lambdaisland/uri" → keep browsing → quit. **Delivered** (`docs/plans/2026-07-02-on-action-loop.md`): `:on-action` on `select` receives the (confirmed) action event and returns `{:items … :status …}`; items replace the list in place via `tlist/set-items` (cursor + filter preserved), the status shows until the next keypress, and the loop continues. Enter and esc still exit; the widgets stay pure (the handler is the caller's code).

Two ways to get there:

- An :on-action handler on select that receives the (confirmed) action event and returns {:items updated-items :status "Removed X"} — the loop continues with new items instead of exiting. Note this doesn't violate "widgets never call handlers": the list widget stays pure; select is the app-level orchestrator, and the handler is the caller's code.
- Or keep select as-is and document the "write your own run loop" pattern for stateful apps.

I lean toward :on-action plus a statuat clears on next keypress). It's thesingle most practical addition for real workflows, and the status line is useful independently ("copied to clipboard", "2 items selected").

Cheap and worth it

- Multi-select — space toggles a checkbox column, enter submits the set. Picking deps to upgrade, files to stage, tests to run. Mostly reuses the list widget: a :selected set and a marker column. **Delivered** (`docs/plans/2026-07-02-multi-select.md`): `:multi?` on the list — a `:selected` set of original indices (stable across filtering) and a `[x]`/`[ ]` checkbox column; `tui/multi-select` returns the chosen items (nil on cancel, [] on empty submit). Composes with filtering and the viewport; `set-items` clears the selection when items are replaced.
- layout/columns — align name  version  date fields across rows. Real lists are almost always tabular, and today every caller hand-pads in :item->text. A small helper (given rows of cells, pad to per-column max visible width) covers 90% of "tables" without a table widget. **Delivered** (`docs/plans/2026-07-02-layout-columns.md`): `layout/columns` pads each column to its max visible width (styled cells align), leaves the last cell unpadded, trims trailing blanks on ragged rows, and returns a vector of row strings. Callers zip the lines onto items and point `:item->text` at them; no table widget, no list change. Left-align only for now (right-align is a future add).
- Inline mode (worth discussing) — anhell prompt without the alternatescreen, gum/fzf style, erasing the widget on exit. Some small tools feel wrong going full-screen for a 5-item pick. Implementation is moderate: track rendered height, cursor-up N lines per frame. I'd rate it "very nice", not blocking. **Delivered** as `:inline?` on `run`/`select`/`confirm` (`docs/plans/2026-07-02-inline-mode.md`); no cross-frame height state needed — each frame returns the cursor to its own top-left.

One upstream flag

term/char-width in let-go is stubbed to always return 1, so any CJK or emoji in item text breaks pad/box/truncation today. tiny-tui can't fix that locally beyond guessing; the right fix is a real wcwidth in term.go, then style/visible-width uses it. Worth an upstream issue/PR whenever you touch let-go — until then it's a documented limitation.

What I'd deliberately keep out

Async/timers/spinners (the blocking read loop is the simplicity of the whole thing — a synchronous
tui/with-status "Fetching…" thunk thaocking call gets most of the valuewith none of the machinery), a layout engine, focus management for multiple live widgets per screen (the :mode swap pattern scaled fine for confirm and will scale to input), theme maps, and the diff renderer
(per-line clears are flicker-free eno

Sequencing, if you want my order

1. Viewport (unblocks real data, no API changes)
2. Input widget + tui/input
3. Filterable select (composes 1+2)
4. :on-action + status line (real workflows)
5. Multi-select
6. layout/columns

After step 4 you could genuinely build the lgx dependency manager, a branch switcher, or a process killer with nothing but tiny-tui — which feels like the right definition of "complete toolkit" for tproject.

What's the first real tool you want to build with it? That's probably the best forcing function for which slice to plan next — happy to help.