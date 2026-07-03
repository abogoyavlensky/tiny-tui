# AGENTS.md

tiny-tui is a minimal terminal UI library for let-go (a Clojure-dialect
Lisp with a Go VM). Pure widgets (`state + msg -> [state event]`,
`state -> string`) with terminal I/O confined to `tiny-tui.screen` and
`tiny-tui.key/read`. The one sanctioned exception is
`tiny-tui.path/suggest-fn`, which reads directory listings (never the
terminal) for the input widget's optional autocomplete; the completion logic
around it stays pure. Design and stage-by-stage history:
`docs/tiny-tui-initial-design.md` and `docs/plans/2026-07-01-tiny-tui-v1.md`
(its closing summary lists the let-go quirks that shaped the code).

## Commands

```bash
lgx test                        # full test suite (test/**/*_test.lg)
lgx fmt                         # cljfmt; run before committing
lgx run                         # select demo (main.lg)
lgx run examples/<name>.lg      # any example
```

## Verification

`lgx test` alone is not enough for changes touching `screen`, `key`, or
the `run` loop: raw mode, the alternate screen, and cleanup ordering only
exist on a real terminal. Drive the examples end-to-end on a pseudo-TTY
with scripted key bytes and inspect the raw escape output, including the
cancel paths (q, esc, Ctrl-C) and the throwing path. The full harness,
key-byte table, output filters, and known pty gotchas are in
[docs/pty-verification.md](docs/pty-verification.md).

## Gotchas

- No forward references at the top level; order `def`/`defn` bottom-up or
  use `declare`.
- Build control bytes with `(char 27)` at runtime; never write literal
  control characters or unicode escapes for them into `.lg` source.
- let-go `try` without `catch` returns the thrown value instead of
  propagating, and `catch` + rethrow skips `finally`. Use the
  capture/cleanup/rethrow pattern from `screen/with-screen*`.
- Widgets never call handlers or touch the terminal; they return events
  and the caller decides.
