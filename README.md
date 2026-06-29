# lumen-calendar

The calendar for **AspisOS**, a capability-based, no-ambient-authority operating
system built on the from-scratch [Aegis](https://github.com/AspisOS/Aegis)
kernel.

lumen-calendar is a month-view calendar with per-day notes. It is an external
client of the [lumen](https://github.com/AspisOS/lumen) compositor, distributed
as a [herald](https://github.com/AspisOS/AspisOS) package and installed as an
`/apps` bundle. Its descriptor's display name is **Calendar**.

## The AspisOS ecosystem

AspisOS is decomposed into independent repositories; lumen-calendar is one
graphical leaf of that tree:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel. Provides the capability model, the `AF_UNIX` socket to the compositor, and the filesystem where notes are stored. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. lumen-calendar connects to its socket for a window and input events. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit. Supplies the renderer, the theme, the timezone offset (`glyph_theme_tz_offset`) for the header clock, and the client side of lumen's window protocol (`lumen_client.h`). |
| [`AspisOS/AspisOS`](https://github.com/AspisOS/AspisOS) | The OS: userland, rootfs, ISO/installer, and the herald package manager that installs this `.hpkg`. |

## What it does

Grounded in `src/main.c`:

- **One window.** Connects to lumen and draws a single 476×430 window showing
  the current month as a 7×6 grid. Today is highlighted, the weekend column
  labels are accented, and a live clock/date sits in the header.
- **Per-day notes.** Selecting a day reveals an inline note editor; typing edits
  the note and Enter saves it. Days with a saved note carry a marker dot. Notes
  persist to `$HOME/.calendar` (defaulting to `/root` when `$HOME` is unset),
  one `YYYY-MM-DD note` line per day; saving an empty note removes that day's
  line.
- **Self-contained date math.** Leap-year handling and a Sakamoto day-of-week
  computation place each cell; the header clock applies
  `glyph_theme_tz_offset()` to render local time and refreshes once a second.
- **Keys.** Arrows move the selection, PgUp/PgDn (or `[` / `]`) change month,
  typing edits the selected day's note, Enter saves, Esc reverts the edit,
  Q/close quits. The header arrows and day cells are also clickable.

## Capabilities

AspisOS has no ambient authority: a process can do nothing except through
capabilities granted at exec time. lumen-calendar's policy
(`pkg/etc/aegis/caps.d/calendar`) is the baseline desktop-app profile:

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
- `HERALD_KEY` signs the `.hpkg` (ECDSA P-256).

Output: `lumen-calendar.hpkg` (a `class=system` herald package) +
`lumen-calendar.hpkg.sig`.

## Package payload

The `.hpkg` is a manifest-first, uncompressed POSIX `ustar` archive with a
detached signature. Its payload tree:

```
/apps/calendar/calendar              the app binary (stripped)
/apps/calendar/app.ini               the bundle descriptor (name=Calendar, exec=calendar)
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
[lumen](https://github.com/AspisOS/lumen) (which in turn ships the desktop fonts
every dependent inherits).

## Status

Early-stage and intentionally simple: a single-month view with one short note
per day kept in a flat `$HOME/.calendar` file — no recurring events, reminders,
multi-line entries, or time-of-day scheduling. It is a useful note-keeping
calendar today and is expected to grow as AspisOS matures.
