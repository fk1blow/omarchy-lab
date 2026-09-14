---
id: 0001
title: Bar icon-size tokens are defined, read, and then silently dropped
status: filed
area: bar
upstream: https://github.com/omacom/omarchy/issues/11359
patch:
found: 2026-09-11
versions: omarchy 4.0.2-1, quickshell 0.3.1-1
---

## What happens

Bar glyph size is not configurable. `[bar] icon-font`, `icon-canvas`, `icon-slot` and
`status-slot` can be set in `~/.config/omarchy/shell.toml` and are accepted without
complaint — they just do nothing.

## Why

Two compounding causes.

**1. The keys are parsed and thrown away.** `shell/Commons/Style.qml` defines all four
with fallbacks and reads each through `barToken()`, which exists specifically to look
them up in `barOverrides`. But the `[bar]` branch of `applyShellValues()` only ever
writes `scale-with-font`, `size-horizontal` and `size-vertical` into that dict. The
other four always return their hardcoded fallback. `size-horizontal` working is what
makes the dead ones look plausible.

**2. Even wired up, only ~2/3 of widgets would respond.** Bar glyphs size off four
different tokens, three of them typography tokens shared with panels, menus and the
tooltip:

| Widgets | Token | px @ base-size 12 |
|---|---|---|
| tray, audio, network, bluetooth, monitor, power, agents, menu, weather, media, active-window, microphone | `Style.bar.iconFont` | 13 |
| indicators, keyboard-layout, system-update | `Style.font.caption` | 10 |
| clock, workspaces (no explicit `fontSize` → `WidgetButton` default) | `Style.font.body` | 12 |
| tray fallback text | `Style.font.bodySmall` | 11 |

`BarIndicator` is the clearest: it inherits `BarIconButton`, which already defaults
`fontSize` to `Style.bar.iconFont`, then overrides it back to `Style.font.caption`.

## Repro

Set `[bar] icon-font = 20` in `shell.toml`, `omarchy restart shell`. Nothing changes,
nothing is logged.

## Fix

Upstream: populate the four keys in `applyShellValues()`, and point every bar widget at
`Style.bar.iconFont`.

Local workaround, in use: eight `dragos.*` widget forks under
`~/.config/omarchy/plugins/`, each existing only to point one widget at
`Style.bar.iconFont`. Tracked as plain copies in the dotfiles `manifest`. The only real
lever left is `[font] base-size`, which `barToken()` multiplies by `base-size / 12` —
so every other `[font]` token is pinned to its base-size-12 value to keep the bump
confined to the bar, and `size-horizontal` is pre-divided to cancel it.
