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

Ask a yes/no question:

```clojure
(when (tui/confirm {:title "Delete project?"
                    :message "This cannot be undone."})
  (delete-project!))
```

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
lgx run examples/inline_select.lg   # tui/select, inline (no alternate screen)
lgx run examples/confirm_delete.lg  # tui/confirm
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
