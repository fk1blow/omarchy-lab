---
id: 0004
title: Shell UI font is hardcoded to monospace and [font] family is silently dropped
status: filed
area: shell
upstream: https://github.com/omacom/omarchy/issues/11359#issuecomment-5781579461
patch:
found: 2026-09-22
versions: omarchy 4.0.2-1, quickshell 0.3.1-1, fontconfig 2:2.18.3-2
---

## What happens

The whole shell — bar, launcher/menu, notifications, OSD — renders in whatever the
fontconfig alias `monospace` resolves to, i.e. the terminal font. There is no
supported way to give the UI a different family.

Setting `[font] family = "Inter"` in `~/.config/omarchy/shell.toml` is accepted
without complaint and does nothing. Same shape as [0001](0001-bar-icon-tokens-unreachable.md):
a key that looks configurable, is not.

Worth being precise about the blast radius — only the shell is affected. GTK apps are
already on `Adwaita Sans`, and `sans-serif` already resolves to a proportional face.
Omarchy is also the outlier among Quickshell shells here; illogical-impulse, Caelestia
and DankMaterialShell all default to Google Sans Flex with a *separate* mono token,
and Noctalia to Roboto. Every one of them separates UI font from mono font, and
Omarchy is alone in having no key for it.

## Why

Three things have to line up, and all three close.

**1. The family is hardcoded.** `shell/Commons/Style.qml`:

```qml
property string fontFamily: "monospace"
```

Every `qs.Ui` component binds to it.

**2. A string family cannot survive `applyShellValues()`.** The `[font]` branch is
integer-only:

```qml
if (section === "font") {
  var ival = parseInt(raw, 10)
  if (!isFinite(ival)) continue      // "Inter" -> NaN -> dropped
  if (key === "base-size") nextBase = ival
  else fontOut[key] = ival
}
```

So `family` is not merely unimplemented — it is structurally unable to pass through
the one function that reads `[font]`.

**3. The escape hatches do not cover the bar.** `OMARCHY_MENU_FONT` is read into
`menuFontFamily`, but only the menu, polkit, emoji and clipboard surfaces use it.
`omarchy font set` changes the family globally, but it is a *monospace* picker
(`omarchy-font-list` is `fc-list :spacing=100`, and Super+Space → Fonts is the same
code path via `Menu.qml`), and it also rewrites the alacritty/kitty/ghostty/foot
configs — so using it for a UI font puts that font in the terminals. See
[0005](0005-omarchy-font-set-overwrites-the-user-s-entire-font.md).

### A trap for anyone verifying this

`Style.resolvedFontFamily` looks like the way to check what the shell is drawing —
its own comment says to read it "when you want to *display* what's drawing". It is
computed by spawning a child process:

```qml
property Process fcMatchProc: Process {
    command: ["fc-match", "-f", "%{family[0]}", "monospace"]
}
```

Under the `prgname` workaround below, that child's `prgname` is `fc-match`, not
`quickshell`, so the rule does not fire and the property reports the *unoverridden*
answer — contradicting what the shell is actually rendering. Use
`/proc/<pid>/maps` instead; a font file appears there only if the process really
opened it.

## Repro

```bash
printf '\n[font]\nfamily = "Inter"\n' >> ~/.config/omarchy/shell.toml
omarchy restart shell
```

Nothing changes, nothing is logged. (`omarchy restart shell` is required regardless —
already-instantiated widgets keep the old font.)

## Fix

Filed as a [comment on #11359](https://github.com/omacom/omarchy/issues/11359#issuecomment-5781579461)
rather than as its own issue: it is the same drop-on-the-floor pattern in the sibling
branch of the same function, so whoever fixes the `[bar]` keys is two lines away from
this one. See [0001](0001-bar-icon-tokens-unreachable.md), which owns that issue.

**Upstream**, smallest useful change: let `[font] family` through `applyShellValues()`
as a string and bind `Style.fontFamily` to it, defaulting to `monospace` so current
behaviour is unchanged. A fuller fix would separate a UI family from the mono family
the way every other Quickshell shell does.

**Local workaround** — scope the alias by program rather than changing it globally.
fontconfig matches on `prgname` (2.10+), and the shell's `argv[0]` is plain
`quickshell`:

```xml
<match target="pattern">
  <test name="prgname"><string>quickshell</string></test>
  <test name="family" qual="any"><string>monospace</string></test>
  <edit name="family" mode="prepend_first" binding="strong">
    <string>Some Sans</string>
  </edit>
</match>
```

Placed *after* the stock monospace block so the resolved list is
`Some Sans, CaskaydiaMono Nerd Font, monospace, …`, which keeps Caskaydia as
per-glyph fallback for the bar's Nerd Font icons instead of tofu.

Verified working, then **reverted** — not for any fault in the mechanism, but because
none of the candidate faces was an improvement, and at 14px proportional UI faces are
nearly indistinguishable from each other. Note the workaround is erased by
[0005](0005-omarchy-font-set-overwrites-the-user-s-entire-font.md).

Full writeup, including the Qt variable-font trap that ate an afternoon:
`~/Projects/dotfiles/docs/fonts.md`.
