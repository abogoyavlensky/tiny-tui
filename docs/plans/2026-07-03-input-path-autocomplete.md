# Input Path Autocomplete Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add filesystem path autocomplete to `tui/input` (generic suggestion mechanism + a path-completion helper), then wire it into skl's target-directory prompt.

**Tech Stack:** let-go (lgx), tiny-tui, `os` module (`os/ls`, `os/stat`, `os/getenv`)

---

## Design

### Approach

Extend the existing `tiny-tui.input` widget with generic suggestion support rather than building a separate widget — a new widget would duplicate all of input's editing/cursor/validate machinery. The widget stays pure: it never touches the filesystem; it only renders and navigates suggestions already in its state.

The impure work (listing directories) lives in a caller-supplied `:suggest-fn (fn [text] -> [candidate-strings])`, invoked by the `tui/input` wrapper in `core.lg` after every text-changing message — the same pattern `select` uses for `:on-action` to keep impure caller code out of widgets.

A new `tiny-tui.path` namespace ships the ready-made filesystem helper: pure completion logic (unit-testable with injected entries) plus a thin impure `suggest-fn` factory around `os/ls`/`os/stat`. This relaxes AGENTS.md's "I/O confined to screen and key/read" note; AGENTS.md gets updated.

### Widget behavior (`tiny-tui.input`)

New state keys, set in `create`:
- `:suggestions` — vector of candidate strings (full replacement texts, not fragments). Empty by default.
- `:suggestion-cursor` — index of the highlighted suggestion, 0 by default.

New pure setter `set-suggestions [s suggestions]` — replaces `:suggestions`, resets `:suggestion-cursor` to 0.

New messages in `update` (all emit no event):
- `:down` / `:up` — move `:suggestion-cursor`, clamped to `[0, count-1]` (no wrap, matching list's edge behavior). No-ops when suggestions are empty.
- `:tab` — with suggestions present: replace `:text` with the highlighted suggestion, move `:cursor` to end, clear `:error`. With none: no-op.

Unchanged: `:enter` submits the current text as-is (subject to `:validate`); `:esc` cancels; editing keys behave as before. Tab-accept is the only way a suggestion enters the text — shell mental model: tab completes, enter submits.

`view` renders suggestion rows below the field (after the error line's position): each row is `"  › "`-style marker + text for the highlighted row (rendered with `style/inverse` on the text), `"    "` + `style/dim` text for the rest — visually consistent with the list widget's cursor marker. No rows render when `:suggestions` is empty.

### Wrapper behavior (`tui/input` in core.lg)

New opt `:suggest-fn`. When present:
- Seed initial suggestions by calling it on the initial `:value` text at create time (so a prefilled value like `.agents/skills` shows completions immediately).
- After each `tinput/update` where `:text` changed, call `(suggest-fn new-text)`, cap with `(take 5 ...)`, and inject via `tinput/set-suggestions`. Unchanged text (pure cursor motion, `:up`/`:down`) must NOT re-run it.
- Help line becomes `tab complete · ↑/↓ navigate · enter submit · esc cancel` (only when `:suggest-fn` is set; the plain input keeps its current bindings).

`suggest-fn` is expected to be total (return `[]` rather than throw) — the wrapper does not guard, per the let-go `try` gotcha.

### Path helper (`tiny-tui.path`)

Pure core (bottom-up `defn` order — no forward references):
- `split-input [text]` → `{:dir "..." :prefix "..."}` — `:dir` is everything up to and including the last `/` (`""` when no slash, meaning the current directory), `:prefix` is the partial name after it.
- `candidates [entries text opts]` → sorted vector of candidate strings. `entries` is a seq of `{:name "x" :dir? bool}` (injected, so this stays pure). Rules:
  - Case-sensitive prefix match of `:name` against `:prefix` (shell-style).
  - `:dirs-only? true` in opts drops non-directories.
  - Dotfiles are hidden unless `:prefix` itself starts with `.`.
  - Directory candidates get a trailing `/` appended — so tab-accepting a dir lets the next keystroke's re-suggest descend into it. (The widget stays generic; the slash convention lives here.)
  - Each candidate is `(str dir name)` — a full replacement for the typed text.

Impure factory:
- `suggest-fn [opts]` → `(fn [text] ...)`:
  - Expand a leading `~/` to `$HOME` for listing only; returned candidates keep the typed `~/` form.
  - Resolve the listing dir from `split-input` (empty `:dir` lists `"."`).
  - Return `[]` when the dir is missing or not a directory (check `(:dir? (os/stat dir))` first, like skl's `list-skills`) — never throw.
  - Build entries via `os/ls` + `(:dir? (os/stat (str dir "/" entry)))`, then delegate to `candidates`.

### Testing strategy

- Unit tests for the new input messages and `set-suggestions` (same style as `input_test.lg`).
- Unit tests for `tiny-tui.path`'s pure `split-input`/`candidates` with injected entries; an integration test for `suggest-fn` against a temp dir created with `os/sh mkdir -p`.
- A headless flow test through `tui/input` (`:screen false`, scripted `:read-key-fn`, fake suggest-fn) covering type → tab → enter.
- New `examples/input_path.lg` (inline, dirs-only).
- pty verification per `docs/pty-verification.md` — the view grows/shrinks rows in inline mode, so suggestion rows appearing/disappearing must erase cleanly, including cancel paths (esc, Ctrl-C).

### skl wiring

`resolve-target` in `../skl/src/skl/commands.lg` adds `:suggest-fn (path/suggest-fn {:dirs-only? true})` to its `tui/input` opts. `tui-opts` still merges last, so tests can override `:suggest-fn` with a deterministic fake.

## File Structure

- Modify: `src/tiny_tui/input.lg` — suggestion state, `:tab`/`:up`/`:down` handling, `set-suggestions`, suggestion rows in `view`.
- Create: `src/tiny_tui/path.lg` — pure `split-input`/`candidates`, impure `suggest-fn` factory.
- Modify: `src/tiny_tui/core.lg` — `:suggest-fn` wiring in the `input` wrapper, conditional help bindings.
- Modify: `test/tiny_tui/input_test.lg` — widget suggestion tests.
- Create: `test/tiny_tui/path_test.lg` — pure logic + temp-dir integration tests.
- Modify: `test/tiny_tui/core_test.lg` — headless autocomplete flow test.
- Create: `examples/input_path.lg` — runnable demo.
- Modify: `README.md` (input section), `AGENTS.md` (I/O confinement note).
- Modify (separate repo): `../skl/src/skl/commands.lg` + its test.

---

### Task 1: Suggestion state and messages in the input widget

**Files:**
- Modify: `src/tiny_tui/input.lg`
- Test: `test/tiny_tui/input_test.lg`

- [x] **Step 1: Write failing tests**
  In `input_test.lg`, using the existing `ups` helper:
  - `create` defaults: `:suggestions` is `[]`, `:suggestion-cursor` is `0`.
  - `set-suggestions` replaces the vector and resets the cursor to 0.
  - `:down`/`:up` move the cursor, clamped at both ends; no-ops with empty suggestions; emit no event.
  - `:tab` with suggestions `["src/" "test/"]` and cursor 1 sets `:text` to `"test/"`, `:cursor` to 5, clears a pending `:error`; emits no event.
  - `:tab` with empty suggestions leaves state unchanged.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — new assertions on missing keys/behavior.

- [x] **Step 3: Implement**
  In `input.lg`: add `:suggestions`/`:suggestion-cursor` to `create`, private `accept-suggestion` and `move-suggestion` helpers (defined before `update` — no forward refs), public `set-suggestions`, and the `:tab`/`:up`/`:down` branches in `update`. Update the namespace docstring and `update`'s docstring.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS, no regressions.

- [x] **Step 5: Commit**
  `git commit -m "feat: add suggestion state and tab-accept to input widget"`

### Task 2: Render suggestions in the input view

**Files:**
- Modify: `src/tiny_tui/input.lg`
- Test: `test/tiny_tui/input_test.lg`

- [x] **Step 1: Write failing tests**
  View tests (compare against `layout/vstack`-built expected strings, as existing view tests do):
  - With suggestions, rows render below the field: highlighted row as marker + `style/inverse` text, others indented + `style/dim`.
  - With empty suggestions, the view is byte-identical to today's output.
  - Error line and suggestions can coexist (error renders in its current position).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [x] **Step 3: Implement**
  Add a private `suggestion-lines` helper and splice it into `view`'s `vstack` (blank separator line before the block, matching the widget's existing spacing rhythm).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: render input suggestions with highlighted row"`

### Task 3: Pure path-completion logic

**Files:**
- Create: `src/tiny_tui/path.lg`
- Create: `test/tiny_tui/path_test.lg`

- [x] **Step 1: Write failing tests**
  For `split-input`: `"src/ti"` → `{:dir "src/" :prefix "ti"}`; `"ti"` → `{:dir "" :prefix "ti"}`; `"src/"` → `{:dir "src/" :prefix ""}`; `"~/pro"` → `{:dir "~/" :prefix "pro"}`.
  For `candidates` with injected entries:
  - Prefix filtering is case-sensitive; results sorted.
  - Dirs get trailing `/`; files don't.
  - `:dirs-only? true` drops files.
  - Dotfile entries hidden for prefix `""`/`"s"`, shown for prefix `"."`.
  - Candidates include the `:dir` part: entries `[{:name "tiny_tui" :dir? true}]` with text `"src/ti"` → `["src/tiny_tui/"]`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL (namespace doesn't exist yet).

- [x] **Step 3: Implement**
  Create `path.lg` with `split-input` and `candidates` only (pure, `defn` order bottom-up). Namespace docstring states the purity split: logic here, `os` access only in `suggest-fn` (Task 4).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: add pure path completion logic"`

### Task 4: Impure suggest-fn factory

**Files:**
- Modify: `src/tiny_tui/path.lg`
- Test: `test/tiny_tui/path_test.lg`

- [x] **Step 1: Write failing tests**
  Build a fixture tree in a temp dir via `(os/sh "mkdir" "-p" ...)` (and a plain file via `os/sh touch`), e.g. `<tmp>/alpha/`, `<tmp>/beta/`, `<tmp>/note.txt`. Assert:
  - `((suggest-fn {}) "<tmp>/a")` → `["<tmp>/alpha/"]`.
  - `((suggest-fn {:dirs-only? true}) "<tmp>/")` → dirs only.
  - Missing dir → `[]`; text pointing at a file path → `[]`. Never throws.
  Clean the fixture up with the capture/cleanup/rethrow pattern from `screen/with-screen*` (let-go `try`/`finally` gotcha).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [x] **Step 3: Implement**
  Add `suggest-fn` to `path.lg` (after the pure fns): `~/` expansion via `(os/getenv "HOME")` for listing only, `os/stat` dir check → `[]` fallback, `os/ls` + per-entry `os/stat` to build `{:name :dir?}` entries, delegate to `candidates`.
  _Note: codex review caught that `os/ls` throws `permission denied` on an existing-but-unreadable directory (`stat-dir?` alone wasn't enough). Replaced the pre-check with a `safe-ls` guard that returns nil on any listing failure (missing/file/unreadable), plus a regression test._

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: add filesystem suggest-fn for path completion"`

### Task 5: Wire :suggest-fn into the tui/input wrapper

**Files:**
- Modify: `src/tiny_tui/core.lg`
- Test: `test/tiny_tui/core_test.lg`

- [x] **Step 1: Write failing tests**
  Headless (`:screen false`, scripted `:read-key-fn`, no-op `:render-fn`) with a fake suggest-fn (pure fn over a fixed map, recording its calls in an atom):
  - Prefilled `:value` seeds suggestions (suggest-fn called once before any key).
  - Typing a char re-runs suggest-fn; `:down` then `:tab` accepts the second candidate; suggest-fn re-runs on the accepted text; `:enter` returns the completed string.
  - Suggest-fn is NOT called on pure cursor motion (`:left`, `:up`/`:down`).
  - More than 5 candidates → widget state holds only 5 (assert via the view or a render-fn capture).
  - Without `:suggest-fn`, behavior and help line are unchanged.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL.

- [x] **Step 3: Implement**
  In `core.lg`'s `input`: seed suggestions at init when `:suggest-fn` is set; in the update fn, compare old/new `:text` and re-suggest (capped `take 5`) only on change; add the suggest-mode help bindings (`tab complete`, `↑/↓ navigate`). Update `input`'s docstring with `:suggest-fn`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS, full suite green.

- [x] **Step 5: Commit**
  `git commit -m "feat: add :suggest-fn autocomplete to tui/input"`

### Task 6: Example, pty verification, and docs

**Files:**
- Create: `examples/input_path.lg`
- Modify: `README.md`, `AGENTS.md`

- [ ] **Step 1: Write the example**
  `examples/input_path.lg`: inline `tui/input` titled "Install skills to", `:value ".agents/skills"`, `:suggest-fn (path/suggest-fn {:dirs-only? true})`, blank-text validation; prints the chosen path or "Cancelled.".

- [ ] **Step 2: Verify on a pty**
  Per `docs/pty-verification.md`, drive `lgx run examples/input_path.lg` with scripted bytes: type a partial dir name, tab (byte 9), arrows, enter — and separately the cancel paths (esc, Ctrl-C). Inspect raw output: suggestion rows appear/disappear and the inline widget erases cleanly on exit.
  Expected: completed path printed on submit; clean erase on every exit path.

- [ ] **Step 3: Update docs**
  README input section: document `:suggest-fn`, the tab/arrows keys, and `tiny-tui.path/suggest-fn` with a short example; add `input_path.lg` to the examples list. AGENTS.md: amend the I/O confinement sentence to include `tiny-tui.path/suggest-fn`.

- [ ] **Step 4: Run `lgx fmt` and the full suite**
  Run: `lgx fmt && lgx test`
  Expected: clean fmt, all tests PASS.

- [ ] **Step 5: Commit**
  `git commit -m "docs: add path autocomplete example and docs"`

### Task 7: Wire autocomplete into skl's target prompt

**Files (in `../skl`):**
- Modify: `src/skl/commands.lg`
- Test: skl's existing commands test (scripted-key flow tests)

- [ ] **Step 1: Write failing test**
  In skl's test for the target prompt: pass a fake `:suggest-fn` via `tui-opts`... note `tui-opts` merges *after* the base opts, so a test-supplied `:suggest-fn` overrides production's. Assert the flow still resolves the target when tab is scripted, and that production code sets `:suggest-fn` (e.g. a unit test that `resolve-target`'s input opts include it, or a flow test scripting tab against a real temp dir).

- [ ] **Step 2: Run skl tests to verify it fails**
  Run: `cd ../skl && lgx test`
  Expected: FAIL.

- [ ] **Step 3: Implement**
  In `resolve-target`: require `tiny-tui.path`, add `:suggest-fn (path/suggest-fn {:dirs-only? true})` to the base input opts (before the `tui-opts` merge). Ensure skl's pinned tiny-tui version/checkout includes the new feature.

- [ ] **Step 4: Run skl tests to verify they pass**
  Run: `cd ../skl && lgx test`
  Expected: PASS. Also verify manually on a pty: `skl add <url>` prompts with live dir suggestions.

- [ ] **Step 5: Commit (in ../skl)**
  `git commit -m "feat: autocomplete local dirs in install target prompt"`
