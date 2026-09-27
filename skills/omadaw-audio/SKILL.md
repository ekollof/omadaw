---
name: omadaw-audio
description: >
  Manage the OmaDAW low-latency audio stack on Omarchy (PipeWire quantum/rate,
  realtime privileges, DAW routing for Bitwig/Reaper/Ardour). Use for audio
  setup, tuning, updating, and fixing xruns, missing devices, or silent DAWs.
  Triggers: pipewire latency, quantum, xruns, realtime group, rtkit, no sound
  in DAW, USB/interface routing, qpwgraph, Bitwig/Reaper/Ardour audio backend.
---

# OmaDAW Audio Skill

You manage an Omarchy (Arch-based) audio-production setup. The repo owns the
truth — do not rediscover package names or config paths from the web first.

## Repo layout (read before acting)

- `omadaw` — idempotent installer. Prefer running it over hand-rolling steps.
- `bin/omadaw-route` — runtime helper: `status`, `profile`, `launch`, `set-default`.
- `config/pipewire/pipewire.conf.d/10-omadaw-{studio,balanced,safe}.conf` — profiles.
- `config/daw/{bitwig,reaper,ardour}.md` — per-DAW backend + plugin-path notes.
- Live state: `~/.config/pipewire/pipewire.conf.d/10-omadaw.conf` (active profile copy).

## Baseline facts (Omarchy, verified Sep 2026)

- PipeWire 1.6.x + WirePlumber, `pipewire-jack` provides JACK compat (`pw-jack`).
- Base packages already ship: `pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber`.
- `realtime-privileges` (extra) owns RT: the **session** must be in the
  `realtime` group. Limits file:
  `/etc/security/limits.d/99-realtime-privileges.conf` (`rtprio 98`,
  `memlock unlimited`). `rtkit-daemon` should be active.
  **When `ulimit -r` is 0, stop and tell the user to reboot, or to log out
  and back in.** Say that explicitly; do not work around it with config
  edits, a different Reaper backend, or `sg`/`sudo`. `id $USER` listing
  `realtime` does not mean this session has it. Confirm with `id` (no
  username) and `ulimit -r` printing `98`. A logout that leaves
  `systemd --user` up (`KillUserProcesses=no`) does not apply the group —
  Hyprland and PipeWire are children of that manager. A reboot does. If
  they already logged out and `ulimit -r` is still 0, tell them to reboot
  (or, from a TTY, `loginctl terminate-user $USER` and then log in).
- Default PipeWire quantum is 1024 @ 48 kHz (~21 ms) — too slack for production.
  OmaDAW profiles: `studio` 128 (~2.7 ms), `balanced` 256 (~5.3 ms), `safe` 512 (~10.7 ms).

## Workflows

### Diagnose (always start here)

```bash
./omadaw doctor
bin/omadaw-route status
pw-metadata -n settings | grep -E "clock.(rate|quantum)"
wpctl status
systemctl --user status pipewire pipewire-pulse wireplumber --no-pager
```

### Fix tree

1. **Session lacks realtime priority (`ulimit -r` is 0)** → if `id $USER`
   does not list `realtime`, `sudo usermod -aG realtime $USER`. Either way,
   **tell the user to reboot or log out and back in**, then re-check
   `ulimit -r` (want 98) before any other audio change. This is the Reaper
   SIGKILL: rtkit grants PipeWire's in-process `data-loop.0` thread realtime
   at priority 1 and arms `RLIMIT_RTTIME`; the kernel sends SIGKILL
   (`si_code=SI_KERNEL`, 128); no userspace killer and no coredump. Traced
   on Reaper 7.79 / PipeWire 1.6.8 (2026-09-24). After a reboot the same
   thread is `SCHED_FIFO`, `RLIMIT_RTTIME` is unlimited, and Reaper stays
   up on the shared graph. Do not "fix" this by switching Reaper to
   PulseAudio.
2. **Xruns / crackling** → step down one profile:
   `bin/omadaw-route profile balanced` (or `safe`), restart DAW, re-test.
   Also confirm `default.clock.rate` matches the interface/DAW project (48 kHz).
3. **DAW silent / device missing** →
   - Reaper: JACK (`linux_audio_mode=0`) or PipeWire ALSA shares the
     interface with other apps once `ulimit -r` is 98. PulseAudio
     (`linux_audio_mode=3`) is the high-latency desktop path. A SIGKILL
     with no coredump is the realtime-limit failure in step 1, not a reason
     to abandon JACK. See `config/daw/reaper.md`.
   - Ardour: ALSA vs JACK per `config/daw/ardour.md`. The extra package
     installs `ardour9`, not `ardour`. `bin/omadaw-route launch ardour`
     tries `ardour`, then `ardour9`, `ardour8`, and `ardour7`.
   - Bitwig: ALSA vs JACK per `config/daw/bitwig.md`.
   - Verify routing in `qpwgraph` (or `helvum`); move streams with
     `bin/omadaw-route set-default sink|source <name>`.
   - After any PipeWire restart, JACK clients do NOT re-register on their own:
     if the DAW has no ports in the graph (`pw-dump` shows no client node),
     relaunch it. Then check links: with JACK there is no device dropdown —
     inputs appear per-track (arm track → Input Mono → in1/in2…).
   - Wrong channel count (interface has 4ch, DAW shows 2): `bin/omadaw-route
     set-channels reaper` auto-matches to the default interface
     (`omadaw` does this too); restart the DAW to apply.
   - Omarchy-level sink switch: `omarchy audio output switch`, `omarchy audio source switch`.
4. **Multiple interfaces (or the wrong one is used)** → never assume which
   device is which; enumerate first:
   `bin/omadaw-route interfaces` (or `wpctl status`), then set the intended
   default with `bin/omadaw-route set-default <sink|source> <id-or-name>`.
   Notes: PipeWire shares all devices (unlike ALSA-exclusive mode, which grabs
   exactly one); every interface must run at the profile rate (48 kHz) or it
   gets resampled; USB interfaces that vanish after suspend need a re-plug +
   `systemctl --user restart pipewire wireplumber`, not a config change.
   Multichannel USB interfaces need the **Pro Audio** card profile or DAWs see
   the wrong channel count: `bin/omadaw-route profiles` to inspect,
   `bin/omadaw-route set-profile <card> pro-audio` to fix (`omadaw`
   offers this for duplex USB cards; WirePlumber remembers it per device).
4. **After Omarchy/system update broke audio** → re-run
   `./omadaw --profile <current> --with-daw <same> --yes`, then `--check`.
   Never hand-edit `/usr/share/omarchy/`; user config lives in `~/.config/`.

### Update / change profile

```bash
bin/omadaw-route profile <studio|balanced|safe>
# or persistently:
./omadaw --profile <name> --with-daw none --yes
```

### Retest from scratch (snapshot → install → rollback → install)

Omarchy snapshots cover `/` but not `/home`; `bin/omadaw-snapshot` handles both:

```bash
./omadaw --profile studio --with-daw all --yes --snapshot  # snapshots first
# ... verify, experiment ...
bin/omadaw-snapshot list
bin/omadaw-snapshot rollback-system   # prints reboot flow (limine menu or omarchy snapshot restore)
bin/omadaw-snapshot rollback-user latest  # restores prefix, pipewire conf, shims
./omadaw --profile studio --with-daw all --yes  # clean second run
```

## Safety rules

- Packages: `omarchy pkg add <repo-pkgs>`, `omarchy pkg aur add <aur-pkgs>`.
  Never raw `pacman -S` when `omarchy` is available (keeps policy consistent).
- Privilege: `sudo` in a visible terminal; `pkexec` only for headless/agent-launched
  background work. Never edit `/usr/share/omarchy/` (read-only, overwritten on update).
- Do not overwrite DAW user configs; `bin/omadaw-route setup-daw-config` only
  writes hints (`~/.config/omadaw-vst-paths.env`) and prints guidance.
- Paid software (Bitwig): ask before installing from AUR.
