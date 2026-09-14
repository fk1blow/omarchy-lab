---
id: 0002
title: Night light indicator reads "on" after resume while the screen is neutral
status: noted
area: nightlight
upstream:
patch:
found: 2026-09-14
versions: omarchy 4.0.2-1, hyprsunset 0.4.0-3, hyprland 0.56.2-1, aquamarine 0.14.0-2
---

## What happens

Wake the machine, the bar's night light icon is lit, the screen is not warm. Toggling
the widget off then on restores the tint.

## Why

Two separate things, and only the first is in doubt.

### The tint is lost

hyprsunset applies a colour transform matrix through `hyprland-ctm-control-v1`, per
output — not a gamma LUT. Something on the resume path drops that CTM without
hyprsunset knowing.

The usual suspects are already fixed on these versions, so this needs pinning down
before anything is written:

- hyprsunset **does** handle hotplug. `src/Hyprsunset.cpp:130-141`: a new `wl_output`
  global is bound and, if already initialised, gets the CTM applied instantly and
  committed; `:146` drops it on global-remove. So an output that is destroyed and
  recreated should come back tinted.
- aquamarine **does** re-send the CTM on modeset. PR #256 "drm: re-send ctm blob on
  modeset" merged 2026-03-13, shipped in v0.14.0 (2026-07-27); installed is 0.14.0-2.
  Plus #297 clear stale color state, #303 init ctm to identity, #320 avoid redundant
  identity modesets.
- hyprsunset#65 "Filter is disabled after toggling monitor DPMS" was closed 2026-08-07
  as fixed by exactly that aquamarine work.

So the DPMS path is *believed* fixed here and suspend/resume is the untested one. On
resume the journal shows amdgpu fully reinitialising both cards (`PCIE GART enabled`,
`DMUB hardware initialized`), which is a heavier path than a DPMS toggle: plausibly the
output survives as the same wl_output global (so hyprsunset's hotplug branch never
fires) while the kernel-side CTM is gone and aquamarine's `NEEDS_RECONFIG` re-send
doesn't cover it. Unverified.

Also unruled-out: hyprsunset's poll thread misbehaving across suspend, cf. hyprsunset#47
(CPU spike after sleep) and #55 (scheduling delayed after suspend).

### The indicator lies about it

Independent of the above, and certainly a bug. Nothing in the chain looks at what is on
the output:

    NightLight.qml  →  nightlight Service.qml  →  hyprctl hyprsunset temperature  →  hyprsunset

`Service.qml` defines on as `temperature < 6000` (`NightlightModel.js`), read by
shelling out to `hyprctl hyprsunset temperature`. That returns hyprsunset's own
in-memory `KELVIN`, which is still 4000 whether or not the CTM survived. Night is
4000K, day 6500K.

The widget also has no re-apply: `NightLight.qml`'s only action is
`setNightlight(!root.active)`. That is why the fix is two taps rather than one —
`reload()` re-applies unconditionally, so a single `hyprctl hyprsunset temperature 4000`
would do it, but the UI cannot send that.

Cold boot behaves correctly: hyprsunset isn't running (it is started by
`omarchy-toggle-nightlight` via `uwsm-app`, not autostarted), `hyprctl` fails, the
service sets `temperature = null`, icon reads off.

## Repro

Suspend, resume, look at the screen. Confirmed on omarchy 4.0.2-1 / hyprsunset 0.4.0-3 /
hyprland 0.56.2-1 / aquamarine 0.14.0-2 — i.e. *with* the aquamarine CTM-on-modeset fix
already installed, so this is not the DPMS bug from hyprsunset#65 and no update helps.

Still unanswered, because it needs a suspend to test: whether a single
`hyprctl hyprsunset temperature 4000` restores the tint, or only a change of value does.
Toggling the widget always sends a different value, so using the widget never answers it.

## Plan

### 1. Find out what actually breaks — nothing gets written before this

- **Is DPMS still broken?** `hyprctl dispatch dpms off && sleep 2 && hyprctl dispatch dpms on`.
  Expectation on aquamarine 0.14: tint survives. If it doesn't, hyprsunset#65 has
  regressed and that is the whole story — reopen it upstream, done.
- **Is suspend broken?** Confirm across a real suspend, not by memory.
- **Does one re-send fix it?** With the tint gone, `hyprctl hyprsunset temperature 4000`
  alone. Source says yes. If it does *not*, hyprsunset's wayland connection is the
  problem, not the CTM, and the whole plan changes.
- **What does hyprsunset see?** Restart it with `--verbose` and read the scope journal
  across a suspend — it logs every CTM calculation, output bind and global-remove.
  `systemctl --user status 'app-Hyprland-hyprsunset-*.scope'` for the unit name.
- **Does the output global survive?** `wayland-info` or a socket2 tail
  (`socat -U - UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`)
  across the suspend — `monitorremoved`/`monitoradded` tells us whether hyprsunset's
  hotplug branch had any chance to fire.

Record the answers here before moving on.

### 2. Local workaround — not written, and not until step 1 is answered

A resume-edge re-send is the obvious shape: `suspend.target` does not exist in the user
manager (`LoadState=not-found`), so there is no unit to hook and it would have to watch
logind's `PrepareForSleep` the way `omarchy-system-sleep-monitor` already does to lock
before suspend, acting on `boolean false` instead of `true`.

Written and installed once on 2026-09-14, then removed the same evening. It was premature
twice over: the mechanism is still unnamed, so there was nothing to say the re-send was
the right lever; and it only fires on resume, so a warm screen the next morning would not
distinguish the workaround working from the bug not happening. Step 1 first.

### 3. The real fix, wherever step 1 points

- **aquamarine / Hyprland** — CTM not restored on the resume modeset path. Most likely,
  given #256 fixed the DPMS case in the same place. Fix goes next to that code.
- **hyprsunset** — if the output global survives resume, hyprsunset can't notice via the
  registry and needs another trigger. Note #7 is an open request for gammastep-style
  hooks, which would be one.
- **hyprsunset** — if a single re-send does not restore the tint, the wayland connection
  or poll thread is broken across suspend. Different bug, higher value.

### 4. Fix the lying indicator — worth doing regardless

Omarchy-side, and true even after step 3 lands upstream, because trusting hyprsunset's
integer is wrong on principle.

- Give `Service.qml` a `reapply()` that re-sends the current temperature, and expose it
  on the existing `IpcHandler` alongside `enable`/`disable`/`toggle`.
- Have the service re-apply on the compositor's own monitor events rather than only on
  user action. The shell already talks to socket2 — `Quickshell.Hyprland` is imported in
  `Bar.qml`, `Workspaces.qml`, `KeyboardLayout.qml`, `PopupCard.qml` and the idle
  service — so there is precedent and no new dependency.
- Then a middle-click or long-press on the indicator that re-applies without flipping,
  so the manual escape hatch is one action instead of two.

Upstreamable to Omarchy on its own, without waiting on the Hyprland side.
