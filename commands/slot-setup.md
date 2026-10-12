---
description: Install the clipboard helpers and slot hotkeys for this machine
---

Set up clipboard handoff on this machine: put the helpers at a stable path, then
point the terminal's hotkeys at them.

The plugin ships the helpers at `${CLAUDE_PLUGIN_ROOT}/scripts`. Resolve that to
an absolute path, but treat it as a **source directory only** — it is
version-numbered (`.../clip-handoff/<version>/...`) and changes on every plugin
update, so a hotkey or config file pointing into it would break silently the
next time the plugin updates. Nothing durable may reference it.

Do this in order:

1. **Verify the clipboard works.** Round-trip a test string:
   `printf 'test\n' | <src>/clip && <src>/clip -o`. If it fails, say which tool
   is missing (pbcopy on macOS, wl-copy or xclip on Linux) and stop — the rest
   is useless without it. On Linux, note that writing to the clipboard from
   inside Claude Code may need the Bash sandbox disabled.

2. **Install the helpers to a stable directory.** Default to `~/.local/bin`,
   creating it if needed; let the user name a different one if they prefer.
   Copy both `clip` and `slot` there, keep them executable, and record the
   choice so the session hook can find them:

       mkdir -p "${XDG_STATE_HOME:-$HOME/.local/state}/clip-handoff"
       printf '%s\n' "<bindir>" > "${XDG_STATE_HOME:-$HOME/.local/state}/clip-handoff/bindir"

   This writes files — say so. If `clip` or `slot` already exist there, mention
   that they are being replaced, and diff them first if they look hand-edited
   rather than installed by this plugin.

3. **Find the terminal config.** kitty is the supported terminal: look for
   `~/.config/kitty/kitty.conf`, and on macOS also
   `~/Library/Preferences/kitty/kitty.conf`. If the user's terminal is not
   kitty, say so plainly and stop: iTerm2 can only send text on a keypress, not
   run a background command, and Terminal.app cannot do either — those need an
   external hotkey tool (Hammerspoon, Raycast, Karabiner), which is outside
   this plugin's scope.

4. **Generate the five mappings**, pointing at the stable directory from step 2,
   never at the plugin's own path:

       map ctrl+alt+1 launch --type=background <bindir>/slot copy 1
       map ctrl+alt+2 launch --type=background <bindir>/slot copy 2
       map ctrl+alt+3 launch --type=background <bindir>/slot copy 3
       map ctrl+alt+4 launch --type=background <bindir>/slot copy 4
       map ctrl+alt+5 launch --type=background <bindir>/slot copy 5

   If `kitty.conf` already has `slot copy` lines pointing somewhere else, show
   the old and new paths and ask before replacing them.

5. **Offer two ways to apply it**, and let the user choose: append the lines to
   `kitty.conf` yourself (a change, not a query — say which), or put them on the
   clipboard to paste. Either way, tell them to reload kitty afterwards:
   `ctrl+shift+F5` on Linux, `cmd+ctrl+,` on macOS.

6. **Tell them how to test it**: queue a harmless command with
   `printf '%s\n' 'echo slot-test' | <bindir>/slot set 1`, then press
   `ctrl+alt+1` and paste. On macOS the notification toast may need permission
   the first time, and if the hotkey does nothing, suggest
   `macos_option_as_alt yes` in `kitty.conf`.

7. **Mention PATH only as a convenience**, if `<bindir>` is not already on it —
   it buys the user `clip` and `slot` as typed commands and nothing else, since
   the agent and the hotkeys both use absolute paths. On macOS `~/.local/bin` is
   not on the default PATH, so this would mean a shell-rc edit; do not do that
   silently and do not imply setup is incomplete without it.
