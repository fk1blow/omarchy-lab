---
id: 0005
title: omarchy font set overwrites the user's entire fonts.conf
status: filed
area: fonts
upstream: https://github.com/omacom/omarchy/issues/12927
patch:
found: 2026-09-22
versions: omarchy 4.0.2-1, fontconfig 2:2.18.3-2
---

## What happens

`omarchy font set <name>` rewrites `~/.config/fontconfig/fonts.conf` from scratch.
Any other fontconfig rule the user has in that file is deleted — no merge, no backup,
no warning. Picking a font in Super+Space → Fonts does the same thing, since it is
the same code path.

`fonts.conf` is the *user's* top-level fontconfig file, not Omarchy's. Anything
normally kept there — font substitutions, per-family aliases, hinting or rendering
preferences, a `prgname` scope — is silently destroyed the next time the terminal
font is changed.

This is the classic "worked until I touched something unrelated" failure. The
connection between *changing your terminal font* and *losing your font rendering
preferences* is not discoverable, and nothing points at it afterwards.

## Why

`omarchy-font-set` writes the file with a heredoc:

```bash
fontconfig_file="$HOME/.config/fontconfig/fonts.conf"
mkdir -p "$(dirname "$fontconfig_file")"
cat >"$fontconfig_file" <<XML
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <match target="pattern">
    ...
  </match>
</fontconfig>
XML
```

`cat >` truncates. The script never reads the existing file, so it cannot preserve
anything, and there is no `.bak`.

The comment above that block is right that fontconfig is the correct source of truth —
it is the *file choice* that is wrong. fontconfig's own convention for exactly this
situation is a drop-in directory: `conf.d/` entries are merged in numeric order, which
is how a program contributes one rule without owning the user's whole config.

## Repro

```bash
cat ~/.config/fontconfig/fonts.conf          # note contents
# add any rule of your own to that file, then:
omarchy font set "JetBrainsMono Nerd Font"
cat ~/.config/fontconfig/fonts.conf          # your rule is gone
```

Nothing is logged and nothing is backed up.

Confirmed on 4.0.2-1 rather than inferred from the source: injecting a
`<match target="font">` rgba rule into `fonts.conf`, then running
`omarchy-font-set` with the *same* font already active, removed the rule and created
no `.bak`. Re-running with an unchanged font is enough — the destruction does not
require the font to actually change.

## Fix

Filed as [omacom/omarchy#12927](https://github.com/omacom/omarchy/issues/12927).
Leave it there — no follow-up comments, no edits, no closing it out. Whether and
when upstream responds is upstream's call.

**Upstream:** write to `~/.config/fontconfig/conf.d/50-omarchy-monospace.conf` instead
of owning `fonts.conf`. That is what the drop-in directory is for, it composes with
whatever else the user has, and it needs no merge logic — the file is still written
wholesale, it just stops being a file that belongs to someone else. A migration can
drop the `monospace` block from `fonts.conf` if it is the only thing there.

Omarchy already uses this convention at the system level -- the package ships
`/usr/share/fontconfig/conf.avail/50-omarchy.conf`, linked into `/etc/fonts/conf.d/`,
which is where the `monospace -> JetBrainsMono Nerd Font` assignment lives. The load
order also works out, which matters because the user-level rule has to override that
package one:

    /etc/fonts/conf.d/50-omarchy.conf   package drop-in (assigns monospace)
    /etc/fonts/conf.d/50-user.conf      sorts after "50-omarchy", and includes:
        ~/.config/fontconfig/conf.d/*       <- proposed location
        ~/.config/fontconfig/fonts.conf     <- user's own file, still loads last

So a drop-in is applied after the package rule and before the user's own `fonts.conf`,
meaning a user rule would outrank Omarchy's instead of being deleted by it.

Failing that, at minimum back the file up the way `omarchy refresh` does, rather than
truncating in place.

**Locally:** nothing to patch. The rule at risk was the `prgname` scope from
[0004](0004-shell-ui-font-is-hardcoded-to-monospace-and-font-f.md), which has since
been reverted, so `fonts.conf` now holds only the stock block. Worth remembering if
that workaround is ever reinstated: keep the real copy in the dotfiles repo and
re-apply with `./restore.sh omarchy` after any `omarchy font set`.
