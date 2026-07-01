# tiny-tui Initial Design

`tiny-tui` is a minimal terminal UI library written specifically for `let-go`.
It is designed for people building small command-line tools with `tiny-cli` who occasionally need a richer interactive flow: choosing an item from a list, running an action against the selected item, and confirming destructive operations.

The library should start small, stay easy to understand, and produce a usable result after each development stage.

## Goals

`tiny-tui` should make it easy to add beautiful interactive elements to otherwise normal CLI tools.

The first version should support:

- Rendering simple terminal screens.
- Reading keyboard input in raw mode.
- Navigating a list with arrow keys.
- Selecting an item with Enter.
- Showing a help line with available keyboard shortcuts.
- Defining custom actions for the currently selected item.
- Confirmation actions with a simple yes/no selection or popup (see below).

The library should feel natural in `let-go`: data maps, simple functions, small namespaces, and predictable state transitions.

## Non-goals for the first version

The first version should deliberately avoid becoming a full TUI framework.

Out of scope initially:

- Mouse support.
- Pagination.
- Virtualized long lists.
- Text input widgets.
- Async tasks, timers, spinners, progress bars.
- Complex layout engine.
- React/Solid-style render tree.
- Incremental/diff renderer.
- Terminal feature detection beyond basic size/input/output needs.

These can be added later only if the simple model proves too limiting.

## Design principles

### 1. Start as a CLI helper, not an app framework

The primary use case is:

```clojure
(defn choose-project-command [opts]
  (let [project (tui/select {:title "Choose project"
                             :items projects
                             :item->text :name})]
    (when project
      (println "Selected" (:name project)))))
```

A `tiny-cli` command should be able to call `tiny-tui`, get a result, and continue as a normal command-line program.

### 2. Keep rendering string-based

Every widget should render to a string.

```clojure
(list/view list-state) ;; => "..."
(help/view bindings)   ;; => "↑/↓ navigate   enter select   q quit"
```

This keeps the library simple, inspectable, and testable.

### 3. Keep widgets pure

Widgets should not perform side effects. They should receive state and a message, then return new state plus an optional event.

```clojure
(list/update list-state :down)
;; => [new-list-state nil]

(list/update list-state :enter)
;; => [same-list-state {:type :select :item item :index 2}]
```

The program/runtime layer handles terminal I/O. The application layer decides what to do with events.

### 4. Prefer convenience APIs first, composable APIs underneath

The first public impression should be simple:

```clojure
(tui/select {...})
(tui/confirm {...})
```

Underneath, the same functionality should be available as composable pieces:

```clojure
(tui/run {:init state
          :update update
          :view view})
```

This gives simple tools an easy path and gives more advanced tools room to grow.

## Conceptual architecture

```text
tiny-tui.core
  run
  select
  confirm

tiny-tui.screen
  init!
  shutdown!
  render!
  clear!

tiny-tui.key
  read
  parse
  match?

tiny-tui.style
  fg
  bg
  bold
  dim
  inverse
  reset
  styled

tiny-tui.layout
  lines
  vstack
  hstack
  pad
  box
  center

tiny-tui.help
  view

tiny-tui.list
  create
  update
  view
  selected-index
  selected-item

tiny-tui.confirm
  create
  update
  view
  active?
```

The implementation can start with fewer namespaces and split later. The important boundary is conceptual: terminal side effects live in `screen`/`key`; widgets stay pure.

## Runtime model

The core loop is intentionally small:

```clojure
(defn run [{:keys [init update view]}]
  (screen/with-screen
    (loop [state init]
      (screen/render! (view state))
      (let [msg (key/read)
            [next-state event] (update state msg)]
        (if (= event :quit)
          next-state
          (recur next-state))))))
```

The real implementation may use different names, but the model should stay close to this.

The runtime should guarantee terminal cleanup with `try`/`finally`, especially raw mode, cursor visibility, and alternate screen state.

## Data model

### Messages

V1 messages can be plain keywords or raw printable strings.

```clojure
:up
:down
:enter
:esc
:ctrl-c
"q"
"d"
"o"
```

A more detailed event map can be added later if needed.

### Commands / effects

V1 can avoid a full command system. The runtime only needs one special event:

```clojure
:quit
```

Widgets can return semantic events:

```clojure
{:type :select
 :item item
 :index index}

{:type :action
 :action :delete
 :item item
 :index index}

{:type :confirm
 :value true}
```

The application interprets these events.

## Development stages

Each stage should leave the project in a useful state.

## Stage 0 — Terminal safety and basic screen control

Purpose: make it safe to enter and leave interactive terminal mode.

This stage introduces only the lowest-level pieces:

```clojure
(screen/init!)
(screen/shutdown!)
(screen/clear!)
(screen/render! "Hello")
(screen/with-screen body)
```

Expected behaviour:

- Switch to alternate screen if available.
- Enable raw mode.
- Hide cursor while the TUI is active.
- Restore cursor and raw mode on exit.
- Restore terminal even if an exception happens.

Usable result:

```clojure
(screen/with-screen
  (screen/render! "Hello from tiny-tui"))
```

This is not interactive yet, but it proves that terminal ownership is safe.

## Stage 1 — Simple rendering primitives

Purpose: make static terminal output pleasant.

Add minimal style and layout helpers:

```clojure
(style/bold "Title")
(style/dim "q quit")
(style/fg :cyan "Projects")
(layout/box "content")
(layout/vstack title body help)
(help/view [{:key "q" :label "quit"}])
```

The first design should avoid a full style object system. Simple functions are enough:

```clojure
(style/bold (style/fg :cyan "Projects"))
```

Usable result:

```text
┌────────────┐
│ Projects   │
└────────────┘

No interaction yet.

q quit
```

This stage is useful for CLI commands that want a nicer summary screen.

## Stage 2 — Key parsing and minimal program loop

Purpose: make the screen interactive.

Add raw key reading and normalized key values:

```clojure
(key/read)  ;; => :up, :down, :enter, :esc, "q", ...
```

Add a small `tui/run` function:

```clojure
(tui/run
  {:init {:count 0}
   :update (fn [state msg]
             (case msg
               :up [(update state :count inc) nil]
               :down [(update state :count dec) nil]
               "q" [state :quit]
               [state nil]))
   :view (fn [state]
           (str "Count: " (:count state) "\n\n"
                "↑/↓ change   q quit"))})
```

Usable result:

A tiny interactive counter/demo app.

This stage validates the foundational architecture before adding widgets.

## Stage 3 — List widget without pagination

Purpose: provide the first real useful widget.

Add a pure list widget:

```clojure
(def list-state
  (list/create
    {:items projects
     :item->text :name}))
```

Core functions:

```clojure
(list/update list-state :up)
(list/update list-state :down)
(list/update list-state :enter)
(list/view list-state)
(list/selected-item list-state)
(list/selected-index list-state)
```

Behaviour:

- Up/down moves the cursor.
- Cursor does not move outside the list bounds.
- Enter returns a `:select` event.
- Empty list renders a friendly empty state.
- No pagination.
- No scrolling.

Example render:

```text
Choose project

› tiny-cli
  tiny-tui
  lgx
  wtr

↑/↓ navigate   enter select   q quit
```

Usable result:

A custom app can already render a list, navigate it, and get a selected item.

## Stage 4 — High-level `select` helper

Purpose: make the common case extremely easy.

Build `tui/select` on top of the list widget and program loop.

```clojure
(tui/select
  {:title "Choose project"
   :items projects
   :item->text :name})
```

Return value:

```clojure
{:type :select
 :item selected-item
 :index selected-index}
```

If the user quits:

```clojure
{:type :cancel}
```

Usable result:

A `tiny-cli` command can call `tui/select` directly and use the result.

```clojure
(defn open-project [_opts]
  (let [res (tui/select {:title "Open project"
                         :items projects
                         :item->text :name})]
    (when (= :select (:type res))
      (open! (:item res)))))
```

This is likely the first version that is useful in real tools.

## Stage 5 — Custom item actions

Purpose: let users do more than select.

Extend list config with actions:

```clojure
(list/create
  {:items projects
   :item->text :name
   :actions [{:id :open
              :key "o"
              :label "open"}
             {:id :delete
              :key "d"
              :label "delete"
              :destructive? true}]})
```

When the action key is pressed, the widget returns an action event:

```clojure
{:type :action
 :action :delete
 :item item
 :index index
 :destructive? true}
```

The list widget should not call handlers itself. The caller decides what happens.

Help line should be generated from available bindings:

```text
↑/↓ navigate   enter select   o open   d delete   q quit
```

Usable result:

A CLI tool can present a list and allow actions against the selected item.

Example:

```clojure
(tui/select
  {:title "Dependencies"
   :items deps
   :item->text :name
   :actions [{:id :remove :key "d" :label "remove" :destructive? true}]})
```

At this point destructive actions can still return immediately. Confirmation comes next.

## Stage 6 — Confirmation

Think to make a popup or palin simple list selection yes/no confirmation.
I don't know what is simpler clearer and more intuitive for users.

Purpose: support safe destructive flows.

Add a pure confirmation widget:

```clojure
(confirm/create
  {:title "Delete dependency?"
   :message "Remove org/example from lgx.edn?"
   :confirm-label "delete"
   :cancel-label "cancel"})
```

Keyboard behaviour:

```text
y / enter  confirm
n / esc    cancel
```

Events:

```clojure
{:type :confirm :value true}
{:type :confirm :value false}
```

Example render:

```text
┌────────────────────────────────────┐
│ Delete dependency?                 │
│                                    │
│ Remove org/example from lgx.edn?   │
│                                    │
│ y delete   n cancel                │
└────────────────────────────────────┘
```

Usable result:

A command can call `tui/confirm` directly:

```clojure
(when (tui/confirm {:title "Delete project?"
                    :message "This cannot be undone."})
  (delete-project! project))
```

This stage is useful even without list integration.

## Stage 7 — List actions with confirmation popup

Purpose: combine list actions and confirmation into a real workflow.

When a list action has `:confirm? true` or `:destructive? true`, `tui/select` opens a confirmation popup instead of immediately returning the action.

```clojure
(tui/select
  {:title "Projects"
   :items projects
   :item->text :name
   :actions [{:id :delete
              :key "d"
              :label "delete"
              :destructive? true
              :confirm-title "Delete project?"
              :confirm-message (fn [project]
                                 (str "Delete " (:name project) "?"))}]})
```

Application state becomes slightly richer:

```clojure
{:mode :list
 :list list-state
 :confirm nil
 :pending-action nil}
```

When confirmation opens:

```clojure
{:mode :confirm
 :list list-state
 :confirm confirm-state
 :pending-action {:action :delete
                  :item item
                  :index index}}
```

Routing rule:

- If confirmation is active, all keys go to confirmation first.
- If confirmation is cancelled, return to list mode.
- If confirmation is accepted, return the original action event with `:confirmed? true`.

Usable result:

A full practical interactive list:

```text
Projects

› tiny-cli
  tiny-tui
  lgx

↑/↓ navigate   enter select   d delete   q quit
```

Pressing `d` opens:

```text
┌────────────────────┐
│ Delete project?    │
│                    │
│ Delete tiny-cli?   │
│                    │
│ y delete   n cancel│
└────────────────────┘
```

The caller receives:

```clojure
{:type :action
 :action :delete
 :item item
 :index index
 :confirmed? true}
```

This is the first complete target experience for `tiny-tui`.

## Stage 8 — Polish, testing, and examples

Purpose: make the first version stable and pleasant.

Add polish only after the core flow works.

Suggested additions:

- Better default styles.
- Configurable cursor marker.
- Configurable selected row style.
- Empty list state.
- Terminal width-aware truncation.
- Basic box width calculation.
- Snapshot-style tests for views.
- Unit tests for list and confirm update functions.
- A small demo app.
- A `tiny-cli` integration example.

Useful examples:

```text
examples/select-project.lg
examples/deps-actions.lg
examples/confirm-delete.lg
```

Testing should focus first on pure functions:

```clojure
(is (= 1 (list/selected-index (first (list/update l :down)))))
(is (= {:type :confirm :value true}
       (second (confirm/update c "y"))))
```

Terminal integration tests can come later.

## First target API

The first public API should be optimized around `select`.

```clojure
(ns my.tool
  (:require [tiny-tui.core :as tui]))

(def result
  (tui/select
    {:title "Select project"
     :items [{:id 1 :name "tiny-cli"}
             {:id 2 :name "tiny-tui"}
             {:id 3 :name "lgx"}]
     :item->text :name
     :actions [{:id :delete
                :key "d"
                :label "delete"
                :destructive? true}]}))

(case (:type result)
  :select
  (println "Selected" (:item result))

  :action
  (println "Action" (:action result) "on" (:item result))

  :cancel
  nil)
```

## Example final V1 experience

```text
Dependencies

› org.clojure/data.json       2.5.1
  lambdaisland/uri            1.19.155
  nooga/let-go-async          git/tag v0.1.0

↑/↓ navigate   enter select   d delete   q quit
```

Delete flow:

```text
┌─────────────────────────────────────────┐
│ Delete dependency?                      │
│                                         │
│ Remove lambdaisland/uri from lgx.edn?   │
│                                         │
│ y delete   n cancel                     │
└─────────────────────────────────────────┘
```

## Future direction

After V1, possible next steps:

- Text input.
- Search/filter list.
- Pagination or viewport.
- Tables.
- Progress/spinner widgets.
- Multi-select.
- Theme map.
- Better Unicode width handling.
- Diff renderer for less flicker.
- WASM/xterm.js compatibility checks.

The key constraint: every new feature should preserve the simple mental model.

```text
state + key message -> new state + event
state -> rendered string
```

That is the core of `tiny-tui`.
