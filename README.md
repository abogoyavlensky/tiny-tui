# tiny-tui

A minimal terminal UI library for [let-go](https://github.com/nooga/let-go).
It adds small interactive moments to otherwise normal command-line tools:
pick an item from a list, run an action on it, confirm a destructive step.
It is not a full TUI framework.

## Quick start

Pick an item:

```clojure
(ns my.tool
  (:require [tiny-tui.core :as tui]))

(def result
  (tui/select {:title "Select project"
               :items [{:id 1 :name "tiny-cli"}
                       {:id 2 :name "tiny-tui"}
                       {:id 3 :name "lgx"}]
               :item->text :name}))

(case (:type result)
  :select (println "Selected" (:name (:item result)))
  :cancel nil)
```

Add actions on the selected item. An action marked `:destructive?` (or
`:confirm?`) asks for confirmation first and returns with `:confirmed? true`:

```clojure
(tui/select
 {:title "Dependencies"
  :items deps
  :item->text :name
  :actions [{:id :open :key "o" :label "open"}
            {:id :delete :key "d" :label "delete"
             :destructive? true
             :confirm-title "Delete dependency?"
             :confirm-message (fn [dep] (str "Remove " (:name dep) "?"))}]})
;; => {:type :action :action :delete :item {...} :index 1
;;     :destructive? true :confirmed? true}
```

Long lists scroll to a window sized to the terminal, keeping the selection
in view with a dim `12/50` position indicator — no configuration needed.

Set `:filterable? true` to let the user type to narrow the list (fzf style):
matching is case-insensitive substring on the item text, or pass `:filter-fn
(fn [query item] boolean)` for custom matching. Arrows navigate the matches
and enter selects. While filtering, letter keys type into the query, so bind
actions to control keys (e.g. `:key :ctrl-d`) if you want them to fire while
the filter is active — plain-letter actions are shadowed by typing. Only esc
cancels.

```clojure
(tui/select {:title "Checkout branch" :items branches
             :item->text :name :filterable? true})
```

Stay in the loop with `:on-action`. Normally an action returns and exits;
with a handler, it updates the list in place and keeps browsing. The handler
receives the (confirmed) action event and returns `{:items new-items :status
"..."}` — the items replace the list (cursor and filter preserved) and the
status shows until the next keypress. Enter and esc still exit. The list
widget stays pure; the handler is your code:

```clojure
(let [deps (atom initial-deps)]
  (tui/select {:title "Dependencies" :items @deps :item->text :name
               :actions [{:id :remove :key "d" :label "remove" :destructive? true}]
               :on-action (fn [ev]
                            (swap! deps remove-item (:item ev))
                            {:items @deps :status (str "Removed " (:name (:item ev)))})}))
```

Pick several with `tui/multi-select` — space toggles a checkbox, enter
submits. It returns the chosen items in list order (`nil` on cancel, `[]` on
an empty submit) and composes with `:filterable?`:

```clojure
(tui/multi-select {:title "Stage files" :items files :item->text :name})
;; => [{...} {...}]   the checked items
```

Align tabular rows with `layout/columns` — it pads each column to its widest
cell (by visible width, so styled cells line up) and leaves the last column
unpadded. Build the aligned lines once, then use them as `:item->text`:

```clojure
(let [lines (layout/columns (map (fn [d] [(:name d) (:version d)]) deps))
      items (map (fn [d line] (assoc d :row line)) deps lines)]
  (tui/select {:title "Dependencies" :items items :item->text :row}))
;;  › org.clojure/data.json  2.5.1
;;    lambdaisland/uri       1.19.155
```

Ask a yes/no question:

```clojure
(when (tui/confirm {:title "Delete project?"
                    :message "This cannot be undone."})
  (delete-project!))
```

Prompt for a line of text. `tui/input` returns the string on submit, or
`nil` on cancel. A `:validate` fn (text → error string or `nil`) blocks
submit and shows an inline error until the input is valid:

```clojure
(let [name (tui/input {:title "What's your name?"
                       :placeholder "type a name"
                       :validate (fn [text]
                                   (when (empty? text) "Name can't be empty"))})]
  (when name (println "Hello," name)))
```

Single-line only. Left/right/home/end move the cursor; backspace and delete
edit; enter submits; esc cancels.

Build a custom app on the same loop the helpers use:

```clojure
(tui/run
 {:init {:count 0}
  :update (fn [state msg]
            (case msg
              :up [(update state :count inc) nil]
              :down [(update state :count dec) nil]
              "q" [state :quit]
              [state nil]))
  :view (fn [state] (str "Count: " (:count state)))})
;; => [final-state event]
```

`update` receives keywords for special keys (`:up` `:down` `:enter` `:esc`)
and one-character strings for printable keys. Returning a non-nil event
stops the loop. The runtime handles Ctrl-C, terminal resize, raw mode, the
alternate screen, and cleanup on exceptions.

### Inline mode

By default a widget takes over the alternate screen. Pass `:inline? true` to
`select`, `confirm`, or `run` to render in place at the cursor instead — gum/
fzf style — erasing the widget on exit so whatever you print next lands where
it was:

```clojure
(let [result (tui/select {:title "Flavors" :items flavors
                          :item->text :name :inline? true})]
  (println "Scooped" (:name (:item result))))  ; prints where the list was
```

Best for small pickers that fit on screen; a widget taller than the terminal
can't scroll back, so reach for the default full-screen mode for large UIs.

## How it works

Widgets are pure: `state + message -> [new-state event]` and
`state -> string`. Only `tiny-tui.screen` and `tiny-tui.key/read` touch the
terminal. This makes every flow testable without a terminal; pass
`:read-key-fn`, `:render-fn`, and `:screen false` to `run`, `select`, or
`confirm` to drive them with scripted keys (see `test/tiny_tui/core_test.lg`).

Namespaces: `core` (run/select/confirm), `list` and `confirm` (widgets),
`style` and `layout` and `help` (string rendering), `screen` and `key`
(terminal I/O).

## Examples

```bash
lgx run examples/hello_screen.lg    # enter/leave the terminal
lgx run examples/static_screen.lg   # styled static screen
lgx run examples/counter.lg         # interactive counter
lgx run examples/select_project.lg  # tui/select
lgx run examples/viewport_select.lg # long list with a scroll window
lgx run examples/filter_select.lg   # type-to-filter select
lgx run examples/deps_manager.lg    # delete-in-place with :on-action
lgx run examples/multi_select.lg    # multi-select with checkboxes
lgx run examples/columns_select.lg  # aligned tabular select (layout/columns)
lgx run examples/inline_select.lg   # tui/select, inline (no alternate screen)
lgx run examples/confirm_delete.lg  # tui/confirm
lgx run examples/input_name.lg      # tui/input with validation
lgx run examples/deps_actions.lg    # actions + confirmation
```

## Development

Install dependencies with [mise](https://mise.jdx.dev/getting-started.html)
(or manually, consulting the `.mise.toml` file):

```bash
mise trust && mise install
```

Run main application commands:

```bash
lgx --help
lgx run    # select demo (main.lg)
lgx test
lgx fmt
```
