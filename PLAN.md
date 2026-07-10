# Home lab rebuild: NUC (CLI + LLM) + HP Mini (Frigate) + Pi 5 (HAOS) — all bare-metal

> **Execution:** each phase has a self-contained implementation prompt in [`prompts/`](prompts/00-README.md). Start with `prompts/00-README.md`.

## Context

The three-machine split (NUC / Raspberry Pi 5 / HP Pro Mini 400 G9) and the real HA audit (144 devices, 1,752 entities, 58 integrations, 51-device Zigbee mesh, Whisper/Piper voice, 419MB recorder DB) were established in an earlier pass — see git history for that reasoning. Since then, two further simplifications landed:

- **No desktop/browser on the NUC.** A separate laptop now runs Claude Desktop, so the NUC's "workstation VM" (XFCE + Chrome + Claude Desktop) has no remaining justification — HA (Pi 5) and Frigate (HP Mini) both have their own web UIs, reachable from that laptop over the network regardless of what runs on the NUC.
- **No virtualization anywhere.** Once Frigate moved to the HP Mini and HA moved to the Pi 5, each x86 machine ended up hosting exactly one workload: NUC = Claude Code CLI + local LLM inference, HP Mini = Frigate. Virtualization earns its keep multiplexing several isolated workloads on shared hardware — with one workload per box, Proxmox on either machine is pure overhead: an LXC/VM passthrough layer to fight with, a hypervisor to patch, and (on the NUC) a PVE 8→9 upgrade that no longer needs to happen at all. Snapshots/PBS were already deprioritized in favor of raw LLM performance, so the main everyday argument for keeping a hypervisor didn't apply here either. Both machines go **bare-metal Debian 13**.

Decision: **teardown and clean rebuild across three machines, all bare-metal.** Home Assistant moves to the Pi 5. Frigate moves to a fresh bare-metal Debian + Docker install on the HP Mini. The NUC gets wiped from Proxmox to bare-metal Debian 13, running Claude Code CLI natively and IPEX-LLM natively for two local models (vision model for Frigate event captioning via `llmvision`, general chat model). The current Coding VM (this session's host) does not survive the NUC wipe — its data gets salvaged first, and Claude Code resumes as a fresh session on the new bare-metal install afterward.

## Target end state

| Machine | OS | IP | Role |
|---|---|---|---|
| Raspberry Pi 5 | HAOS (bare-metal appliance) | .101 | Home Assistant: Zigbee2MQTT, Mosquitto, Matter, ESPHome, voice (Whisper/Piper), Samba, Tailscale |
| HP Pro Mini 400 G9 | Debian 13 + Docker | .102 | Frigate: 4 cameras, VAAPI decode + OpenVINO GPU detection on UHD 770 |
| NUC 14 Pro | Debian 13 (native) | .103 | Claude Code CLI + IPEX-LLM (Arc iGPU): vision model kept warm for `llmvision` camera captioning, + a 7–14B chat/coding model |

No hypervisor on any machine. Both Proxmox hosts (NUC `.100`, HP Mini `.110` in the old scheme) go away — plain Debian gets a single static reservation each. Static DHCP reservations for all three machines plus the 4 cameras.

Sizing is simpler with nothing to split between guests: NUC uses its full 32GB/all cores natively for the CLI + both models; HP Mini uses its full 32GB/all cores for Frigate (vast headroom — no NAS/PBS role assigned, per earlier deferral).

## Phase 0 — NUC: SSH access to the *current* Proxmox host (temporary, pre-wipe only)

Salvage has to happen while the existing Proxmox install is still up, before anything is wiped.

1. Check/generate SSH keypair on this Coding VM.
2. Lawrie pastes one provided command into the Proxmox web UI → NUC node → Shell to authorize the key.
3. Verify `ssh root@192.168.68.100` from here.

## Phase 1 — Audit + salvage everything that dies with Proxmox

- Host audit: `pveversion`, `qm list`/`pct list`, storage layout, per-guest configs.
- **Salvage HA**: trigger a **full** backup from the old HAOS VM (add-on data, Zigbee2MQTT device database, Matter fabric certs, ESPHome secrets, voice config, recorder). Copy the `.tar` off-host (to this VM and to GitHub or another machine — anywhere that survives the NUC wipe).
- **Salvage Frigate**: copy `config.yml` (camera URLs/credentials, zones/masks) off the old container.
- **Salvage this Coding VM itself** — it does not survive Phase 2: `~/.claude/` (memory, settings), `~/claude/` projects, `~/nuc-rebuild` (already on GitHub — verify it's fully pushed), `~/awa`, SSH keys, `.gitconfig`/`.git-credentials`, MCP server configs/tokens, npm global setup. Push/copy all of it somewhere that isn't the NUC.
- `vzdump` the old HAOS VM and old Frigate container to host storage as a last-resort archive, space permitting.

## Phase 2 — NUC: wipe Proxmox, install bare-metal Debian 13

**Physical step, done by Lawrie, whenever convenient — this ends the current Claude Code session.**

- Boot a Debian 13 installer USB on the NUC, wipe the disk, install Debian 13 fresh, static IP **.103**.
- Basic hardening (SSH key auth, updates) and re-establish this VM's salvaged SSH key for remote access.
- A **new** Claude Code CLI session starts here afterward and restores the salvaged `~/.claude/`, `~/claude/`, and other data from Phase 1.

## Phase 3 — HP Mini: install bare-metal Debian 13 + Docker

No wipe needed — Proxmox was never installed on this machine, so this is a first install, not a migration.

- Debian 13 installer, static IP **.102**, Docker installed directly on the host (no LXC/VM layer).
- Same SSH-key-authorization pattern as Phase 0/2, from the (new) NUC session to the HP Mini.

## Phase 4 — Raspberry Pi 5: Home Assistant OS

- Lawrie upgrades the Pi 5 from microSD to NVMe (recommended given recorder write volume), flashes the official HAOS Raspberry Pi 5 image, boots it. Static reservation **.101**.
- Restore the Phase 1 **full** backup → Zigbee2MQTT reconnects automatically (network-attached coordinator, no re-pairing needed), Matter/ESPHome/voice/Samba/automations/HACS all return. Tailscale may need manual re-auth.
- Add `recorder:` `exclude`/`purge_keep_days` tuning — the Pi 5's fixed 8GB has no headroom for the current install's unbounded recorder growth.
- Drop the VS Code/code-server add-on now that Claude Code covers dev work from the NUC.

## Phase 5 — HP Mini: Frigate on the UHD 770

- Docker Compose Frigate, `/dev/dri/renderD128` mounted directly (no passthrough config needed — it's the host's own device now).
- Rebuild `config.yml` from salvaged camera credentials: `go2rtc` one connection per camera; `ffmpeg: hwaccel_args: preset-vaapi`; detector `openvino` on `GPU` (UHD 770); MQTT → the Pi 5's Mosquitto at `.101`; keep salvaged zones/masks.
- Verify: `intel_gpu_top` shows video+render load; Frigate UI inference speed; low CPU.

## Phase 6 — HA ↔ Frigate integration

- HACS + Frigate integration in HA (Pi 5) pointed at Frigate (HP Mini) → cameras, events, sensors.
- Camera dashboard + one worked example automation (person detected → phone notification with snapshot).

## Phase 7 — NUC: Claude Code CLI + IPEX-LLM

- Restore Claude Code CLI setup from the Phase 1 salvage (native install, no container).
- Install IPEX-LLM (Intel's Xe-optimized inference stack, SYCL backend) natively — `/dev/dri` is the host's own device, no passthrough config.
- Two models: a small vision-capable model (e.g. Moondream2 or Qwen2-VL-2B) **kept warm** for `llmvision` camera captioning, and a separate 7–14B model for general chat/coding. Verify both generate on GPU (not CPU fallback) via `intel_gpu_top` during inference.
- Wire the vision model into Frigate/HA event handling: on a new Frigate event, call the local Ollama-compatible API for a description, feed it into the notification pipeline built in Phase 6.

## Phase 8 — Verification

- End-to-end walk test spanning all three machines: Frigate (HP Mini) detects a person → MQTT to HA (Pi 5) → HA calls `llmvision` → hits the NUC's LLM stack for a description → phone notification with snapshot + description.
- All three machines' relevant services (`systemd` units, Docker containers) auto-start after a reboot test on each.
- Confirm nothing from the old Coding VM is unmigrated before considering the NUC done.

## Phase 9 — Automated maintenance (ongoing, recurring)

Sets up a weekly cron job that keeps the fleet patched without drifting back into an unmanaged pile:

- **Fully automatic, no LLM in the loop**: `unattended-upgrades` for OS-level security patches on all three machines (now just plain `apt` — no Proxmox version to track).
- **Weekly scheduled Claude Code run** (Linux cron on the NUC — not Anthropic's cloud scheduler, since this needs LAN access): checks HA's Core/Supervisor/OS/add-on/HACS update entities via REST API and Frigate's Docker image version. Auto-applies the low-risk subset (HACS integration updates, Frigate patch-version bumps) under a narrowly scoped permissions allowlist. Everything else — HA Core/Supervisor/OS updates, Frigate major bumps — gets reported and pushed as a phone notification for manual approval.
- **Never auto-applied**: Zigbee coordinator and ESPHome device firmware (bricking risk), HA Core major versions.

## Deferred

- NAS build / backups — explicitly deferred in favor of LLM performance and simplicity; ad hoc `tar`/`rsync` snapshots of config directories remain the interim safety net (no Proxmox vzdump anymore).
- Google Coral Dual Edge TPU M.2 accelerator — no compatible slot on any of the three machines.
- NPU experiment (OpenVINO `NPU` device on the NUC's Meteor Lake NPU) once the GPU baseline for both LLMs is proven — no passthrough concerns now, it's just another local device.

