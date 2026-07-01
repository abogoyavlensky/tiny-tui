The patch passes the current tests, but the new generic list/select flow cannot select falsey items and reports cancellation instead. This is a functional edge-case bug in the introduced widget behavior.

Review comment:

- [P2] Allow selecting falsey list items — /Users/andrew/Projects/tiny-tui/src/tiny_tui/list.lg:31-32
  When the selected item is `nil` or `false`, `when-let` treats it as absent, so pressing Enter emits no select event; through `tui/select` this keeps looping until EOF and then returns `{:type :cancel}` even though the list is non-empty. Since the widget accepts arbitrary `:items`, the selection check should be based on list non-emptiness/index validity rather than item truthiness.