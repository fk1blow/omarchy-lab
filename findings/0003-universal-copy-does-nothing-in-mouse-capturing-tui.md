---
id: 0003
title: Universal copy silently does nothing inside a mouse-capturing TUI
status: patched
area: clipboard
upstream:
patch:
found: 2026-09-15
versions: omarchy 4.0.2-1, ghostty 1.3.1-2, hyprland 0.56.2-1, claude-code 2.1.263
---

## What happens

`SUPER+C` is bound as **"Universal copy"** and works everywhere — until the focused
terminal is running a full-screen TUI that captures the mouse. Then it does nothing at
all: no copy, no error, no toast, no log line. `SUPER+V` is unaffected.

Found with Claude Code in ghostty, but nothing here is specific to either; any TUI that
turns on mouse tracking will do it, and any Omarchy terminal will.

The failure is silent across three layers at once, which is what makes it expensive.
There is no way to tell from the outside whether Hyprland failed to fire the bind,
ghostty failed to receive it, or the TUI ignored it — and the natural first guess,
that the compositor is eating the key, is wrong.

## Why

`SUPER+C` is not a copy. It is a chord forwarder that branches on whether the focused
window is tagged `terminal`, in
`/usr/share/omarchy/default/hypr/bindings/clipboard.lua`:

```lua
o.bind("SUPER + C", "Universal copy",  universal_clipboard_shortcut("CTRL", "C", "CTRL", "Insert"))
o.bind("SUPER + V", "Universal paste", universal_clipboard_shortcut("CTRL", "V", "SHIFT", "Insert"))
```

In a terminal it sends `CTRL+Insert`, because plain `CTRL+C` is SIGINT. Omarchy ships
the matching half in its own ghostty config,
`/usr/share/omarchy/config/ghostty/config`, overriding two of ghostty's defaults:

```
keybind = shift+insert=paste_from_clipboard     # ghostty default: paste_from_selection
keybind = control+insert=copy_to_clipboard      # ghostty default: copy_to_clipboard:mixed
```

Both halves are correct and deliberate. The hole is what `copy_to_clipboard` copies:
**ghostty's own selection.** A TUI that captures the mouse takes drag events for
itself, so the selection lives in the application and ghostty has none. Copying
nothing is indistinguishable from the key not working.

Ghostty's `performable:` keybind prefix explains the silence exactly:

> Only consume the input if the action is able to be performed. For example, the
> `copy_to_clipboard` action will only consume the input if there is a selection to
> copy. **If there is no selection, Ghostty behaves as if the keybind was not set.**

Paste has no equivalent failure: `paste_from_clipboard` is always performable, so
ghostty consumes `SHIFT+Insert` and writes to the pty whatever is focused. That
asymmetry is why `SUPER+V` keeps working and only copy goes dead — and it actively
misleads, because "paste works, so the binding must be fine" is the wrong conclusion.

## Repro

1. `ghostty`, then run any mouse-capturing TUI (Claude Code, or anything using the
   alternate screen with mouse tracking on).
2. Select text inside the TUI with the mouse.
3. `SUPER+C`. Nothing is copied; the clipboard keeps its previous contents.
4. Quit the TUI, select text at the shell prompt, `SUPER+C`. Works.

Diagnostics that all look healthy while it fails: the bind is listed by
`omarchy menu keybindings --print`, the window carries the tag
(`hyprctl clients -j` → `tags: [..., 'terminal*']`), and ghostty has the keybind
(`ghostty +list-keybinds | grep insert`).

## Fix

`CTRL+SHIFT+C` works in both worlds, where `CTRL+Insert` only works in one. Ghostty's
`ctrl+shift+c` is left at its default and is `performable:`, so with no ghostty
selection it acts as if unbound and the key reaches the application — which is what a
TUI needs — while at a shell prompt ghostty owns the selection and copies as before.

Local patch, in use, in `~/.config/hypr/bindings.lua`:

```lua
hl.unbind("SUPER + C")

o.bind("SUPER + C", "Universal copy", function()
  if mac_active_window_is_terminal() then
    mac_send_once("CTRL SHIFT", "C")
  else
    mac_send_once("CTRL", "C")
  end
end)
```

`mac_send_once` and `mac_active_window_is_terminal` are local copies of the
`send_shortcut_once` / `active_window_is_terminal` helpers from Omarchy's
`clipboard.lua`, which are file-local and cannot be imported.

Upstream would be the same one-line change to the terminal branch of
`universal_clipboard_shortcut`. **Deliberately not filed.**

Worth re-checking after an `omarchy update`: this overrides a stock binding, so if
upstream ever changes the terminal chord the unbind and rebind should be revisited.
Nothing breaks loudly if it does — it would just go quiet again, the same way.

## Not the cause

Ruled out along the way, each of which looked plausible:

- **Hyprland eating the key.** It fires correctly; `SUPER+C` works in the same ghostty
  window at a shell prompt.
- **The held `SUPER` leaking into the injected chord.** Real hazard in general, but
  `send_key_state` with explicit mods does not leak it — same reason Omarchy uses it
  rather than `wtype`.
- **The terminal tag missing.** Ghostty is matched by
  `default/hypr/apps/terminals.lua` and carries `terminal*`.
- **The kitty keyboard protocol.** A reasonable suspect since the TUI enables it and
  that is a known source of swallowed chords, but not what is happening here.
