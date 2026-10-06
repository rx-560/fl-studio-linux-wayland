# FL Studio on Wine + Wayland fixes

Personal compatibility stack for running FL Studio 20 under Wine on Wayland.

Detailed install instructions are in `/docs/setup.md`.

Current stack:

- Wine 11.18 based on giang17's D2D/DComp work
- XShape -> ARGB alpha workaround for FL Studio's fruit/About window
- xwayland-satellite PR500
- FL Studio dialog-as-toplevel patch
- Microsoft Edge WebView2 for plugins such as Cymatics Corrosion and Dark Sky
- FL Studio "Detach All Plugins" enabled to avoid embedded plugin rendering corruption
- Wine shell32 patch to default file dialogs to Modified, newest-first sorting

## Fixes

| Problem | Fix |
| --- | --- |
| FL Studio fruit/About black corners | Wine XShape -> alpha patch |
| Settings dialog movement/input desync | xwayland-satellite PR500 + dialog patch |
| SerumFX crash when enabling effects | giang17 D2D/DComp Wine |
| Corrosion / Dark Sky black UI | WebView2 runtime |
| Embedded Serum movement corruption | Enable "Detach All Plugins" |
| WebView2 causing terrible frame pacing | Do not force `--disable-gpu` |
| Wine file dialogs reopening sorted by name | shell32 default-sort patch (Modified, newest first)

## Wine base

giang17 Wine branch:

`d2d1-dcomp-11.18`

Base commit:

`a05cc9f2603462e8d43f5e5de05e5d4ef853e1ff`

## xwayland-satellite base

PR500 commit:

`d8832532bcf15a5f0173afeca05735d17011555a`

## Notes

`WINE_X11_BAKE_SHAPE_ALPHA` should only be set for FL Studio, not globally.
