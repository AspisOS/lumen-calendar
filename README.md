# lumen-calendar

The calendar application for **AspisOS**, a capability-based,
no-ambient-authority operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

lumen-calendar is a month-view calendar with per-day notes. It speaks the
[lumen](https://github.com/AspisOS/lumen) external-window protocol, is a
standalone component of the Lumen desktop, and is distributed as a
[herald](https://github.com/AspisOS/AspisOS) package installed as an `/apps`
bundle.

## Role in the system

- An ordinary Lumen client: it connects to the compositor and draws a single
  476x430 window (its descriptor's display name is **Calendar**) showing the
  current month as a 7x6 grid. Today is highlighted, weekend labels are accented,
  a live clock/date sits in the header, and days with a saved note carry a marker
  dot.
- Per-day notes are persisted to `$HOME/.calendar` (defaulting to `/root` when
  `$HOME` is unset), one `YYYY-MM-DD note` line per day; saving an empty note
  removes the day's line.
- Date math is self-contained (leap-year handling, Sakamoto's day-of-week
  algorithm); the header clock uses `glyph_theme_tz_offset()` to render local
  time.
- Keys: arrows move the selection, PgUp/PgDn (or `[` / `]`) change month, typing
  edits the selected day's note, Enter saves, Esc reverts the edit, Q/close
  quits. The header arrows and day cells are also clickable.

## Capabilities

lumen-calendar's cap policy (`pkg/etc/aegis/caps.d/calendar`) is the baseline
desktop-app profile:

```
service
```

It carries no elevated capabilities of its own; reading and writing
`$HOME/.calendar` happens under the `service` profile granted to a Lumen client.

Because its herald package id (`lumen-calendar`) differs from the bundle/exec
name (`calendar`) and it installs a binary plus a cap policy and an app
descriptor across `/apps` and `/etc`, it is a `class=system` package:
first-party and signature-trusted, installed verbatim by herald.

## Building

lumen-calendar fetches a pinned [glyph](https://github.com/AspisOS/glyph)
toolkit artifact (the GUI libraries it links) and builds against it, then packs
a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-calendar.hpkg` (a `class=system` herald package) +
`lumen-calendar.hpkg.sig`.

## Package payload

```
/apps/calendar/calendar              the app binary
/apps/calendar/app.ini               the bundle descriptor
/etc/aegis/caps.d/calendar           its capability policy
```

## Repository layout

```
src/        calendar source
pkg/        install-tree skeleton shipped verbatim (app.ini + caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — lumen-calendar is a Lumen client, so installing it pulls
[lumen](https://github.com/AspisOS/lumen) (which in turn provides the desktop
fonts).
