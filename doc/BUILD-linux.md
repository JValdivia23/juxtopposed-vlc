# Building this fork on Linux (Meson)

> Status on CachyOS: `meson setup` verified
> (`--buildtype=debugoptimized --wrap-mode=nofallback
> -Dqt=enabled -Dmedialibrary=disabled -Dtests=auto`:
> 800 targets, 164 tests registered, 42 Qt registrations with
> both media-library tests correctly omitted). Full build,
> runtime, tests, and install are still pending. There is no
> installable Linux release; Linux users build the native VLC
> 4.0.0-dev tree from source. The Windows beta builds on the
> [Releases page](https://github.com/dsojevic/juxtopposed-vlc/releases)
> ship with the media library; the baseline below disables it,
> so expect that feature difference.

This guide uses **Meson** and the current build definitions
(root `meson.build` requires Meson `>= 1.10.0`). Note that
upstream itself prints "The Meson build system of VLC is
currently EXPERIMENTAL and INCOMPLETE", so this is the
practical path, not a claim of official support. The legacy
`INSTALL` file describes a different (autotools) flow and is
not these instructions.

## Prerequisites (Arch / CachyOS)

Starting point for toolchain, Qt 6, and the media stack:

```sh
sudo pacman -S --needed base-devel git meson ninja cmake pkgconf python \
    flex bison nasm \
    qt6-base qt6-declarative qt6-svg qt6-shadertools qt6-wayland \
    ffmpeg alsa-lib libpulse pipewire wayland wayland-protocols \
    libxkbcommon xcb-util-keysyms mesa lua gnutls libgcrypt \
    libxml2 sqlite libpng zvbi
```

This list is a starting point, not a tested minimal
fresh-machine closure. Option handling falls into two kinds:

- `auto` (the majority): a missing dependency silently
  disables that module.
- `enabled` (must be present or explicitly disabled):
  `avcodec`, `avformat`, `swscale`, `png`, `zvbi`, `lua`,
  `screen`. Distribution `ffmpeg` covers the
  `libav*-based` ones; `libpng` and `zvbi` are listed above
  for the `png` and `zvbi` options.

Further notes:

- `nasm` is optional (`required: false` in `meson.build`);
  without it you lose x86 assembly acceleration paths but
  the build succeeds.
- `qt6-wayland` is optional, and build-time versus run-time
  needs differ. The Wayland compositor sources compile on
  the QtGui private headers plus the `wayland`/`xcb` pieces;
  actually *running* on Wayland additionally needs the Qt
  Wayland platform plugin from that package. Without it the
  build still succeeds and uses XCB when available (see
  below).
- The Qt interface needs Qt 6 `>= 6.2` (`Core`, `Gui`,
  `Widgets`, `Qml`, `Network`, `Quick`, `QuickControls2`,
  including private headers). `-Dqt=enabled` turns a missing
  or incomplete Qt into a configure error instead of a
  silently skipped UI.
- `medialibrary` needs an external `medialibrary >= 0.13`
  via pkg-config. It is not installed by this baseline and
  not in the package list above, so the baseline disables it
  explicitly; provide a compatible pkg-config package to
  enable it. Expect reduced library functionality without it
  (history, media-library views).

Other distributions are unverified; install the equivalents
of the packages above.

## Configure and build

Always use a **separate build directory** and a **private
install prefix** (for example under your home directory).
Never install this development tree over the system VLC 3.x
libraries or plugins.

From the parent of your checkout, so the build directory
ends up beside (not inside) the source tree:

```sh
git clone https://github.com/dsojevic/juxtopposed-vlc.git
meson setup build-linux juxtopposed-vlc \
    --buildtype=debugoptimized --wrap-mode=nofallback \
    --prefix="$HOME/.local/vlc-dev" \
    -Dqt=enabled \
    -Dmedialibrary=disabled
meson compile -C build-linux -j2
```

`-j2` caps build parallelism for small machines (suggested
for a 7.4 GiB host); raise it to suit your own hardware,
never unlimited on memory-constrained hosts. `meson
configure build-linux` lists detected option values after
setup.

## Running from the build tree

Use the **`vlc-static`** binary, not `bin/vlc`: only the
`vlc-static` target is compiled with `TOP_BUILDDIR` /
`TOP_SRCDIR`, which point the plugin, data, library and
libexec paths at the build tree (`bin/vlc.c`,
`bin/meson.build`). The name marks it as the build-tree
launcher, not a fully statically linked executable.

Isolate test runs from your everyday VLC configuration with
a throwaway XDG root (VLC maps `XDG_CONFIG_HOME` /
`XDG_DATA_HOME` / `XDG_CACHE_HOME` to `<root>/vlc`, see
`src/posix/dirs.c`). The `env KEY=value` prefix works in
both POSIX `sh` and `fish`:

```sh
test_root="$(mktemp -d)"
mkdir -p "$test_root/config" "$test_root/data" "$test_root/cache"
env XDG_CONFIG_HOME="$test_root/config" \
    XDG_DATA_HOME="$test_root/data" \
    XDG_CACHE_HOME="$test_root/cache" \
    ./build-linux/bin/vlc-static --version
```

(`fish` users: `set test_root (mktemp -d)` for the first
line; the `env`-prefixed invocations are unchanged.) Use the
same XDG wrapping for every later run, replacing
`--version` with your actual arguments.

Do not set `VLC_PLUGIN_PATH` to the system VLC 3.x plugin
directory: this is a VLC 4 (`libvlc` ABI 12.0.0) build and
must only load its own build-tree plugins. Close other VLC
instances before testing.

## Optional: install to a user prefix (documented, unverified)

The build definitions do install (per-target `install: true`
with user-prefix-relative destinations), but no install has
been run or measured yet, and the tree sets no RPATH and
ships no install hooks. In particular, a plain `meson
install` followed by `<prefix>/bin/vlc` is not expected to
just work: the system loader does not search your prefix
library directory, so `libvlc.so.12` and `libvlccore.so.9`
will not resolve. Until a staged install is validated,
prefer the build-tree `vlc-static` runner above.

Once an install exists, find your actual library directory
first — do not assume `lib` vs `lib64` or multi-arch
triplets:

```sh
meson configure build-linux | grep -E '^[[:space:]]*(libdir|prefix)[[:space:]]'
```

then launch per-process (never a global export, never
`ld.so.conf` changes) with full XDG isolation:

```sh
env LD_LIBRARY_PATH="$HOME/.local/vlc-dev/lib" \
    XDG_CONFIG_HOME="$test_root/config" \
    XDG_DATA_HOME="$test_root/data" \
    XDG_CACHE_HOME="$test_root/cache" \
    "$HOME/.local/vlc-dev/bin/vlc" --version
```

(substitute the real libdir from the configure output for
`lib` above). A non-destructive staged check the install
itself writes only where told is `meson install -C build-linux
--no-rebuild --destdir <scratch-dir>`; validation of that
is still pending.

## Wayland notes

Recent `master` already contains the Linux fixes reported in
issue #4 (Wayland EGL setup, PipeWire device selection,
video embedding when Wayland integration is unavailable, and
the `libvlccore` export the Qt plugin links against).
Do not carry the old standalone patches by default.

If video still opens in a separate window on Wayland, the
optional fallback is the XCB platform with in-window video
(`embedded-video` defaults to true), still under XDG
isolation:

```sh
env QT_QPA_PLATFORM=xcb \
    XDG_CONFIG_HOME="$test_root/config" \
    XDG_DATA_HOME="$test_root/data" \
    XDG_CACHE_HOME="$test_root/cache" \
    ./build-linux/bin/vlc-static --embedded-video
```

## Tests

`meson compile` does not build test executables (they are
`build_by_default: false`), while plain `meson test`
silently rebuilds prerequisites with an uncapped Ninja. So
build the test targets with the job cap first, then run with
`--no-rebuild`. Start with a small smoke subset:

```sh
meson compile -C build-linux -j2 base_model_test test_qt_track_folder_rules
env QT_QPA_PLATFORM=offscreen \
    XDG_CONFIG_HOME="$test_root/config" \
    XDG_DATA_HOME="$test_root/data" \
    XDG_CACHE_HOME="$test_root/cache" \
    meson test -C build-linux --no-rebuild \
        base_model_test test_qt_track_folder_rules
```

Offscreen plus throwaway XDG dirs keeps this headless and
isolated; offscreen alone is not proof of either.

For the full Qt suite (`VLC:qt`, 42 registrations), build
all test prerequisites capped first — this builds every
test's prerequisites, not just Qt's, so it is the larger
build — then run with `--no-rebuild`:

```sh
ninja -C build-linux -j2 meson-test-prereq
env QT_QPA_PLATFORM=offscreen \
    XDG_CONFIG_HOME="$test_root/config" \
    XDG_DATA_HOME="$test_root/data" \
    XDG_CACHE_HOME="$test_root/cache" \
    meson test -C build-linux --no-rebuild --suite VLC:qt --num-processes 2
```

No pass/fail counts are claimed here; report whatever this
tree actually produces.
