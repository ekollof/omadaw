# Ardour (Omarchy / OmaDAW)

- Install: `omarchy pkg add ardour` (in extra). The package binary is
  `ardour9` (older builds used `ardour8`); there is often no `ardour` on
  `PATH`. `bin/omadaw-route` resolves `ardour`, then `ardour9`, `ardour8`,
  `ardour7`.
- Launch: `bin/omadaw-route launch ardour` (wraps with `pw-jack` so the
  JACK backend routes through PipeWire and stays visible in qpwgraph).
- On first run Ardour asks for the audio backend:
  - JACK (via PipeWire) — recommended; shares the device with other apps.
  - ALSA — exclusive device access, lowest raw latency, invisible to patchbay.
- Plugin paths: Edit → Preferences → Plugins → VST2/VST3/LV2. Ensure
  `~/.vst`, `~/.vst3`, `~/.lv2` are listed; bridged Windows plugins show up
  after `bin/omadaw-vst sync`.
- Sample rate: match PipeWire (`default.clock.rate = 48000` in the OmaDAW
  profile) to avoid resampling. Check with `pw-metadata -n settings`.
- Multiple / multi-channel interfaces: every input appears as its own PipeWire
  node. List them with `bin/omadaw-route interfaces`, pick defaults with
  `bin/omadaw-route set-default`, and route them in qpwgraph.
