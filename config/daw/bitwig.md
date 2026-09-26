# Bitwig Studio (Omarchy / OmaDAW)

- Install: `omarchy pkg aur add bitwig-studio` (proprietary, license required).
  omadaw asks before installing.
- Launch: `bitwig-studio` or `bin/omadaw-route launch bitwig`.
- Audio backend: Settings → Audio → ALSA (direct, lowest latency) or
  JACK (routes through PipeWire, visible in qpwgraph). Prefer JACK when you
  want to record/routing alongside other apps.
- Plugin locations (Settings → Locations → Plug-ins), ensure present:
  - VST3: `/home/<you>/.vst3`
  - VST2: `/home/<you>/.vst`
  - CLAP: `/home/<you>/.clap`
- Windows VSTs appear here automatically after `bin/omadaw-vst sync`
  (yabridge drops `.so` shims into `~/.vst` / `~/.vst3`).
- Low latency: use the `studio` PipeWire profile (128 frames). If xruns:
  `bin/omadaw-route profile balanced`.
