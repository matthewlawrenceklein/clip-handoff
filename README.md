# clip-handoff

A Claude Code plugin for the commands the agent *can't* run for you — sudo, a
password prompt, a GUI, another machine. Instead of printing them for you to
select by hand, the agent puts them on your clipboard, and when it needs to see
the output, hands you a command that copies its own output back.

Three pieces:

- **Bare commands on the clipboard.** Ready to run: no leading `!`, no `$ `
  prompt, no explanatory comments mixed into the text you paste.
- **Slots.** Up to five numbered commands queued at once, each loaded onto the
  clipboard with a terminal hotkey, for steps that have to run separately or
  whose output you need to read between them.
- **Output capture.** `… 2>&1 | tee >(pbcopy)` so a command's output both prints
  for you and lands on the clipboard, ready to paste straight back into the
  session.

## Install

    /plugin marketplace add matthewlawrenceklein/clip-handoff
    /plugin install clip-handoff
    /slot-setup

`/slot-setup` is the only manual step: terminal keybindings live in terminal
config, which a plugin cannot write for you. It verifies the clipboard, copies
the two helpers to a stable directory (`~/.local/bin` by default), then
generates five `map` lines pointing there and either appends them to your
`kitty.conf` or hands them to you to paste.

The stable copy matters. Claude Code installs a plugin to a version-numbered
path — `~/.claude/plugins/cache/clip-handoff/clip-handoff/<version>/` — so a
hotkey pointing into the plugin would break silently on the next update.
`/slot-setup` records its install directory in
`${XDG_STATE_HOME:-~/.local/state}/clip-handoff/bindir`, and the session hook
reads it, checks the installed helpers against the ones it shipped with, and
says when an update means re-running `/slot-setup`.

## How it works

A `SessionStart` hook (`hooks/emit-protocol`) renders `hooks/protocol.md` with
this machine's script paths and native clipboard command substituted in, and
emits it as `additionalContext`. That makes the protocol always-on, the way a
`CLAUDE.md` section would be, but installable and updatable.

Two scripts do the work:

- `scripts/clip` — copies stdin to the clipboard, or prints it back with
  `clip -o`. Resolves `pbcopy`/`pbpaste` on macOS, `wl-copy`/`wl-paste` on
  Wayland, `xclip` on X11.
- `scripts/slot` — stores slot text under
  `${XDG_STATE_HOME:-~/.local/state}/clip-slots`, and `slot copy N` puts slot N
  on the clipboard with a desktop toast. Toasts use `notify-send`,
  `terminal-notifier`, or `osascript`, whichever exists; a missing notifier is
  never fatal.

Deliberate choices worth knowing:

- **The agent calls the scripts by absolute path**, handed to it by the hook.
  Nothing has to be on `PATH`, which matters on macOS where `~/.local/bin`
  isn't on the default `PATH`. `PATH` buys you `clip` and `slot` as typed
  commands and nothing more, so `/slot-setup` offers it as a convenience rather
  than a step.
- **Commands handed to you use the short native tool** (`pbcopy`, `wl-copy`) so
  what you paste stays readable, while the agent's own copies use `clip`.
- **Everything is POSIX sh with no interpreter dependency.** A hook written in
  node or python breaks wherever those resolve to an unpinned version-manager
  shim, and the resulting error names the shim rather than the plugin.

## Platform support

| | macOS | Linux (Wayland) | Linux (X11) |
|---|---|---|---|
| clipboard | `pbcopy`/`pbpaste`, built in | `wl-copy`/`wl-paste` | `xclip` |
| toast | `osascript`, built in | `notify-send` | `notify-send` |
| hotkeys | kitty | kitty | kitty |

On macOS nothing needs installing. On Linux, install `wl-clipboard` or `xclip`
if missing, and note that writing to the clipboard from inside Claude Code may
require the Bash sandbox to be off.

`tee >(…)` is process substitution: it needs zsh or bash, not `sh`.

## Limitations

- **kitty only.** iTerm2 can only send text on a keypress, not run a background
  command; Terminal.app can do neither. Those need an external hotkey tool
  (Hammerspoon, Raycast, Karabiner), which this plugin doesn't try to configure.
- **Five slots**, matching five hotkeys. Arbitrary, but more than that is hard
  to keep straight anyway.
- Slot contents are plain files under `$XDG_STATE_HOME`. Don't queue secrets.
