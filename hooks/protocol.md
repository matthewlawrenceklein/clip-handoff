# Command handoff protocol

Any command the user has to run in their own terminal (sudo, a password prompt, a
GUI, another machine) goes onto their clipboard, not just into the reply. Never
make them select text by hand.

Helpers, always callable at these absolute paths:

    @@BIN@@/clip
        copy stdin to the clipboard; `clip -o` prints it back
    @@BIN@@/slot
        queue up to 5 numbered commands; the user loads slot N with the
        terminal hotkey for N

In commands you hand over for the user to run, write the clipboard tool as
`@@COPY@@` — it is this machine's native command, so it stays short and readable
for them. Use the absolute `clip` path above only for copies you make yourself.

## Rules

- Copy bare, ready-to-run commands only: no leading `!`, no `$ ` prompt, no
  explanatory comments inside the copied text. In zsh a leading `!` negates the
  exit status, so `! a && b` silently skips `b`.
- Say in the reply that it is on the clipboard, and how many lines or steps it
  holds.
- One step, or several that can run back-to-back in one window: ONE clipboard
  block.

      printf '%s\n' 'cmd one' 'cmd two' | @@BIN@@/clip

  Read it back with `@@BIN@@/clip -o` to confirm the copy landed.
- Steps that must run separately (different windows, or you need to read the
  output of one before giving the next): use slots.

      @@BIN@@/slot clear
      printf '%s\n' 'cmd' | @@BIN@@/slot set 1      # then 2, 3, ...
      @@BIN@@/slot copy 1

  List them in the reply as "Slot 1: ...", "Slot 2: ..." saying what each does
  and the order to run them in.
- If you expect to read the output, make the command capture it. Append
  `2>&1 | tee >(@@COPY@@)` so the output both prints for them and lands on the
  clipboard, ready to paste straight back. For a multi-command step, wrap the
  whole thing:

      { cmd one
        cmd two
      } 2>&1 | tee >(@@COPY@@)

  Notes: if the step uses sudo, give `sudo -v` its own first line so the
  password prompt happens before anything is piped. The pipeline's exit status
  is tee's, not the command's, so do not chain `&& something` off it and do not
  trust `$?` — any failure will be visible in the captured text instead. This
  overwrites the clipboard, which is fine alongside slots: the user reloads the
  next step with its hotkey. Skip the capture for commands whose output you do
  not need back.
- For every block and every slot, say explicitly whether it is read-only or
  changes something. Never let the user paste a config change thinking it was a
  query.
- Never run sudo yourself; hand it over. Never print passwords or tokens: have
  the user enter secrets with `read -rs` into a file with mode 600.
