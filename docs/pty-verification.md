# Verifying tiny-tui under a pseudo-TTY

Unit tests cover every pure function, and whole `select`/`confirm` flows run
headlessly through `run`'s testing hooks (`:read-key-fn`, `:render-fn`,
`:screen false`). What they cannot cover: raw mode, the alternate screen,
cursor visibility, and cleanup ordering. `term/read-key` and `term/write`
work on plain pipes, but `term/raw-mode!` and `term/size` need a real
terminal. The `script` utility provides one, so all of this is verifiable
without a human at a keyboard.

## The harness

```bash
printf '<key bytes>' | timeout 15 script -qec "lgx run examples/counter.lg" /dev/null
```

- `script -qec CMD /dev/null` runs CMD on a fresh pseudo-TTY and discards
  the typescript file.
- `printf` supplies keystrokes as raw bytes. When they run out, stdin
  closes; a still-running app sees end-of-input, which tiny-tui treats as
  cancel/quit.
- `timeout 15` kills a hung run (a missed key otherwise blocks forever;
  the run then ends with "Session terminated, killing shell").

Key bytes:

| Key | Bytes |
|---|---|
| up / down / right / left | `\033[A` `\033[B` `\033[C` `\033[D` |
| enter | `\r` |
| esc | `\033` |
| Ctrl-C | `\003` |
| word left / right (macOS Option) | `\033b` / `\033f` |
| word left / right (CSI: Alt / Ctrl + arrow) | `\033[1;3D` `\033[1;5D` / `\033[1;3C` `\033[1;5C` |
| delete word (Alt+d / Ctrl+Delete) | `\033d` / `\033[3;5~` |
| backspace word (Option+Backspace / Ctrl-W) | `\033\177` / `\027` |
| printable (`q`, `d`, `y`, `n`) | the character itself |

An `ESC` immediately followed by another byte in the same `printf` is read as
an Alt/Option chord (that's how Option+Left etc. work), *not* as a lone esc. So
to test that esc cancels, either end the `printf` at `\033` (nothing pending)
or split the following key into a later `printf` after a short `sleep`.
Likewise, gate the whole keystroke stream behind a leading `sleep` so raw mode
is active before any byte arrives (a raw `\177` sent in cooked mode line-erases
instead).

Example: drive the deps demo down one row, press `d`, confirm with `y`:

```bash
printf '\033[Bdy' | timeout 15 script -qec "lgx run examples/deps_actions.lg" /dev/null
```

## Reading the output

Strip escape sequences to assert on rendered content and final prints:

```bash
... | sed -e "s/$(printf '\033')\[[0-9;?]*[a-zA-Z]//g" | tr -d '\r' | tail -5
```

Keep them to assert on terminal control itself:

```bash
... | od -c | tail -20
... | grep -a -c "$(printf '\033')\[?1049l"   # count main-screen restores
```

A healthy run shows, in order: `ESC[?1049h` (alternate screen on) and
`ESC[?25l` (hide cursor) at the start; frames as `ESC[H` + lines each ending
in `ESC[K`, joined by `\r\n`, ending with `ESC[J`; then `ESC[0m`,
`ESC[?25h`, `ESC[?1049l` exactly once at the end. Also verify the failure
path: a body that throws inside `screen/with-screen` must still emit the
restore sequence and then surface the error (see the "kaboom" pattern in
the V1 plan summary).

### Inline mode

Inline mode (`:inline? true`) never enters the alternate screen, so verify by
what's *absent* as much as what's present. A healthy inline run
(`examples/inline_select.lg`) shows `ESC[?25l` (hide cursor) near the start and
`ESC[?25h` (show cursor) near the end, but **no** `ESC[?1049h`/`ESC[?1049l`.
Frames begin with `\r` (not `ESC[H`) and end with a cursor-up (`ESC[<n>A`)
returning to the frame's top-left; the final teardown is a lone `ESC[J` that
erases the widget, after which the app's own `println` prints where the widget
sat. Assert the alternate screen is never touched:

```bash
printf '\033[B\r' | timeout 15 script -qec "lgx run examples/inline_select.lg" /dev/null \
  | grep -a -c "$(printf '\033')\[?1049h"   # expect 0
```

#### Inline sessions

`tui/with-inline-session` runs a whole flow of widgets in one raw-mode span, so
the terminal is taken over once for the flow rather than once per widget. A
healthy session run (`examples/inline_flow.lg`) shows exactly **one**
`ESC[?25l` / `ESC[?25h` pair around the entire flow (count each — more than one
means a widget entered raw mode on its own), still **zero** `ESC[?1049h`, and a
`clear-below` (`ESC[J`) as each widget erases before the next renders in its
place. The final `println` lands where the widgets sat.

```bash
raw=$({ sleep 1.3; printf 'r!\r\ry'; sleep 0.5; } \
       | timeout 20 script -qec "lgx run examples/inline_flow.lg" /dev/null)
printf '%s' "$raw" | grep -a -c "$(printf '\033')\[?25l"    # expect 1 (hide once)
printf '%s' "$raw" | grep -a -c "$(printf '\033')\[?25h"    # expect 1 (show once)
printf '%s' "$raw" | grep -a -c "$(printf '\033')\[?1049"   # expect 0
```

Also verify the cancel paths (esc inside the nested prompt leaves the outer
select running; q / Ctrl-C at each stage) and the throwing path — a handler
that throws inside the session must still emit `ESC[?25h` and restore cooked
mode before the error surfaces, exactly like `with-screen`.

## Gotchas learned while building V1

- Bytes sent before the app enables raw mode pass through the pty in
  cooked mode, where ICRNL turns `\r` into `\n`. This is why
  `tiny-tui.key` maps both CR and LF to `:enter`; do not remove that
  mapping to "clean up" the table.
- The first byte may echo before raw mode disables echo, leaving a stray
  `x` or `^[` at the start of captured output. Ignore it.
- `term/size` reports `[0 0]` on the `script` pty. `core/run` falls back
  to `[80 24]`; any new layout code must tolerate nonsense sizes rather
  than assume the fallback happened.
- let-go `try` quirks (documented on `screen/with-screen*`): a `try`
  without `catch` returns the thrown value instead of propagating, and a
  `catch` that rethrows skips the `finally`. Never rely on `finally` for
  cleanup that must always run; use the capture/cleanup/rethrow pattern.

## Order of verification

1. `lgx test` for widgets, views, and key parsing (pure functions).
2. Headless flow tests through `run`'s hooks with scripted messages.
3. PTY runs of the examples for anything touching `screen`, `key`, or the
   run loop, including at least one cancel path and the throwing path.
4. A human look in a real terminal before calling a release done.
