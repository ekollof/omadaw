# OmaDAW — AGENTS.md

> Read this first. It exists so you don't rediscover the Omarchy audio stack from scratch.

## What this repo is

OmaDAW sets up an Omarchy (Arch-based, Hyprland) machine for low-latency audio
production: realtime PipeWire, a dedicated Wine prefix + yabridge bridge for
Windows VSTs, and routing/config for Bitwig, Reaper, Ardour.

## How to work here

- **Deterministic setup first:** `./omadaw --profile studio --with-daw all --yes`
  is idempotent — re-run it instead of hand-rolling package/config steps.
- **Agent-driven setup:** `./omadaw --agent "<goal>"` delegates to the
  Omarchy default agent (`omarchy agent prompt`, resolved via
  `omarchy-default-agent`) with this repo as context. Use it when the task needs
  inspection (interface quirks, which DAWs, VST inventory).
- **Diagnose first, then fix:**
  `./omadaw doctor` → `bin/omadaw-route status` → `bin/omadaw-vst doctor`.
  `doctor` fast-forwards this repo when the worktree is clean, reinstalls a
  changed PipeWire profile and an already-enabled VST watch unit, and prints
  any `AGENTS.md` or `skills/` files to re-read before acting. `./omadaw
  --check` is the same health check without the fetch.
- **Skills (load the matching one before acting):**
  - `skills/omadaw-audio/SKILL.md` — PipeWire quantum/rate, realtime group,
    DAW routing, xruns. Triggers: latency, xruns, no DAW sound, device routing.
  - `skills/omadaw-vst/SKILL.md` — Wine prefix, yabridge install/sync/discovery.
    Triggers: VST, yabridge, setup.exe, plugin missing from DAW, Waves Central.
  - The `omarchy` skill (if present) governs desktop/window config — out of scope
    here except `omarchy audio ...` sink/source helpers.
- **This machine:** read `MEMORY.md` before rediscovering interfaces, which
  DAWs are installed, or the plugin inventory. Record machine-specific
  findings there. Skills stay general.
- **Contribute general quirks:** when a failure would happen on another
  OmaDAW machine, add it to the matching `skills/*/SKILL.md` in the same
  change as the fix. Symptom, cause, and the command that fixes it. The
  Native Access `native-access://` handler and the Guitar Rig 5 `vcrun2013`
  crash are the examples. Do not put hostnames, serials, license files, or
  this install's inventory in the skill or the commit.

## Repo map

| Path | Purpose |
|---|---|
| `omadaw` | Full installer: realtime → PipeWire profile → Wine/VST prefix → DAWs → watch unit (`--snapshot` first for rollback) |
| `bin/omadaw-snapshot` | Rollback: `create`, `list`, `rollback-user`, `rollback-system`, `delete` |
| `bin/omadaw-track` | Tracking mode: `on` (Reaper → exclusive ALSA hw, ~3ms), `off` (desktop), `status` |
| `bin/omadaw-vst` | VST wrapper: `init`, `install <setup.exe>`, `run`, `add`, `sync`, `list`, `doctor`, `enable-watch` |
| `bin/omadaw-route` | Routing: `status`, `profile`, `launch <daw>`, `set-default`, `profiles`/`set-profile`, `set-channels`, `setup-daw-config` |
| `config/pipewire/pipewire.conf.d/10-omadaw-{studio,balanced,safe}.conf` | Quantum/rate profiles (128/256/512 @ 48 kHz) |
| `config/daw/{bitwig,reaper,ardour}.md` | Backend + plugin-path notes per DAW |
| `config/systemd/user/omadaw-vst-sync.{path,service}` | New/updated-VST auto-discovery unit |
| `skills/*/` | Agent skills (this file points at them; they own the details) |
| `MEMORY.md` | This machine only: interface, installed DAWs, plugin inventory, incidents |

## Facts already verified (don't re-derive; re-check only if suspect)

- Omarchy base ships `pipewire pipewire-alsa pipewire-pulse pipewire-jack
  wireplumber`; default quantum 1024 @ 48 kHz. `pipewire-jack` provides `pw-jack`.
- RT = `realtime-privileges` (extra) + `rtkit`: user in `realtime` group,
  limits at `/etc/security/limits.d/99-realtime-privileges.conf` (`rtprio 98`,
  `memlock unlimited`). **Advise the user to reboot, or to log out and back
  in, before treating realtime as active.** `id $USER` is the account
  database and can list `realtime` while this session still has `ulimit -r`
  of 0. Check `id` with no username and `ulimit -r` (want `98`). A plain
  logout often leaves `systemd --user` running (`KillUserProcesses=no`), and
  Hyprland/PipeWire keep the old groups; a reboot always applies them. Do
  not try to grant the group to a running user manager.
- Bridge = `yabridge` + `yabridgectl` 5.1.x (extra), runner `wine-staging` 11.x.
  Prefix is `~/.wine-omadaw` — never `~/.wine`. Shims live in `~/.vst`, `~/.vst3`.
- DAWs: `reaper`, `ardour` in extra; `bitwig-studio` in AUR (proprietary — ask
  before installing). Patchbay: `qpwgraph` (default) / `helvum`.
- Audio backend policy (verified 2026-09-24, Reaper 7.79 / PipeWire 1.6.8):
  shared-graph Reaper (JACK, `linux_audio_mode=0`, or PipeWire ALSA) is valid
  once `ulimit -r` is 98. If it is 0, rtkit gives PipeWire's in-process
  `data-loop.0` thread realtime at priority 1 and sets `RLIMIT_RTTIME`. The
  kernel then SIGKILLs that thread (`si_code=SI_KERNEL` / 128) and Reaper
  dies with no coredump and no userspace `kill`. **Tell the user to reboot
  or relogin; do not debug this as a Reaper or JACK bug and do not switch
  them to PulseAudio to "stop the killing".** After a reboot, `data-loop.0`
  is `SCHED_FIFO`, `RLIMIT_RTTIME` is unlimited, and Reaper stays up.
  PulseAudio (`linux_audio_mode=3`) is the high-latency desktop path.
  Exclusive ALSA is `bin/omadaw-track` (other apps lose the interface).
- Snapshots are split: `omarchy snapshot` (snapper, config `root`) covers `/`
  but **not** `/home`. `bin/omadaw-snapshot` pairs it with a user-state bundle
  (`~/.local/share/omadaw/snapshots/`: wine prefix, pipewire conf, shims).
  Full rollback = `rollback-system` (reboot via limine) + `rollback-user latest`.
- Interfaces are generic: never hardcode a device name, vendor, or channel
  count. Enumerate with `bin/omadaw-route interfaces` (or `wpctl status`) and
  select with `bin/omadaw-route set-default`. Setups may have zero, one, or
  several interfaces.

## Omarchy conventions (mandatory)

- Packages: `omarchy pkg add <repo-pkgs>` / `omarchy pkg aur add <aur-pkgs>`.
  Fall back to `pacman`/`yay` only when `omarchy` is absent.
- Default agent: `omarchy-default-agent` prints it; run it via
  `omarchy agent prompt "<task>"`. Empty output = none set →
  `omarchy default agent <name>`.
- Privilege: `sudo` in a visible terminal; `pkexec` only for headless/agent
  background work. Never `sudo` commands that self-elevate.
- **Never edit `/usr/share/omarchy/`** (read-only, overwritten on update).
  User state: `~/.config/pipewire/...`, `~/.config/systemd/user/...`, `~/.vst{,3}`.

## Verification

```bash
bash -n omadaw bin/omadaw-vst bin/omadaw-route
./omadaw --check
bin/omadaw-route status
bin/omadaw-vst doctor
bin/omadaw-snapshot list
pw-metadata -n settings | grep -E "clock.(rate|quantum)"
```

Keep changes small, scripts `shellcheck`-clean, and update the relevant
`skills/*/SKILL.md` when you learn a new failure mode that applies to any
OmaDAW machine. That update is part of finishing the task, not an optional
note. Findings about this install go in `MEMORY.md`, which is gitignored.
