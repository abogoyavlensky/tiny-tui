# tiny-tui Roadmap 2 — from toolkit to small framework

The first roadmap (`ROADMAP.md`) asked "what turns V1 into something you'd
reach for weekly" and delivered it: viewport, input, filtering, `:on-action`,
multi-select, columns, inline mode. skl and wtr are proof. This roadmap asks
the next question: **what widens the variety of apps you can build** —
watchers, viewers, forms, dashboards — without giving up the property that
makes tiny-tui pleasant: pure widgets (`state + msg -> [state event]`,
`state -> string`), one blocking loop, terminal I/O confined to `screen` and
`key/read`.

## Principles (unchanged, restated)

- **Pure widgets, impure edges.** Every feature below must keep
  `update`/`view` pure and unit-testable; anything impure lives in `screen`,
  `key/read`, or a caller-supplied handler.
- **One loop, one mode at a time.** No focus engine, no windowing. The
  `:mode` swap pattern (list ↔ confirm) and chained widget calls (now with
  inline sessions) scale further than they look.
- **Helpers over engines.** `layout/columns` instead of a table widget;
  `tui/form` as chained inputs instead of focus management. Prefer a 40-line
  pure helper to a subsystem.
- **Opt-in complexity.** Every addition is a keyword option or a new
  namespace; a V1 app upgraded to the latest tag behaves identically.

## Where we are

Widgets: list (viewport, filter, multi-select, actions, `:on-action`),
input (validate, suggestions, path autocomplete), confirm, help.
Layout/style: vstack/hstack/pad/columns/box, 16/256 colors.
Modes: full-screen and inline. In flight
(`docs/plans/2026-07-03-word-editing-and-inline-session.md`): word-wise
editing keys and `with-inline-session` (one raw-mode span for a whole flow,
nested prompts from `:on-action`).

What the current model *cannot* express, in increasing order of structural
impact: long text (no truncation/wrapping/horizontal scroll), reading flows
(no pager), multi-field collection (no form), anything involving **time**
(the loop blocks on a keypress — no spinner, no progress, no auto-refresh),
and the mouse. Those gaps define the phases below.

---

## Phase 1 — Text at scale (pure helpers, no structural change)

Small, independent, high-frequency papercuts. Everything here is a pure
function plus tests; any of these can ship in an afternoon.

- **`style/truncate`** — cut styled text to a visible width with an
  ellipsis, preserving escape sequences and closing them properly. Today a
  long item wraps the row and breaks the frame; every list wants this.
  The list widget should apply it automatically to rows given its `:width`.
- **`layout/wrap`** — word-wrap styled text to a width, returning lines.
  Unlocks long confirm messages, help text, and is a prerequisite for the
  pager (Phase 3).
- **Right-align in `layout/columns`** — per-column `{:align :right}` spec
  (sizes, counts, durations). Explicitly deferred from V1.
- **Input horizontal scroll** — the input widget renders a window around the
  cursor when text exceeds the field width (`:tui/size` is already there).
  Today long input overflows the line; this is the input's missing viewport.
- **`:mask?` on input** — render `•` per char (passwords, tokens). ~5 lines
  in `field-str`, worth having for any auth-touching CLI.

**Unlocks:** every existing app gets sturdier with real-world data.

## Phase 2 — Time: `:tick-ms` (the one structural addition)

The old roadmap deliberately excluded async, and stays right about actors,
channels, and event queues. But a **poll-based tick** gets nearly all the
value with almost none of the machinery: with `:tick-ms 100` on `run`, the
loop uses `term/key-pending?` + a short sleep instead of a hard-blocking
read; when the deadline passes with no key, it injects a `:tick` message
like any other. No goroutines in the framework, no queues, no races —
`update` stays pure, `:tick` is just a message, and headless tests script it
like every other key.

Built on that, in order:

- **`:tick` plumbing in `run`** — opt-in; without `:tick-ms` the loop is
  byte-for-byte the current blocking read. `run` throttles view calls to one
  per tick (no redraw when update returns identical state, a cheap `=`).
- **`tui/with-status`** — run a blocking thunk (in a let-go goroutine) while
  the loop animates a spinner line and swallows keys; returns the thunk's
  value, restores the screen. The synchronous "Fetching…" story the old
  roadmap wanted, now honest. Confirm-before-slow-action flows (`skl`
  cloning a repo) stop looking frozen.
- **Spinner + progress widgets** — trivial pure views once `:tick` exists
  (`frames[(mod tick n)]`; a bar is `pad` math). Progress state is fed by
  the caller from `with-status`-style work that reports counts.
- **`:on-tick` for `select`** — symmetric with `:on-action`: a caller
  handler returning `{:items … :status …}` on a timer. This is the
  "watcher" unlock: an auto-refreshing process list, a CI-status board, a
  `wtr` dashboard that notices new worktrees — mini-htop-shaped apps with
  ~10 lines of app code.

**Unlocks:** spinners, progress, watchers, dashboards.
**Risk to watch:** battery/CPU of the poll loop — default to no tick, and
document 100–250ms as the sane range. Verify `key-pending?` + sleep behavior
on the pty harness before building on it.

## Phase 3 — Reading: the pager widget

A pure `tiny-tui.pager`: state `{:lines :offset :height :query}`, messages
`:up/:down/:pageup/:pagedown/:home/:end` (`g`/`G`), `/` to search with
`n`/`N` between matches (reuses the filter-input pattern from the list),
match highlighting via `style/inverse`. View is a slice of lines plus a dim
`"42/1300  57%"` indicator — same shape as the list viewport, so most of the
scroll math already exists to be shared or copied.

`tui/pager` helper: full-screen or inline, `q`/`esc` to close, returns nil
(it's a viewer, not a chooser). Optional `:actions` like select's, so a log
viewer can bind `y` = "copy line" or enter = "jump to this commit"
(returning an event with the current line).

**Unlocks:** log viewers, `git show`/diff review, README/help screens,
"details" drill-down from any list (select → pager → back, an inline-session
natural). This plus Phase 2 is `tail -f`: `:on-tick` appends lines, pager
sticks to bottom unless the user scrolled up.

## Phase 4 — Collecting: `tui/form` and grouped lists

- **`tui/form`** — multi-field collection *without* focus management: a
  vector of field specs (`{:id :title :widget :input|:select|:confirm
  :validate …}`) run as chained prompts inside one inline session; esc steps
  *back* a field instead of cancelling the flow (esc on the first field
  cancels); returns `{:id value …}` or nil. A review step at the end
  (rendered with `layout/columns`, enter = accept / field-key = edit) makes
  it feel like a real form while staying a sequence of the widgets we
  already have. This is the "project scaffolding wizard" / "release
  checklist" unlock, and skl's clone→pick→target flow expressed in one
  declaration.
- **Grouped lists** — `:groups` on the list widget: non-selectable dim
  header rows interleaved with items (cursor skips them, filtering hides
  empty groups). Real pickers group things ("Recent", "All"; worktrees by
  repo). Pure list-widget change, composes with viewport/filter/multi.
- **Per-item styling hook** — `:item-style` fn (item → attrs map for
  `style/styled`) so callers mark dirty/failed/disabled rows without
  pre-baking ANSI into `:item->text` (which fights the inverse cursor row
  today).

**Unlocks:** wizards, config editors, richer pickers.

## Phase 5 — Mouse (cheap, because upstream already did the hard part)

let-go's key tokenizer already keeps SGR mouse reports (`ESC[<b;x;yM`)
intact as single tokens and ships a decoder. What's missing is thin:

- `screen` enables/disables reporting (`ESC[?1006h` + `?1002l` writes on
  init/shutdown) behind `:mouse? true` on `run` — off by default.
- `key/parse` translates reports to messages: `{:type :mouse :action :press
  :button :left :x … :y …}`, wheel to `:wheel-up`/`:wheel-down`.
- Widgets opt in where it's natural and ignore the rest: wheel scrolls the
  list viewport and the pager; click moves the list cursor (click on the
  already-selected row = enter). Nothing else — no drag, no hover.

Hit-testing is the one design wrinkle (a click carries screen coordinates;
the widget knows only its own lines). Keep it at the `select`/`pager`
helper level, which knows the chrome layout — widgets stay coordinate-free
by receiving pre-translated messages (`:click-row 3`).

**Unlocks:** casual-use tools people don't live in — wheel-scrolling a log
or clicking a pick is the difference between "TUI" and "app" for
non-terminal-native users.

## Phase 6 — Polish that compounds

- **Theme accents** — not a theming engine: one small map
  (`{:accent :cyan :error :red :dim-style …}`) consulted by widgets for
  their hardcoded choices (cursor marker, error line, checkbox, indicator),
  settable once via `tui/set-theme!`. Apps get brand consistency; default
  behavior unchanged.
- **"Write your own widget" guide** — the framework's versatility ceiling is
  custom widgets, and the pattern (pure `create`/`update`/`view`, run via
  `tui/run`, headless tests, `:mode` composition) is fully established but
  only documented by example. One `docs/custom-widgets.md` with a worked
  example (e.g. a two-pane picker) multiplies what others build.
- **`tiny-tui.test` helpers** — `script-keys` (feed a vector of messages as
  `:read-key-fn`), `render-to-string` (fixed `:tui/size`, stripped or raw),
  cutting the boilerplate every downstream app currently copies from
  tiny-tui's own tests. Snapshot-friendly by construction.
- **Bracketed paste** — enable `ESC[?2004h`; `key/read` accumulates between
  `200~`/`201~` and delivers one string message. Pasting a path or token
  into input today trickles char-by-char through suggest-fn recomputes (and
  a pasted ESC can cancel); paste becomes atomic and safe. Mostly a `key`
  change; verify on the pty harness.

## Upstream flags (let-go)

- **`term/char-width` is stubbed to 1** — CJK/emoji still break padding,
  truncation, and box drawing. `style/visible-width` is the single choke
  point, so the day let-go grows a real wcwidth, tiny-tui inherits
  correctness everywhere. Worth filing/PRing; until then a documented
  limitation (and `style/truncate` should at least not *split* a multi-byte
  rune).
- **`term/sleep` / time primitive** — Phase 2's poll loop needs a portable
  short sleep; check what let-go exposes before planning the tick.

## Deliberate non-goals (still)

- A layout engine, flexbox, or constraint solver — `vstack`/`columns`/`wrap`
  compose far enough.
- Focus management / multiple live widgets per screen — `:mode` swap,
  chained prompts, and `tui/form` cover the real cases.
- A multiline text editor — that's legmacs's job.
- A general async runtime, event queue, or subscriptions — `:tick` +
  caller-owned goroutines inside `with-status` is the ceiling.
- A diff renderer — per-line clears remain flicker-free at these frame
  rates; revisit only if a profiled app proves otherwise.
- Windows/panes/tabs, drag & drop, hover states.

## Suggested order and why

1. **Phase 1** (text at scale) — pure, independent, makes today's apps
   sturdier; good warm-up tasks.
2. **Phase 2** (`:tick-ms` + `with-status` + `:on-tick`) — the single
   biggest expansion of the app space; do it early so later phases (pager
   tailing, form spinners) can assume time exists.
3. **Phase 3** (pager) — second-biggest unlock, benefits from `wrap` (1)
   and tick (2).
4. **Phase 4** (form, grouped lists) — builds on inline sessions once
   they've proven out in skl/wtr.
5. **Phase 5** (mouse) — anytime after the pager exists (wheel needs a
   scroll target); independent otherwise.
6. **Phase 6** — continuous; the custom-widget guide and test helpers are
   worth pulling forward whenever a downstream app starts copying
   boilerplate.

The forcing function, as with V1: pick the next real tool first. A
`watch`-style CI/process dashboard would drive Phases 1–2; a log/diff
viewer drives 1–3; a project scaffolder drives 4. Building the tool and the
phase together keeps every feature honest.
