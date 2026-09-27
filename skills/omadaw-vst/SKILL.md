---
name: omadaw-vst
description: >
  Manage Windows VST plugins on Omarchy via the dedicated OmaDAW Wine prefix
  and yabridge (install setup.exe, sync bridges, discover new/updated plugins,
  diagnose missing plugins). Use for any Windows-VST, yabridge, yabridgectl,
  Wine/Proton prefix, Waves Central, or plugin-scan task. Triggers: VST,
  yabridge, setup.exe, wine prefix, Waves, Waves Central, WaveShell,
  Native Access, Service Center, plugin not showing in DAW, bridge sync.
---

# OmaDAW VST Skill

Windows VSTs run through **yabridge** inside a **dedicated** Wine prefix.
The repo wrapper `bin/omadaw-vst` owns every operation — call it, don't
reimplement yabridgectl invocations from memory.

## Baseline facts (verified Sep 2026)

- Bridge: `yabridge` + `yabridgectl` 5.1.x (extra). Runner: `wine-staging` 11.x.
- Prefix: `~/.wine-omadaw` ( env `WINEPREFIX` ). It is **not** `~/.wine`
  (that prefix belongs to games/launchers) — never install VSTs there.
- Plugin homes inside prefix:
  `drive_c/Program Files/Steinberg/VstPlugins` (VST2),
  `drive_c/Program Files/Common Files/VST3` (+ `.../VST2`, `.../CLAP`).
- Linux scan dirs (yabridge shims land here): `~/.vst`, `~/.vst3`
  (`~/.clap`, `~/.lv2` for native formats).
- DAWs must list those scan dirs (see `config/daw/*.md`); after any change,
  re-scan inside the DAW.

## Workflows

### Install a Windows plugin from its installer

```bash
bin/omadaw-vst install ~/Downloads/SomePluginSetup.exe
# .msi works too; extra silent flags are forwarded: install <file> /S ...
# Custom prefix: bin/omadaw-vst install <file>  (with WINEPREFIX=/path exported)
```

`install` runs the installer **inside** `~/.wine-omadaw`, then `sync`s so the
new `.dll`/`.vst3` is bridged immediately.

### Waves Central

Waves licenses through Waves Central (its own account login). The PACE/iLok
stack does not activate Waves products. Do not install or download plugins;
stop when the Central login window is open and let the user sign in.

Verified 2026-09-24 on wine-staging 11.17, prefix `~/.wine-omadaw` (win64):
Central **17.0.4**. The Windows button on
<https://www.waves.com/downloads/central> redirects
`/dlrdr?id=central-win` to
`https://cf-installers.waves.com/WavesCentral/Install_Waves_Central.exe`
(NSIS). Save it under `~/Downloads/` and install silently:

```bash
bin/omadaw-vst install ~/Downloads/Install_Waves_Central.exe /S
```

Confirm the version in the prefix uninstall key `DisplayVersion` (17.0.4).
The app is `drive_c/Program Files/Waves Central/Waves Central.exe`.

Before the first launch, Wine's stub
`drive_c/windows/system32/WindowsPowerShell/v1.0/powershell.exe` exits 0 and
prints nothing. Central treats that as "cannot access PowerShell" and stops.
Install only this winetricks verb (it pulls `powershell_core` 7.4.x, then
replaces the stub with a wrapper):

```bash
WINEPREFIX=~/.wine-omadaw winetricks -q powershell
```

The check Central runs must print a version (`7.4` after that verb):

```bash
WINEPREFIX=~/.wine-omadaw wine powershell -command "&{'{0}.{1}' -f \$PSVersionTable.PSVersion.Major, \$PSVersionTable.PSVersion.Minor}"
```

Launch and leave the window open:

```bash
bin/omadaw-vst run "drive_c/Program Files/Waves Central/Waves Central.exe"
```

An agent or other non-TTY parent hits `Uncaught Exception: Error: open EBADF`
in `createWritableStdioStream` / `process.stderr` (Electron `debug` touching
stderr). That is a piped stderr, not a missing DLL. Wrap the same command in
`script -q -f -e -c '…' /tmp/waves-central.typescript` so stderr is a pty.
A normal terminal does not need `script`.

These log lines happened on the working 17.0.4 run and were not blockers once
PowerShell answered: bundled VC++ 2022 MSI `WixDependencyCheck` returned 259
while `vcruntime140.dll` and DisplayVersion `14.34.31931` were present (do
not add `vcrun2022` unless a DLL is actually missing); `curl.exe` missing
("unable to detect OS curl version"); GPU process exit then swiftshader;
`Register-WMIEvent` warning under PowerShell 7.

After the user finishes installing products, bridge. Central copies VST3
shells into `drive_c/Program Files/Common Files/VST3/` (already registered;
`WaveShell*-VST3 *_x64.vst3`, a single-file PE DLL, not a bundle directory).
VST2 shells go to `drive_c/Program Files/VSTPlugIns/` (`WaveShell*-VST *_x64.dll`),
which yabridge does not watch. The same shells also sit under
`drive_c/Program Files (x86)/Waves/WaveShells V<N>/`. Register one of those
two VST2 directories, not both, and not the instl cache, AAX, or WPAPI copies:

```bash
bin/omadaw-vst add "$HOME/.wine-omadaw/drive_c/Program Files/VSTPlugIns"
bin/omadaw-vst sync
```

The shell folder version follows the account's products (this machine
installed V13 shells). Then re-scan in the DAW.

### Register / sync / list

```bash
bin/omadaw-vst add "<dir-containing-dlls>"  # one-off dir registration
bin/omadaw-vst sync                          # re-bridge; discovers new/updated plugins
bin/omadaw-vst list                          # yabridgectl status + shim inventory
```

Do not `yabridgectl add` `~/.vst`, `~/.vst3`, `~/.clap`, or `~/.lv2`.
Those are where centralized shims are written. Registering one makes the
next sync bridge its own output.

Symptom: `~/.vst/yabridge/yabridge` and `~/.vst3/yabridge/yabridge` appear,
the shim count roughly doubles, and `yabridgectl status` lists the copies
as not yet synced. `bin/omadaw-vst init` used to add `~/.vst` and `~/.vst3`
on every setup (verified 2026-09-27).

`yabridgectl rm` then asks to delete every leftover `.so` in that
directory. The real bridges live there. Answer anything other than `YES`
(a piped `n` is enough), then delete only the nested copy:

```bash
printf 'n\n' | yabridgectl rm ~/.vst
printf 'n\n' | yabridgectl rm ~/.vst3
rm -rf ~/.vst/yabridge/yabridge ~/.vst3/yabridge/yabridge
bin/omadaw-vst sync
```

`init` and `sync` now drop those registrations and delete a single nested
`yabridge/yabridge` directory before syncing. A shim whose Windows file is
already gone stays as a broken symlink inside the `.vst3` bundle. Doctor
reports `EMPTY`, and `yabridgectl status` warns that winedump could not
read the file. `yabridgectl sync --prune` removes that bundle. Unregister
the Linux dirs before that prune, or the scan treats the bundle as a plugin
and rebuilds it.

### Discovery of new/updated plugins

Two layers (both intentional):

1. Manual/immediate: `bin/omadaw-vst sync` (also auto-runs after `install`).
2. Background: systemd user path unit `omadaw-vst-sync.path` watches the
   prefix VST dirs + `~/.vst{,3}` and runs `yabridgectl sync` on change:
   ```bash
   bin/omadaw-vst enable-watch | disable-watch | watch-status
   ```
   Sources: `config/systemd/user/omadaw-vst-sync.{path,service}`.

### Doctor (plugin missing from DAW? run this first)

```bash
bin/omadaw-vst doctor
```
Fix tree:

1. Prefix missing → `bin/omadaw-vst init` (Proton variant:
   `bin/omadaw-vst init --proton <path>`; needs `umu-run`, else falls back to wine).
2. `yabridgectl status` fails → `bin/omadaw-vst sync`, retry with `--no-verify`
   is already handled inside `sync`.
3. Shim exists but DAW doesn't list it → check DAW scan paths (`~/.vst`, `~/.vst3`
   present in DAW settings per `config/daw/*.md`), then DAW "re-scan"
   (Reaper: clear cache/re-scan).
4. 32-bit plugin on 64-bit-only wine → needs matching wine arch; report it,
   don't hack the prefix.
5. Run an installer GUI manually: `bin/omadaw-vst run "drive_c/path/to/setup.exe"`.

### Load-testing without a DAW (`bin/omadaw-vst test`)

`doctor` ends with a smoke load-test (all `*shell*` shims + first 2 VST2/VST3,
via `carla-discovery`, no DAW). Verdicts: `PASS(n)` = loads and enumerates n
plugins; `EMPTY` = loads but lists nothing (unactivated Waves shells look
exactly like this — check the `.wle` first); `TIMEOUT` = didn't finish in
time (first-run auth dialogs block headless loads — authorize via the
standalone/app first, e.g. Addictive, IK Custom Shop prompts); `CRASH` =
loader died. Full sweep: `bin/omadaw-vst test --all [--timeout S] [--jobs J]`
 (slow on big collections); or `test <pattern>` for one vendor
(`test WaveShell`, `test AmpliTube`). Nonzero exit on any failure.

A `TIMEOUT` on WaveShell1 while WaveShell2 and WaveShell3 `PASS`, with a
non-empty `.wle` and no window, is the shell's size. A 60s budget is shorter
than the load. WaveShell1 13.0 (VST2 and VST3) enumerated 173 plugins in
105s with two jobs (verified 2026-09-27). The smoke test's default budget
is 180s for that reason. Confirm a leftover timeout with
`bin/omadaw-vst test --timeout 180 --jobs 1 WaveShell1` before treating it
as an activation dialog.
6. Waves plugins missing after Central → the Waves Central section above
   (PowerShell verb, pty launch, then `add` the VST2 dir and `sync`).

## Safety rules

- Never `export WINEPREFIX=~/.wine` for VST work; default is `~/.wine-omadaw`.
- Prefer `wine-staging` (Proton only when a plugin demonstrably needs it;
  note the reason in your summary).
- Packages via `omarchy pkg add` for repo packages and `omarchy pkg aur add`
  for AUR packages, including the bridge. Prefer the extra packages
  `yabridge` and `yabridgectl` when they match the installed Wine. When extra
  is still 5.1.1 and Wine is 9.22 or newer, plugin editors draw but mouse
  clicks miss; install `yabridge-git` and `yabridgectl-git` from the AUR,
  then `bin/omadaw-vst sync`. Paid DAWs such as Bitwig still need a yes
  before install.
- After an Omarchy update, re-run `bin/omadaw-vst sync` (wine/yabridge ABI
  bumps can stale shims) before deeper debugging.

## Waves plugins (verified Sep 2026, Central 17 / V13)

Waves installs only through Waves Central (Electron app) — no standalone
setup.exes. It runs in wine-staging but needs help:

- Prefix deps (add only on failure, diagnose first): PowerShell in the prefix
  (Central shells out to `where powershell`; wrapper or PS Core MSI + copy
  `pwsh.exe`→`powershell.exe`), `winetricks corefonts gdiplus vcrun2019`.
  Host needs `ntlm_auth` (`omarchy pkg add samba`) or login fails.
- Flow: `bin/omadaw-vst install Install_Waves_Central.exe` → launch Central
  → **user logs in** (browser handoff) and installs products → activate.
- **Activate to Local Disk C (this machine), never USB** — Wine cannot see
  USB license drives. Verify activation WITHOUT the GUI: the `.wle` in
  `drive_c/ProgramData/Waves Audio/Licenses/` must have a NON-EMPTY
  `<Licenses>` section. Empty section = shells enumerate zero plugins
  (no error anywhere — this is the failure mode).
- Keep `WavesLocalServer.exe` (in `ProgramData/Waves Audio/WavesLocalServer/
  …/Win64/`) running while scanning/loading plugins.
- Bridge both shell flavors: `bin/omadaw-vst add ".../Waves/WaveShells V13"`
  (VST2) plus the auto-registered `Common Files/VST3`, then `sync`. In Reaper,
  delete bare `WaveShell*=<id>` lines (no plugin names after `=`) from
  `reaper-vstplugins64.ini` and restart to force a re-scan of the shells.

## Vendor managers (Electron apps) under Wine

Waves Central and IK Product Manager are Electron apps: they install and run,
but their windows may render blank/white (GPU/rasterizer + `os-info`
failures). Launch with GPU rendering off — `run` forwards extra args:

```bash
bin/omadaw-vst run "drive_c/Program Files/IK Multimedia/IK Product Manager/IK Product Manager.exe" \
  --disable-gpu --disable-software-rasterizer --no-sandbox
```

(verified: IK PM login screen renders correctly with these flags, Sep 2026).
Prefer vendor managers over file-copy migration: copies bring no registry
entries, no authorization, and version soup (e.g. TR5 singles without the
T-RackS suite shell spam activation popups at scan). Copy only as scaffolding
(`bin/omadaw-migrate`), then install properly and remove the copies.

## Vendor managers under Wine (verified Sep 2026)

- **Waves Central** (Electron): needs PowerShell in the prefix + `ntlm_auth`
  on the host (`omarchy pkg add samba`) for login. Products install to the
  prefix; activate to Local Disk C, never USB (invisible under Wine).
- **IK Product Manager** (Electron): renders blank unless launched with
  `--disable-gpu --disable-software-rasterizer --no-sandbox` (forwarded via
  `bin/omadaw-vst run <exe> [args...]`). Login works, downloads install
  in-prefix. Download path may follow migrated settings into a second
  `drive_c/users/<name>` profile — harmless, leave it.
- **Native Access** (Electron, verified 2026-09-27, wine-staging 11.17).
  Service Center is retired. `servicecenter.native-instruments.com` resolves
  and does not answer on port 443, and the old `/servicecenter/*.xml` URLs
  are 404. Wine can still reach `www.native-instruments.com`. Activate in
  Native Access (`Native-Access_2.exe`, NSIS, via `bin/omadaw-vst install`).
  The app is `drive_c/Program Files/Native Instruments/Native Access/Native Access.exe`.
  The login screen renders without the IK Product Manager GPU flags.

  Login finishes in the **host** browser (`auth.native-instruments.com` →
  `na-cloud.native-instruments.com/.../auth-handler`), then redirects to
  `native-access://`. Wine registers that scheme
  (`HKCU\Software\Classes\native-access` → `Native Access.exe`), and the
  Linux browser does not, so the app stays on "Please log in" after a
  successful sign-in. Register a user handler, then log in again:

  ```bash
  mkdir -p ~/.local/bin ~/.local/share/applications
  cat > ~/.local/bin/native-access-url <<'EOF'
  #!/bin/sh
  # Electron crashes with open EBADF unless stderr is a pty.
  export WINEPREFIX="${WINEPREFIX:-$HOME/.wine-omadaw}"
  export WINEDEBUG="${WINEDEBUG:--all}"
  export DISPLAY="${DISPLAY:-:0}"
  export WAYLAND_DISPLAY="${WAYLAND_DISPLAY:-wayland-1}"
  export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"
  export NA_URL="$1"
  exec script -q -f -e -c \
    'wine "C:\Program Files\Native Instruments\Native Access\Native Access.exe" "$NA_URL"' \
    /tmp/native-access-url.typescript
  EOF
  chmod 755 ~/.local/bin/native-access-url
  cat > ~/.local/share/applications/native-access-url.desktop <<EOF
  [Desktop Entry]
  Type=Application
  Name=Native Access
  NoDisplay=true
  Exec=$HOME/.local/bin/native-access-url %u
  MimeType=x-scheme-handler/native-access;
  Terminal=false
  EOF
  update-desktop-database ~/.local/share/applications
  xdg-mime default native-access-url.desktop x-scheme-handler/native-access
  ```

  A handler that runs `wine` directly, with stderr a pipe, shows "A JavaScript
  error occurred in the main process": `open EBADF` in
  `createWritableStdioStream` / `process.stderr`. Same failure as Waves
  Central. `script -q -f -e -c` is required. After the handler is in place,
  the existing "Signing in…" tab can be reopened; a fresh Login click also
  works. Do not print the auth-handler URL. It carries the session token.

  Guitar Rig 5 (verified 2026-09-27) then crashes on launch. The dialog names
  `Documents\Native Instruments\Guitar Rig 5\Crashlogs\*-mini.nicrash`.
  That file is a minidump, exception `0x80000100` (`EXCEPTION_WINE_STUB`) in
  Wine's builtin `msvcr120.dll` for
  `Concurrency::details::_AsyncTaskCollection::_NewCollection`. The VC++ 2013
  concurrency runtime is not implemented. Install the native redist and
  relaunch. Do not treat this as a missing ISO mount:

  ```bash
  WINEPREFIX=~/.wine-omadaw winetricks -q vcrun2013
  ```

  That sets `msvcr120`, `msvcp120`, `atl120`, and `vcomp120` to
  `native,builtin`. The standalone then opens
  (`Guitar Rig 5.exe`, window title `Guitar Rig 5 - Native Instruments`).

- **Toontrack Product Manager** (Qt): renders natively, no flags needed.
  It stages installers under `~/Downloads/Toontrack/` (tens of GB for SDX
  libraries — watch disk; clean up after). Sound libraries land in
  `drive_c/ProgramData/Toontrack/`.
- **XLN Online Installer 5.0.0** (verified 2026-09-26, wine-staging 11.17).
  Two separate failures:

  1. The build imports `d2d1`/`dcomp` and hangs at `before filterWindow`
     with no window (Direct2D/wined3d deadlock). Do not disable those DLLs
     (hard imports; the process exits 53). Force a software renderer for
     this exe only:

     ```bash
     WINEPREFIX=~/.wine-omadaw wine reg add \
       "HKCU\Software\Wine\AppDefaults\XLN Online Installer.exe\Direct3D" \
       /v renderer /t REG_SZ /d no3d /f
     ```

  2. Once the window is up, a self-update relaunches
     `Temp\App\Cotton XLN Online Installer\updateBinary\XLN Online Installer.exe`
     and then `remove_all`s that running file. Wine returns
     `STATUS_CANNOT_DELETE` (dialog: `boost::filesystem::remove_all: Access
     denied [system:5]`, destination NULL) because the image is mapped.
     Do not relaunch that `updateBinary` path. After the payload has landed,
     start the copy that is not the mapped updater:

     ```bash
     WINEPREFIX=~/.wine-omadaw wine \
       "$HOME/.wine-omadaw/drive_c/ProgramData/XLN Audio/XLN Online Installer/Run/XLN Online Installer.exe"
     ```

  The window title is `XLN Audio - XLN Online Installer`. Leave it for the
  user to sign in. Offline per-product installers via `bin/omadaw-vst install`
  are the fallback if that window still does not appear. Do not rename
  `dont_trace.txt` to `do_trace.txt` except while debugging; tracing writes
  multi-megabyte logs under `Documents/XLN Online Installer Logs/`.

- **Plugin GUI mouse on Wine 11 + yabridge 5.1.1** (verified 2026-09-26,
  Addictive Drums 2 in Reaper, Hyprland 0.56, XWayland). The editor opens and
  draws, but clicks miss. yabridge 5.1.1 does not track window position on
  Wine 9.22 and newer, so the plugin treats the window as if it were at the
  X11 origin. Hyprland's global `0,0` is not that origin on this layout: the
  left monitor is X11 `x=-1920`, and X11 `0,0` is the top-left of `DP-1`
  (Hyprland `1920,0`). Confirm with `xdotool getwindowgeometry` (`X=0`, `Y=0`)
  before telling the user to click. Moving the window away from that corner
  brings the miss back. Fix it for every plugin by installing the AUR master
  builds, which include the Wine 10 embedding changes, then syncing:

  ```bash
  omarchy pkg aur add yabridge-git yabridgectl-git
  bin/omadaw-vst sync
  ```

  Those packages `conflict` with the extra `yabridge` and `yabridgectl`
  packages. Remove the extra packages first, then install the built AUR
  packages. On Wine 11.17 the stock AUR `yabridge-git` PKGBUILD fails to
  link (`__wine$func$...` undefined) because Arch passes `-flto=auto`.
  Build with `options=('!lto')`, strip `-flto=auto` from `CFLAGS`,
  `CXXFLAGS`, and `LDFLAGS`, and set `-Dbitbridge=false` unless 32-bit
  Wine import libraries are installed. The working build here is
  `yabridge-git 5.1.1.r57.gb580a9f7`.

  Do not downgrade `wine-staging` to 9.21 to chase this; this prefix is on
  11.17 for the rest of the stack.

- **JUCE plugin GUI draws but never updates** (verified 2026-09-26, IK
  Multimedia TONEX under yabridge-git and Wine 11.17). Clicks can land and
  the audio can change while the editor stays frozen. That is WineD3D not
  redrawing the JUCE surface. Install DXVK into the prefix and reopen the
  plugin so `yabridge-host.exe` starts again:

  ```bash
  WINEPREFIX=~/.wine-omadaw winetricks -q dxvk
  ```

  That copies native `d3d11.dll` and `dxgi.dll` into the prefix and sets
  `HKCU\Software\Wine\DllOverrides` to `native`. The already-running host
  keeps WineD3D until the editor is closed and opened again. DXVK is
  prefix-wide, so every later `yabridge-host.exe` uses it. A plugin that
  is already open stays on WineD3D until that host process starts again.

- **Dropdowns are separate Hyprland windows** (verified 2026-09-26, Hyprland
  0.56, XWayland, yabridge-git `5.1.1.r57`). yabridge does not draw the menu
  inside the plugin surface. Each Win32 popup is its own floating XWayland
  client: class `yabridge-host.exe`, title `menu`. Hyprland then runs its
  open animation on it. The white frame is not a title bar. It is Hyprland's
  2px border plus four empty-titled strips (about 12px) that sit on the
  menu's edges. The pointer jumps onto a child menu because Hyprland warps
  on focus, and it sticks because Wine grabs the pointer.

  Edit the user Hyprland config (`~/.config/hypr/`, Omarchy's `o.window`
  helper). Do not edit `/usr/share/omarchy/`. Afterward run `hyprctl reload`
  and `hyprctl configerrors`.

  ```lua
  -- ~/.config/hypr/looknfeel.lua
  hl.config({
    cursor = {
      no_warps = true,
      warp_on_change_workspace = 2,
    },
  })
  ```

  ```lua
  -- ~/.config/hypr/hyprland.lua
  o.window({
    class = "^yabridge-host\\.exe$",
    xwayland = true,
    float = true,
  }, {
    no_anim = true,
    no_shadow = true,
    no_blur = true,
    no_dim = true,
    decorate = false,
    rounding = 0,
    border_size = 0,
    tag = "-default-opacity",
    opacity = "1 override",
  })

  -- Wine closes a dropdown when the plugin editor stops being the active
  -- window. Hyprland's focus passes through "nothing" on the way to the
  -- menu, and that gap is enough. suppress_event "activatefocus" is the
  -- real switch; a focus_on_activate field on the window rule is ignored.
  o.window({
    class = "^yabridge-host\\.exe$",
    title = "^menu$",
    xwayland = true,
    float = true,
  }, {
    no_follow_mouse = true,
    no_initial_focus = true,
    suppress_event = "activatefocus",
  })

  o.window({
    class = "^yabridge-host\\.exe$",
    title = "^$",
    xwayland = true,
    float = true,
  }, {
    no_focus = true,
    no_initial_focus = true,
    no_follow_mouse = true,
    suppress_event = "activatefocus",
  })

  o.window({
    class = "^yabridge-host\\.exe$",
    title = "^$",
    xwayland = true,
    float = true,
  }, {
    opacity = "0 override",
  })
  ```

  Do not move those blank strips off-screen. yabridge treats the pointer
  leaving the plugin window as a reason to drop keyboard focus
  (`LeaveNotify` → `set_input_focus(false)`), and a dropdown closes when
  that happens. Hiding the strips with `opacity = "0 override"` is enough.
  `suppress_event = "activatefocus"` on title `menu` keeps the popup from
  becoming the active window. A `focus_on_activate` field on the window
  rule is ignored. The close is intermittent because Hyprland sometimes
  drops focus to nothing while the menu is being mapped, and Wine treats
  that as the editor going away.

  Also stop Wine from grabbing the pointer. Reopen the plugin afterward so
  `yabridge-host.exe` starts again:

  ```bash
  WINEPREFIX=~/.wine-omadaw wine reg add \
    "HKCU\Software\Wine\X11 Driver" /v GrabPointer /t REG_SZ /d N /f
  ```

  Reaper's own windows stay class `REAPER` and are not matched. `no_warps`
  does not stop workspace changes from moving the cursor when
  `warp_on_change_workspace` is `2`.
