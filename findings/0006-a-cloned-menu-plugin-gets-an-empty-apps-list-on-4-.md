---
id: 0006
title: A cloned menu plugin gets an empty Apps list on 4.0.4
status: noted
area: shell/plugins
upstream:
patch:
found: 2026-09-29
versions: omarchy 4.0.4-1, quickshell 0.3.1-1
---

## What happens

After `omarchy update` from 4.0.2 to 4.0.4, Super+Space → Apps shows
**"Nothing here yet"**. Every other part of the menu works — the root list,
Learn, Style, Setup — only the application list is empty.

The trigger is a clone of the menu made with `omarchy plugin clone omarchy.menu`
(here, to change the bar button's `horizontalMargin` and nothing else — its
`Menu.qml` and `MenuModel.js` were byte-identical to stock). The clone had been
in daily use on 4.0.2 without problems.

Nothing is logged. No QML error, no warning from the shell — the list is just
empty, which reads as "the update uninstalled my apps".

## Why

Cloning a plugin whose manifest has a non-widget kind does more than copy a bar
button. `PluginRegistry` adds the source to `disabledPlugins` (and records it in
`cloneSourceRestores`), so the clone becomes *the* menu: `omarchy-menu toggle`
resolves `omarchy.menu` to the enabled clone.

The clone is not first-party, so it does not get the host shell. 4.0.4 gives
third-party plugins a scoped `PluginShellApi` instead
(`shell.qml`, `pluginShellFor()` → `createScopedPluginShell()`), and a
menu-kind plugin reaches the app list through that API's detached
`PluginAppLibraryApi`. `Menu.qml` reads it as
`root.shell ? root.shell.appLibrary : null`, and `mergeAppRows()` returns early
when that is `null`.

Instrumenting the clone with `onShellChanged` shows the API arriving and then
being destroyed straight after:

    shellChanged -> PluginShellApi_QMLTYPE_20(0x71baa7c24dc0)  at t
    shellChanged -> null                                       at t + 77 ms

The menu is `keepLoaded`, so its Loader never fires `onLoaded` again and nothing
re-assigns `shell`. Apps stays empty for the life of the process, across any
number of `omarchy restart shell`s.

Only two places destroy a scoped API: `createScopedPluginShell()` when the cached
capability profile differs, and `prunePluginApis()` when a plugin is inactive or
its profile no longer matches. For this clone the profile is the same from every
call site (`own-service|no-bar|no-indicators|menu`), which leaves the prune's
`active` test — `pluginRegistry.isEnabled(id)`. For a third-party plugin that is
`findEntryLocation(config, id).found`, i.e. it depends on `shell.json` having been
read. **Likely, not confirmed:** a prune runs during startup before the config is
loaded, sees the clone as disabled, and revokes its API. Confirming it needs
logging inside `/usr/share/omarchy/shell/shell.qml`, which was not done.

Ruled out on the way, so nobody re-checks them: the apps are installed;
Quickshell's `DesktopEntries` sees all ~80 entries; `hidden-entries.sh` hides 28;
the `launcher.hides` list is stock; `qt6-base` 6.11.2-3 vs -2 behaves the same;
the user `omarchy-menu.jsonc` extension is empty.

## Repro

```bash
omarchy plugin clone omarchy.menu        # any id, e.g. me.menu
omarchy restart shell
omarchy menu toggle apps                 # "Nothing here yet"
```

To see the revocation, add to the clone's `Menu.qml` after `property var shell: null`:

```qml
onShellChanged: console.warn("shellChanged -> " + shell + " at " + Date.now())
```

then `omarchy restart shell` and read `quickshell log --pid <pid>`.

## Fix

**Locally:** removed the clone with `omarchy plugin remove <clone-id>`. That
disables it through the shell, moves the folder to a `.bak`, and restores
`omarchy.menu` into the clone's bar slot — Apps works immediately after
`omarchy restart shell`.

**Rule of thumb until upstream changes:** clone only plugins whose kinds are
`bar-widget` alone. A clone of anything that is also `menu`, `panel` or
`overlay` takes over the stock plugin's runtime role, and on 4.0.4 it does so
with less access than the original had.

**Upstream, if it is filed:** either re-assign `shell` to live panel items after
`prunePluginApis()` revokes one (the bar path already re-syncs through
`syncPluginApis()`; the panel Loader path does not), or have `prunePluginApis()`
skip revocation until the shell config has loaded. Separately, `mergeAppRows()`
returning silently on a missing `appLibrary` is what made this undiagnosable —
a single `console.warn` there would have pointed straight at it.
