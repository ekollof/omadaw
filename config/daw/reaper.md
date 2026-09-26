# Reaper (Omarchy / OmaDAW)

- Install: `omarchy pkg add reaper` (native Linux build, in extra).
- Launch: `reaper` or `bin/omadaw-route launch reaper` (plain — `pipewire-jack`
  is picked up automatically, no `pw-jack` wrapper needed).
- Audio backend: Preferences → Audio → Device → **JACK** (recommended once
  realtime is active — `ulimit -r` prints `98`; shared graph, ~2.6 ms at the
  studio profile, other apps keep working). Title bar shows e.g.
  `[48kHz 24bit WAV : 2/2ch 128spls ~2.6/2.6ms JACK]` when healthy.
  **PulseAudio** (`linux_audio_mode=3`) is the always-works fallback
  (higher latency, ~75 ms) — also the safe seed `omadaw` writes for
  first launch before the realtime reboot.
- Config keys (`~/.config/REAPER/reaper.ini`, `[reaper]` section):
  `linux_audio_mode=0` prefers JACK, `3` prefers PulseAudio. If the preferred
  backend's library is missing, Reaper silently falls back (e.g. mode `0`
  with no libjack runs ALSA instead) — so a surprising title-bar backend
  name means fallback, not a config error.
- ⚠️ Reaper dying with SIGKILL and no coredump, after seconds or a minute, is
  the kernel realtime limit, not a crash and not a userspace killer. PipeWire's
  `data-loop.0` thread inside Reaper is the target (`si_code=SI_KERNEL`).
  rtkit arms `RLIMIT_RTTIME` when `ulimit -r` is 0. **Tell the user to reboot
  or log out and back in** so the session joins the `realtime` group; do not
  treat this as a JACK bug. A logout that leaves `systemd --user` running
  does not apply the group — if `ulimit -r` is still 0 afterwards, tell them
  to reboot. Confirmed after reboot (2026-09-24): `data-loop.0` is
  `SCHED_FIFO`, max realtime timeout is unlimited, Reaper stayed up on the
  shared graph. Details: `skills/omadaw-audio/SKILL.md`.
- Plugin paths (Preferences → Plug-ins → VST):
  - `~/.vst;~/.vst3` (yabridge shims), plus `~/.clap`, `~/.lv2` as needed.
  - After adding Windows VSTs: `bin/omadaw-vst sync`, then in Reaper
    "Re-scan" (or "Clear cache/re-scan").
- Extensions (optional): `reapack`, `sws` — `omarchy pkg add reapack sws`.
- Latency note: the PulseAudio backend buffers (~tens of ms) — fine for
  mixing/editing. JACK at the studio profile is the tracking path (~2.6 ms
  round trip, verified 2026-09-24).
- **Tracking mode (guitar/bass, ~2.6/5.3ms):** `bin/omadaw-track on` stops
  PipeWire and gives Reaper the interface exclusively over bare ALSA
  (`linux_audio_mode=1`, `alsa_indev/outdev=hw:<card>`). Title bar reads e.g.
  `[48kHz 24bit WAV : 4/4ch 128spls ~2.6/5.3ms ALSA U192k]`. Desktop audio
  pauses while tracking; `bin/omadaw-track off` restores PipeWire + the
  PulseAudio config. Auto-detects the first USB capture card
  (`--device hw:X` overrides).
- If Reaper can't see a bridged plugin: run `bin/omadaw-vst doctor`,
  check 64-bit vs 32-bit (yabridge needs matching wine arch), then re-scan.
