# Setup

These instructions reproduce the setup used to run FL Studio 20 under
Wine on Wayland with the fixes in this repository.

The commands below assume Fish shell and use these paths:

- source trees: `~/src`
- custom Wine install: `~/.local/wine-d2d1-dcomp-11.18`
- custom xwayland-satellite binary: `~/.local/bin/xwayland-satellite-fl`
- Wine prefix: `~/.wine`
- X display: `:12`

These instructions assume FL Studio 20 is already installed in the Wine
prefix at `~/.wine`.

Adjust paths as needed.

## 1. Requirements

You need the normal Wine build dependencies, including both 64-bit and
32-bit development libraries for a traditional multilib build.

You also need:

- Git
- Xwayland
- Rust/Cargo
- clang
- xcb
- xcb-util-cursor
- winetricks

This setup was tested with:

- Wine base: giang17/wine `d2d1-dcomp-11.18`
- Wine base commit:
  `a05cc9f2603462e8d43f5e5de05e5d4ef853e1ff`
- xwayland-satellite PR 500 base:
  `d8832532bcf15a5f0173afeca05735d17011555a`

## 2. Clone this repository

```fish
cd ~/src
git clone https://github.com/rx-560/fl-studio-linux-wayland.git
```

## 3. Build patched Wine
Clone giang17's Wine fork:

```fish
cd ~/src

git clone https://github.com/giang17/wine.git \
    wine-d2d1-dcomp-11.18

cd wine-d2d1-dcomp-11.18

git switch d2d1-dcomp-11.18
git reset --hard a05cc9f2603462e8d43f5e5de05e5d4ef853e1ff
```

Apply the FL Studio XShape/alpha patch:

```fish
git am \
    ~/src/fl-studio-linux-wayland/wine/patches/fl-studio-xshape-alpha.patch
```

### Build 64-bit Wine

```fish
mkdir -p ~/src/wine-d2d1-dcomp-11.18-build
cd ~/src/wine-d2d1-dcomp-11.18-build

../wine-d2d1-dcomp-11.18/configure \
    --enable-win64 \
    --prefix="$HOME/.local/wine-d2d1-dcomp-11.18"

make -j(nproc)
make install
```

### Build the traditional 32-bit companion

```fish
mkdir -p ~/src/wine-d2d1-dcomp-11.18-build32
cd ~/src/wine-d2d1-dcomp-11.18-build32

../wine-d2d1-dcomp-11.18/configure \
    --with-wine64="$HOME/src/wine-d2d1-dcomp-11.18-build" \
    --prefix="$HOME/.local/wine-d2d1-dcomp-11.18"

make -j(nproc)
make install
```

Reinstall the 64-bit half afterwards:
```fish
cd ~/src/wine-d2d1-dcomp-11.18-build
make install
```

Verify:
```fish
"$HOME/.local/wine-d2d1-dcomp-11.18/bin/wine" --version
```

## 4. Build patched xwayland-satellite

Clone upstream:
```fish
cd ~/src

git clone https://github.com/Supreeeme/xwayland-satellite.git \
    xwayland-satellite-fl

cd xwayland-satellite-fl
```

Fetch PR 500 and select the tested base commit:
```fish
git fetch origin refs/pull/500/head:pr500
git switch pr500
git reset --hard d8832532bcf15a5f0173afeca05735d17011555a
```

Apply the FL Studio dialog patch:
```fish
git am \
    ~/src/fl-studio-linux-wayland/xwayland-satellite/patches/fl-dialog-toplevel.patch
```

Build:
```fish
cargo build --release
```

Install the binary:
```fish
install -Dm755 \
    target/release/xwayland-satellite \
    "$HOME/.local/bin/xwayland-satellite-fl"
```

## 5. Install WebView2

WebView2 is required by plugins such as Cymatics Corrosion and Dark Sky.

You will need a recent version of winetricks for the installation of WebView2.

Back up an existing Wine prefix before installing WebView2 if it contains
anything important.

Start the patched xwayland-satellite instance:
```fish
"$HOME/.local/bin/xwayland-satellite-fl" :12 &
set satellite_pid $last_pid
sleep 1
```

Install WebView2 into the Wine prefix.
The explicit `WINE64` override is intentional: winetricks may otherwise
mis-detect this custom traditional-multilib Wine install.
```fish
env -u WAYLAND_DISPLAY \
    WINE="$HOME/.local/wine-d2d1-dcomp-11.18/bin/wine" \
    WINE64="$HOME/.local/wine-d2d1-dcomp-11.18/bin/wine" \
    WINESERVER="$HOME/.local/wine-d2d1-dcomp-11.18/bin/wineserver" \
    WINEPREFIX="$HOME/.wine" \
    DISPLAY=:12 \
    winetricks -q webview2
```

Stop the temporary satellite:
```fish
kill $satellite_pid
wait $satellite_pid 2>/dev/null
```

Verify:
```fish
find "$HOME/.wine/drive_c/Program Files (x86)/Microsoft/EdgeWebView/Application" \
    -iname msedgewebview2.exe \
    -print
```

## 6. Install the launcher

```fish
install -Dm755 \
    ~/src/fl-studio-linux-wayland/launcher/flstudio \
    ~/.local/bin/flstudio
```

The launcher deliberately sets:
```text
WINE_X11_BAKE_SHAPE_ALPHA=1
```
only for FL Studio.
Do not set this variable globally.

## 7. FL Studio setting

In FL Studio Settings > General, enable:
```text
Detach all plugins
```

This avoids corruption seen when some plugin UIs are embedded inside
FL Studio's main window.



## What each fix does
| Problem | Fix |
| --- | --- |
| Black corners around FL fruit/About window | Wine XShape-to-alpha patch |
| Settings window geometry/input desync | xwayland-satellite PR500 + dialog patch |
| SerumFX crash when enabling effects | giang17 D2D/DComp Wine |
| Corrosion / Dark Sky black UI | Microsoft Edge WebView2 |
| Embedded Serum rendering corruption | Detach all plugins |
| Very poor frame pacing with WebView2 | Do not force `--disable-gpu` |

## Known issues

WebView2-based plugins can use a significant amount of CPU.

Corrosion can occasionally flicker or show stale/white areas until its
UI is invalidated by interacting with a control.

Do not force WebView2 to use `--disable-gpu`; in testing this caused
very poor overall FL Studio frame pacing despite FL reporting a high
GUI frame rate.

