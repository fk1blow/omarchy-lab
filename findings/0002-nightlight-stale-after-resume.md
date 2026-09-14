---
id: 0002
title: Night light indicator reads "on" after resume while the screen is neutral
status: noted
area: nightlight
upstream:
patch:
found: 2026-09-14
versions: omarchy 4.0.2-1, hyprsunset 0.4.0-3, hyprland 0.56.2-1
---

## What happens

Wake the machine, the bar's night light icon is lit, the screen is not warm. Toggling
off then on restores the tint.

## Why

Nothing in the chain looks at the gamma actually on the output. The indicator reads
hyprsunset's own in-memory integer:

    NightLight.qml  →  nightlight Service.qml  →  hyprctl hyprsunset temperature  →  hyprsunset

`Service.qml` defines on as `temperature < 6000` (`NightlightModel.js`), from shelling
out to `hyprctl hyprsunset temperature`. Night is 4000K, day 6500K. Setting it just
runs `hyprctl hyprsunset temperature <n>`.

hyprsunset is long-lived (started by `omarchy-toggle-nightlight` via `uwsm-app`, scope
`app-Hyprland-hyprsunset-*`, not autostarted). On suspend/resume — or DPMS off, which
`idle.screensaver` triggers — the output is reinitialised and the gamma LUT resets to
linear. hyprsunset is never notified and does not re-push. Its counter still says 4000,
so `hyprctl` says 4000, so the icon stays lit.

One tap isn't enough because re-sending the same value looks like a no-op; off→on sends
6500 then 4000, two genuinely new values, so a fresh LUT gets pushed.

Same fragility is already acknowledged upstream in `omarchy-toggle-nightlight`:

    # A freshly-started hyprsunset applies its default temperature at the end of
    # its boot, overriding anything set before then — so resend until it sticks
    for _ in {1..10}; do ...

Cold boot behaves differently: hyprsunset isn't running, `hyprctl` fails, the service
sets `temperature = null`, and the icon correctly reads off. The bug is specific to
resume/DPMS.

## Repro

    hyprctl dispatch dpms off && sleep 2 && hyprctl dispatch dpms on

Tint gone, `omarchy toggle nightlight --status` still reports `{"enabled":true,"temperature":4000}`.

## Fix

Nothing upstream reapplies it — no nightlight touch anywhere in `omarchy/bin/` beyond
the toggle, and the only sleep-related unit is `omarchy-sleep-lock.service`, which just
locks.

Candidates, in rough order of how much they fix:

1. **hyprsunset** re-applies on output re-enable. Real fix, upstream-hyprland side.
2. **Omarchy** ships a resume hook: a `systemd --user` unit `WantedBy=suspend.target`
   that reads the current temperature, bounces it through a different value, re-sends.
   Doesn't cover DPMS.
3. **Omarchy shell** stops trusting `hyprctl` as the source of truth and re-applies on
   the compositor's monitor-added / DPMS events (socket2). Covers both, and fixes the
   lying indicator rather than just the tint.

Local workaround: not written yet.
