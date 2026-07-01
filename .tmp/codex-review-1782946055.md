The patch adds the requested screen namespace, but the cleanup guarantee can fail during partial terminal initialization, and the new smoke-test command is not runnable as written.

Full review comments:

- [P2] Restore the terminal if setup fails after raw mode — /Users/andrew/Projects/tiny-tui/src/tiny_tui/screen.lg:70-72
  If `init!` succeeds in enabling raw mode but a later setup call such as `term/alternate-screen`, `term/hide-cursor`, `term/clear`, or `term/flush` throws, this macro never enters the `try`, so `shutdown!` is not called and the user's terminal can be left in raw mode. This matters for the advertised safe enter/leave behavior when terminal writes fail during startup.

- [P3] Fix the documented example command — /Users/andrew/Projects/tiny-tui/examples/hello_screen.lg:3-3
  Running the example exactly as documented from the repo root fails with `unable to load namespace tiny-tui.screen` because plain `lg examples/hello_screen.lg` only searches `.` and does not include `src` on the source path. Anyone using this smoke-test command needs a source path-aware invocation such as the project runner or an explicit `-source-paths` value.