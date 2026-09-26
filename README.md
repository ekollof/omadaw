# OmaDAW — low-latency audio production on Omarchy

Setup for realtime PipeWire audio, Windows VSTs (dedicated Wine prefix +
yabridge), and Bitwig / Reaper / Ardour routing on Omarchy.

## Quick start

`omadaw` prepares the system, checks its health, and drops you into the
default agent harness (repo skills in context) on request:

```bash
./omadaw --profile studio --with-daw all --yes   # set up (idempotent)
./omadaw doctor                                  # health check
./omadaw agent "set up for a USB interface at 128 frames with Reaper"
```

## What you get

- **Realtime:** `realtime-privileges` + `rtkit`, user in `realtime` group.
- **PipeWire profiles:** `studio` 128 / `balanced` 256 / `safe` 512 frames @ 48 kHz
  (`config/pipewire/...`, switch live with `bin/omadaw-route profile <name>`).
- **Windows VSTs:** dedicated `~/.wine-omadaw` prefix (not `~/.wine`) + yabridge:
  ```bash
  bin/omadaw-vst install ~/Downloads/SomePluginSetup.exe
  bin/omadaw-vst sync          # discover new/updated plugins
  bin/omadaw-vst enable-watch  # background auto-sync on new files
  bin/omadaw-vst doctor
  ```
- **DAWs:** Reaper + Ardour from repos, Bitwig from AUR (asks first);
  per-DAW backend/plugin notes in `config/daw/`; launch with
  `bin/omadaw-route launch <bitwig|reaper|ardour>`; patch with `qpwgraph`.
- **Rollback:** `./omadaw --snapshot ...` snapshots system (snapper) +
  user state (`~/.wine-omadaw`, pipewire conf, shims) first; retest with
  `bin/omadaw-snapshot rollback-system` + `rollback-user latest`.

## Troubleshooting

- **Running out of disk during big sound-library installs:** move the library
  directory to a roomier disk and symlink it back, e.g.
  `mv ~/.wine-omadaw/drive_c/.../Toontrack /games/soundlibs/ && ln -s
  /games/soundlibs/Toontrack <original-path>`. Wine follows the link, so all
  `C:\…` paths keep working with nothing to reconfigure. Stop the apps using
  the files first.

## Repo layout

See `AGENTS.md` (agent handbook) for the full map, Omarchy conventions, and
verification commands. Agent skills live in `skills/omadaw-audio/` and
`skills/omadaw-vst/`.
