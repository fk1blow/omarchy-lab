---
id: 0007
title: omarchy font set never shows its restart-terminal notification
status: noted
area: fonts
upstream:
patch:
found: 2026-09-29
versions: omarchy 4.0.4-1
---

## What happens

With Ghostty (or Foot) running, `omarchy font set <font>` is supposed to send a
notification telling you to restart the terminal. It never appears. Instead the
command prints to the terminal:

    2786
    Usage: omarchy-notification-send [--app-name <app-name>] [-g <glyph>] ... <headline> [description] ...

The `2786` is Ghostty's PID. Both lines are noise, and the one thing the user
needed — "restart Ghostty to see the font change" — is lost. By the script's own
account the `SIGUSR2` it sends Ghostty is not enough for a font change, so open
Ghostty windows keep the old font with nothing saying why.

## Why

`omarchy-font-set`, lines 76–82:

```bash
if pgrep -x ghostty; then
  omarchy-notification-send -g  "You must restart Ghostty to see font change"
fi

if pgrep -x foot; then
  omarchy-notification-send -g  "You must restart Foot to see font change"
fi
```

Two bugs:

1. **`-g` takes a value.** In `omarchy-notification-send`, `-g`/`--glyph`
   consumes the next argument as the glyph. The message becomes the glyph, no
   headline is left, and the script prints usage and exits. The double space
   after `-g` suggests a glyph argument was there once and got deleted.
2. **`pgrep` without `--quiet`** prints every matching PID to stdout, which is
   where the stray `2786` comes from.

## Repro

```bash
ghostty &                                   # any running Ghostty
omarchy font set "$(omarchy font current)"  # same font: changes nothing else
# -> a PID and a usage line on stdout, no notification
```

Re-applying the current font is enough; nothing needs to change.

## Fix

**Upstream:** give `-g` its glyph (or drop it) and quiet `pgrep`:

```bash
if pgrep -x --quiet ghostty; then
  omarchy-notification-send -g "󰛖" "You must restart Ghostty to see font change"
fi
```

and the same for Foot.

**Locally:** nothing to patch — restart Ghostty by hand after changing the font.
