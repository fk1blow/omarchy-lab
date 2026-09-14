# omarchy-lab

A notebook for Omarchy rough edges found while using the thing: what breaks, why it
breaks, whether it was filed upstream, and what is patched locally in the meantime.

Not a fork and not a plugin. No Omarchy code lives here — findings point at
`/usr/share/omarchy/` paths and at upstream issues.

## Layout

    findings/NNNN-slug.md   one rough edge, one file
    patches/                local fixes not (yet) upstreamable
    INDEX.md                generated table, see ./lab.sh index
    lab.sh                  the only script

## A finding

A markdown file with a `---` header block. Only the header is machine-read; everything
below it is prose for a human.

    ---
    id: 0002
    title: Night light indicator reads "on" after resume while the screen is neutral
    status: noted
    area: nightlight
    upstream: https://github.com/omacom/omarchy/issues/11359
    patch: patches/nightlight-resume.sh
    found: 2026-09-14
    versions: omarchy 4.0.2-1, hyprsunset 0.4.0-3
    ---

Then: **What happens** / **Why** / **Repro** / **Fix**. The Why is the part worth
writing — it is usually three files deep and gone from memory in a week.

### status

| | |
|---|---|
| `noted` | understood, nothing filed |
| `filed` | upstream issue or PR open |
| `accepted` | upstream agrees, not shipped |
| `fixed` | shipped upstream, local patch can go |
| `patched` | living with a local workaround, no upstream path |
| `wontfix` | upstream declined, or not worth it |

## Usage

    ./lab.sh new "night light goes stale after resume"   scaffold the next file
    ./lab.sh list [status]                               one line each
    ./lab.sh index                                       regenerate INDEX.md
    ./lab.sh check                                       ask gh for upstream state

`check` only reports; it never edits a header. Status is a judgement call, so it stays
hand-set.

## Where the local patches actually live

Mostly not here. Machine config is in `~/Projects/dotfiles` (manifest + `plugins.txt`),
and patched plugins are git checkouts under `~/.config/omarchy/plugins/`. `patches/`
here is for standalone scripts and units that have no home yet — a finding's `patch:`
field points wherever the fix really is.
