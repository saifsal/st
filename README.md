# st

st is a simple terminal emulator for X. It is version 0.9.2 of the suckless terminal: a small VT, drawn with Xft, configured by editing a header and recompiling. The aim is a terminal that does the sequences you actually need and leaves the rest out (`LEGACY` is the short statement of that choice).

The shipped `config.h` sets the background opacity to 0.6 and turns off distinct bold, italic, and roman font faces. `main` also accepts `-A` to override that opacity for one process. `st.1` documents the command line; it does not mention `-A`.

## Features

- UTF-8, 256 colors, and direct RGB color (`38;2` / `48;2`).
- Xft fonts via fontconfig. The default face is `monospace`.
- A compile-time config file, `config.h`: font, colors, opacity, geometry, keyboard and mouse bindings, fallback shell.
- Runtime opacity with `-A`, clamped to 0 through 1. The shipped default is `0.6`.
- Switches `disablebold`, `disableitalic`, and `disableroman`. All three are `1` in the shipped config, so those faces are not requested from fontconfig. The corresponding SGR attributes are still parsed.
- Alternate screen (disabled with `-a`), bracketed paste, and the usual mouse-tracking modes.
- Mouse selection on the primary selection: double-click snaps to a word, triple-click snaps to a line, Alt drags a rectangle.
- Clipboard copy and paste, primary-selection paste, and font zoom.
- A serial line instead of a PTY (`-l`), embedding into another window (`-w`), and optional logging of the I/O stream (`-o`).
- Terminfo entries for `st-256color` (the default `TERM`), `st`, `st-meta`, `st-bs`, and the 256-color variants, plus `st-mono`.
- Optional external `scroll` and `utmp` programs. Both are `NULL` in this `config.h`, so st does not run them.

st does not keep a scrollback buffer. `FAQ` explains the usual substitutes: `tmux`, `screen`, the suckless `scroll` helper, or the scrollback patch.

## Examples

Invocations that match the flags in `x.c` and `st.1`:

```sh
make
tic -sx st.info
./st
./st -f "monospace:pixelsize=16" -g 100x30 -A 0.9
./st -e tmux
```

`-e` has to be the last option. Anything after it is the command. The same is true if you omit `-e` and just pass a command.

A serial line, from the man page:

```sh
st -l /dev/ttyS0 115200
```

## Prerequisites

- A C compiler installed as `c99`. The POSIX makefile uses that name. The build checked here used GCC 13.3 via that driver.
- `make` and `pkg-config`.
- Xlib and Xft headers and libraries.
- fontconfig and freetype.
- `libm`, `librt`, and `libutil` (present on a normal glibc system; OpenBSD has no `librt`, see below).
- `tic` from ncurses, to install `st.info`.
- A running X server to open a window. `./st -v` does not need one.

Debian and Ubuntu package names that satisfy this:

```sh
sudo apt install build-essential pkg-config libx11-dev libxft-dev \
  libfontconfig1-dev libfreetype6-dev ncurses-bin
```

`make` was run successfully after those packages were installed, with `config.mk` left unchanged.

## Install

`config.h` is already in the repository. Edit it first if the font, colors, or bindings should differ, then:

```sh
make
sudo make install
```

`PREFIX` defaults to `/usr/local`, so the binary lands at `/usr/local/bin/st` and the man page at `/usr/local/share/man/man1/st.1`. `make install` also runs `tic -sx st.info`. The same recipe was run as `make DESTDIR=/tmp/st-root install`: the binary and the man page went under that prefix, and `tic` still wrote `~/.terminfo`.

`X11INC` and `X11LIB` in `config.mk` point at `/usr/X11R6`. That prefix is not required when the compiler already searches `/usr/include` and the linker already searches the system library directory, which is the case on current Debian and Ubuntu. If X11 lives somewhere else, set the two variables and rebuild.

To stage the binary and the man page without writing `/usr/local`:

```sh
make DESTDIR=/tmp/st-root install
```

`tic` ignores `DESTDIR`. Run as a normal user, it writes `~/.terminfo`. Run as root, it writes the system terminfo database.

Uninstall:

```sh
sudo make uninstall
```

On OpenBSD, `FAQ` and the comments in `config.mk` say to drop `-lrt` (the functions live in libc there) and to use `MANPREFIX = ${PREFIX}/man`.

If you only want to try st and skip `make install`, compile the terminfo entry yourself:

```sh
tic -sx st.info
```

That wrote seven entries when checked.

## Usage

```
st [-aiv] [-c class] [-f font] [-g geometry] [-n name] [-o file]
   [-T title] [-t title] [-w windowid] [[-e] command [args ...]]
st [-aiv] [-c class] [-f font] [-g geometry] [-n name] [-o file]
   [-T title] [-t title] [-w windowid] -l line [stty_args ...]
```

`-A opacity` is implemented and missing from that usage string and from `st.1`. It overrides `alpha` for this process.

| Flag | Effect |
| --- | --- |
| `-a` | Disable the alternate screen. |
| `-A opacity` | Background opacity from 0 to 1. Default in `config.h` is 0.6. |
| `-c class` | Window class. Default is `termname` (`st-256color`). |
| `-f font` | fontconfig font pattern, instead of `font` in `config.h`. |
| `-g geometry` | `[=][<cols>{xX}<rows>][{+-}<xoffset>{+-}<yoffset>]`. |
| `-i` | Fix the position given with `-g`. |
| `-n name` | Window instance name. Default is `termname` (`st-256color`). |
| `-o file` | Copy the I/O stream to `file`. `-` is stdout. |
| `-T` / `-t title` | Window title. Default `st`, or the command name when `-e` is used. |
| `-w windowid` | Embed in an existing window. |
| `-l line` | Use a tty device instead of a PTY. Remaining arguments are `stty` flags; the default line setup is 8 bits, no parity, 1 stop bit, 38400 baud. |
| `-v` | Print the version to stderr and exit. The exit status is 1. |
| `-e command` | Run `command` instead of the shell. Must be the last option. |

What st executes, when `-e` is not used:

1. The `scroll` program, if `scroll` in `config.h` is set (it is `NULL` here). `utmp`, if set, is its argument; otherwise the shell is.
2. The `utmp` program, if only that pointer is set.
3. `$SHELL`, then the shell from the passwd entry, then `shell` in `config.h` (`/bin/sh`).

Keyboard shortcuts from `config.h`. `st.1` lists the same keys. Shift-Insert, middle click, and Ctrl-Shift-Num Lock are only in `config.h`:

| Keys | Action |
| --- | --- |
| Ctrl-Shift-Page Up / Page Down | Larger / smaller font |
| Ctrl-Shift-Home | Default font size |
| Ctrl-Shift-c / Ctrl-Shift-v | Copy to the clipboard / paste from it |
| Ctrl-Shift-y, Shift-Insert, middle click | Paste the primary selection |
| Print Screen | Write the selection to the `-o` file |
| Shift-Print Screen | Write the full screen to the `-o` file |
| Ctrl-Print Screen | Toggle copying the stream to the `-o` file |
| Break (Ctrl-Pause) | Send a break on the serial line |
| Ctrl-Shift-Num Lock | Toggle num lock handling |

Wheel up sends `^Y`, wheel down sends `^E`. Shift-wheel sends the shifted page-up and page-down sequences. The full table is `shortcuts`, `mshortcuts`, and `key` in `config.h`.

`man st` after installation has the same option list minus `-A`, plus the customization note: change `config.h` and recompile.

## Configuration

`config.h` is included by `x.c`. Change a value, run `make`, and the new binary picks it up. Values that `st.c` also reads are non-static globals declared again in `st.h` (`termname`, `tabspaces`, `allowaltscreen`, the default colors, `scroll`, `utmp`, and others).

Worth knowing about the shipped file:

- `font` is `"monospace"`. `borderpx` is 2. The window starts at 80×24.
- `alpha` is `0.6`. `-A` overrides it without a rebuild.
- `disablebold`, `disableitalic`, and `disableroman` are `1`.
- `termname` is `st-256color`. The first 16 `colorname` entries are the ANSI colors. Indexes after 255 are the extras: `defaultcs` 256 is `#cccccc` (cursor), `defaultrcs` 257 is `#555555`, `defaultfg` 258 is `gray90`, `defaultbg` 259 is `black`.
- `bellvolume` is 0 (silent). `blinktimeout` is 800 ms. The cursor is a block (`cursorshape` 2).
- `allowwindowops` is 0, so non-interactive window operations such as OSC clipboard writes stay off.
- `tabspaces` is 8. `st.info` has `it#8`. If you change one, change the other and rerun `tic`.
- `shell` is `/bin/sh`, used only when `$SHELL` and the passwd shell are empty.

`config.mk` holds `VERSION`, `PREFIX`, `MANPREFIX`, and the library flags. It is not a second copy of the visual theme.

## Development and testing

```sh
make
./st -v          # prints "./st 0.9.2" to stderr; exits 1
./st -e true     # needs a display; exits 0 when the command exits 0
make clean
```

There is no test suite and no `make test` target. `make` plus `./st -v` is the check that was used for this tree.

`make CFLAGS='-O1 -Wall -Wextra'` links, and it prints warnings that are already in `x.c`. The default recipe does not pass `-Wall`.

`make dist` does not succeed here. It tries to package `config.def.h`, and this repository has `config.h` instead:

```
cp: cannot stat 'config.def.h': No such file or directory
```

Agent-oriented notes, layout, and the gotcha list live in `AGENTS.md`. `CLAUDE.md` and `.github/copilot-instructions.md` carry the same text.

## Project status

Version 0.9.2, as of the `0.9.2` commit on 4 January 2025. The emulator builds, installs, and runs.

`TODO` is the remaining list, in the project's own words:

- VT: double-height lines.
- A simple way to multiplex.
- Drawing: diacritics, a simpler font cache, better brightening of the upper colors.
- Bugs: shift-up/down selection in emacs, and removing the DEC test sequence when it is appropriate.

`FAQ` is the place for behavior that surprises people and stays that way inside st: missing terminfo, scrollback left to another program, the backspace key, OpenBSD's missing `librt`, `w3m` images against the double buffer, and the Xft color-emoji crash.

## Licence

MIT/X Consortium License. The copyright holders and the full terms are in `LICENSE`.
