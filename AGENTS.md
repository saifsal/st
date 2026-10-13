# st

## Overview

st is a simple terminal emulator for X. `st.c` is the VT state machine (escape sequences, UTF-8, the PTY, and selection). `x.c` is the X11 front end: it opens the window, draws with Xft, and reads the keyboard and mouse. Appearance and key bindings are chosen at compile time in `config.h`, then the binary is rebuilt. The upstream description still holds: it is a terminal emulator for X which sucks less, by leaving out the legacy surface described in `LEGACY`.

This tree is version 0.9.2 (`VERSION` in `config.mk`). `./st -v` prints that version. The latest commit on `main` is `0.9.2`, dated 4 January 2025, and the program builds and runs. There is no test suite and no CI. Work that is still open is listed in `TODO`.

The committed `config.h` sets the background opacity `alpha` to 0.6 and sets `disablebold`, `disableitalic`, and `disableroman` to 1. `main` accepts `-A` to override `alpha` at runtime. `config.def.h` is not in the repository; `config.h` is the configuration that ships.

## Repository layout

- `st.c` — terminal state, escape sequences, UTF-8, the PTY, and selection.
- `x.c` — X11 window, fonts, colors, input, and `main`. This is the only file that includes `config.h`.
- `st.h` — shared types (`Glyph`, `Line`, `Rune`) and the functions both sides use.
- `win.h` — window and drawing operations implemented by `x.c` and called from `st.c`.
- `arg.h` — `ARGBEGIN` / `ARGEND` / `EARGF` macros for the command line.
- `config.h` — font, colors, opacity, keyboard and mouse bindings, and the fallback shell. Committed. There is no `config.def.h`.
- `config.mk` — `VERSION` (0.9.2), `PREFIX` (`/usr/local`), and the compiler and linker flags.
- `Makefile` — POSIX makefile. Targets: `all`, `clean`, `dist`, `install`, `uninstall`.
- `st.info` — terminfo source. Entries: `st`, `st-256color`, `st-meta`, `st-meta-256color`, `st-bs`, `st-bs-256color`, `st-mono`.
- `st.1` — man page. `make install` substitutes `VERSION` into the installed copy.
- `README` — short install note. `make dist` copies this file. `README.md` is the human documentation.
- `FAQ` — terminfo, scrollback, the backspace key, OpenBSD, `w3m` images, and the Xft emoji bug.
- `LEGACY` — why unsupported historical sequences stay unsupported.
- `TODO` — unfinished emulation and drawing work.
- `LICENSE` — MIT/X Consortium License.

No test directory and no CI configuration are present.

## Setup

The link line in `config.mk` needs a `c99` compiler, `make`, `pkg-config`, Xlib, Xft, fontconfig, freetype, and `libm`, `librt`, and `libutil`. Compiling `st.info` needs `tic` from ncurses.

On Debian and Ubuntu the corresponding packages are `build-essential`, `pkg-config`, `libx11-dev`, `libxft-dev`, `libfontconfig1-dev`, `libfreetype6-dev`, and `ncurses-bin`. With those installed, `make` completes using the stock `config.mk`.

`config.h` is already in the tree, so the makefile rule that would copy `config.def.h` does not run. Edit `config.h` before building when the defaults should change.

`X11INC` is `/usr/X11R6/include` and `X11LIB` is `/usr/X11R6/lib`. On the Ubuntu system used to check the build, those directories are absent and `make` still succeeds: the compiler searches `/usr/include`, and the linker finds `libX11` and `libXft` on its default path. When a build cannot find X11, set `X11INC` and `X11LIB` in `config.mk` or on the `make` command line. The commented OpenBSD block in `config.mk` drops `-lrt` and changes `MANPREFIX`. `FAQ` records the same `-lrt` note.

## Build, run and test

The makefile is marked `.POSIX`, so `CC` defaults to `c99` and `CFLAGS` defaults to `-O1`. Commands that were run successfully:

```
make
./st -v
```

`make` writes `./st`. `./st -v` prints the invoked name and `0.9.2` to stderr (`./st 0.9.2` when invoked that way) and exits 1, because `-v` calls `die()`.

```
./st
./st -e true
./st -A 0.85
```

`./st` runs the shell. `./st -e true` was run with `DISPLAY=:1` and exited 0. `-A` sets background opacity and clamps it to the range 0 through 1. It is parsed in `main` and is omitted from `usage()` and from `st.1`.

Terminfo, without installing the binary. This wrote seven entries:

```
tic -sx -o "$HOME/.terminfo" st.info
```

Install and uninstall honor `DESTDIR` for the binary and the man page. `PREFIX` defaults to `/usr/local`. Checked with a temporary `DESTDIR`:

```
make DESTDIR=/tmp/st-root install
make DESTDIR=/tmp/st-root uninstall
make clean
```

`make install` also runs `tic -sx st.info`, which ignores `DESTDIR`. As a normal user, `tic` writes `~/.terminfo`. As root, it writes the system terminfo database.

Automated tests are not applicable. There is no test suite and no `make test` target. The check before a commit is `make` and `./st -v`.

`make dist` is the release-tarball target (`st-0.9.2.tar.gz`). It fails on this tree: the recipe copies `config.def.h`, and that file is absent (`cp: cannot stat 'config.def.h'`).

`make CFLAGS='-O1 -Wall -Wextra'` also links. It prints existing warnings in `x.c` (signed comparisons, unused parameters, an unused `selclear_`). Those warnings are already in the tree.

## Code style and conventions

C, built as C99 with `-D_XOPEN_SOURCE=600`. Indent with two spaces. The C sources contain no tab characters, and lines are wrapped at 80 columns. Put the opening brace on the same line as the function or control statement. Names are short (`tnew`, `ttyread`, `xdrawline`). Fatal errors go through `die()`, which prints to stderr and `exit(1)`.

`config.h` is included from `x.c` only. A setting read from `st.c` has to be a non-static global with an `extern` declaration in `st.h`. A setting used only from `x.c` can live entirely in `config.h`. New flags go through the macros in `arg.h`, in the same `ARGBEGIN` block as the existing options.

Comments are `/* */` block comments and explain terminal behavior that the control flow does not make obvious. Nothing in the tree configures a formatter or a linter. Match the file being edited, and leave unrelated lines alone.

## CI and tooling

Not applicable. No GitHub Actions workflow, and no other CI configuration, is in the repository.

Local tooling is the POSIX `Makefile`, `pkg-config` for the fontconfig and freetype flags, and `tic` for `st.info`. No linter, formatter, or test harness is configured.

## Working rules for agents

- Keep changes small and focused.
- Run the tests before committing. Automated tests are not applicable; run `make` and `./st -v`.
- Never commit secrets.
- Update the docs when behavior changes, including `README.md`, `st.1` when flags or shortcuts change, `FAQ` when user-visible behavior changes, and these instruction files.
- Use conventional commit messages.
- Keep `AGENTS.md`, `CLAUDE.md`, and `.github/copilot-instructions.md` in sync. `AGENTS.md` is canonical. Copy the text; do not point the other two at it with symlinks.

Do not commit `st`, `*.o`, or `st-*.tar.gz`. Do not delete `config.h`. Adding `config.def.h` only helps if the makefile recipes and the real configuration still agree.

## Gotchas

- `config.def.h` is missing. `make` works because `config.h` is already present. Removing `config.h` makes the next build run `cp config.def.h config.h` and fail. `make dist` fails for the same missing file, and it leaves a `st-0.9.2/` directory behind after the `cp` error.
- `./st -v` exits 1. Every `die()` path does, including the version print.
- `-A <opacity>` works and is clamped with `LIMIT(alpha, 0.0, 1.0)`. `usage()` and `st.1` still document only `-aiv`.
- Shipped defaults make the background translucent (`alpha` 0.6) and skip distinct bold, italic, and roman faces. Both are fixed at compile time except opacity, which `-A` can override. `xinit` prefers a 32-bit TrueColor visual, and `xloadcols` stores `alpha` in the default background pixel. A compositing window manager is what actually shows that alpha.
- `disablebold`, `disableitalic`, and `disableroman` skip the fontconfig slant or weight request. The SGR attributes are still parsed; the face just does not change.
- `make install` runs `tic -sx st.info` with no `DESTDIR`. Only the `st` binary and `st.1` land under `DESTDIR`/`PREFIX`.
- `termname` is `st-256color`. Programs that do not recognize `st` need the terminfo installed with `tic` or `make install`. `FAQ` covers the symptoms.
- st has no scrollback buffer. `FAQ` points at a multiplexer, the external `scroll` helper, or a patch. `scroll` and `utmp` in `config.h` are `NULL`, so neither helper is exec'd.
- Backspace sends DEL (`^?`, the `XK_BackSpace` entry `"\177"`). `FAQ` explains the Linux convention. The `st-bs` terminfo entries reverse it.
- `tabspaces` is 8 and `st.info` has `it#8`. Change them together, then reinstall the terminfo entry.
- Wheel up sends `^Y` and wheel down sends `^E`. Shift-wheel sends the shifted page-up and page-down sequences. Those bindings are `mshortcuts` in `config.h`.
- `w3m` image drawing expects a single pixmap. st is double-buffered, which `FAQ` describes.
- Color emoji can raise BadLength inside Xft. `FAQ` treats that as an Xft bug, fixed in libXft 2.3.5, and asks that it not be reported against st.
- `pledge` is wrapped in `#ifdef __OpenBSD__`.
- `st.1` contains the literal `st\-VERSION`. The install recipe rewrites it with `sed`.
- `README` (no extension) is what `make dist` packages. `README.md` is what people read on GitHub. Update both when install behavior changes.
