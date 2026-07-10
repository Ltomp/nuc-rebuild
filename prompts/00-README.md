# Home lab rebuild — phase prompts for Claude Code (Sonnet)

These files implement `PLAN.md` one phase at a time, across **three physical machines, all bare-metal — no hypervisor anywhere**. Each file is a **complete, self-contained prompt** for a fresh Claude Code session running **Sonnet**. The session has none of the planning context, so every prompt repeats the facts it needs.

## How to use

1. Start a new Claude Code session (Sonnet) on the machine listed in the table below.
2. Paste the entire contents of the phase file as your first message (or, if this repo is checked out on that machine, tell Claude: *"Read prompts/phase-N-….md and follow it exactly."*).
3. Let the phase run to completion. Claude will stop and ask you before anything destructive — **read those confirmations carefully before saying yes.**
4. Do not start the next phase until the current phase's **Exit criteria** are all met and its report file exists.

| Phase | File | Run the session on | Interruption? |
|---|---|---|---|
| 0 | `phase-0-nuc-host-ssh.md` | Coding VM (`.180`, temporary — still exists pre-wipe) | No |
| 1 | `phase-1-nuc-audit-salvage.md` | Coding VM | No |
| 2 | `phase-2-nuc-bare-metal-install.md` | Coding VM, then physically on the NUC | **Yes — permanently. The NUC is wiped from Proxmox to bare-metal Debian 13; the Coding VM and this session cease to exist.** A fresh Claude Code session starts on the new install afterward. |
| 3 | `phase-3-hpmini-bare-metal-install.md` | New NUC session (drives the HP Mini over SSH once Debian is installed) | No, but needs Lawrie to physically install Debian 13 on the HP Mini first |
| 4 | `phase-4-pi5-haos.md` | NUC session (drives the Pi 5 over SSH once flashed) | No, but needs Lawrie to physically flash/boot the Pi 5 |
| 5 | `phase-5-hpmini-frigate.md` | NUC session (drives the HP Mini over SSH) | No |
| 6 | `phase-6-ha-frigate-integration.md` | NUC session | No |
| 7 | `phase-7-nuc-cli-llm.md` | NUC session (local — this is the NUC's own build-out) | No |
| 8 | `phase-8-verify.md` | NUC session | Each host's reboot test drops the session for that host only; the NUC's own reboot test drops this session — restart it afterward. |
| 9 | `phase-9-automated-maintenance.md` | NUC session | No — this phase is a one-time setup of a **recurring weekly cron job**, not itself a migration step |

## The three machines

| Machine | Hardware | Role | OS |
|---|---|---|---|
| **NUC 14 Pro** | Core Ultra 5 125H, 32GB DDR5, 2TB NVMe, **Arc iGPU** | Claude Code CLI + IPEX-LLM (vision + chat models, Arc iGPU uncontended) | Debian 13, bare-metal (was Proxmox VE — wiped in Phase 2) |
| **Raspberry Pi 5** | 8GB RAM, NVMe (upgraded from microSD) | Home Assistant OS, bare-metal — Zigbee2MQTT, Mosquitto, Matter, ESPHome, voice (Whisper/Piper), Samba, Tailscale | HAOS (appliance OS, was always bare-metal) |
| **HP Pro Mini 400 G9** | i7-12700T (12C/20T), 32GB DDR4, **UHD 770 iGPU**, NVMe + SATA bay | Frigate (GPU-accelerated camera detection) | Debian 13 + Docker, bare-metal (never had a hypervisor) |

**Why this split, and why bare-metal:** the Zigbee coordinator is a network device (`tcp://192.168.68.98:6638`), not USB, so HA's physical location doesn't matter to the mesh — moving it to the Pi 5 frees the NUC entirely for the CLI + LLM workload. Frigate needs a capable iGPU for VAAPI decode + OpenVINO detection; the HP Mini's UHD 770 handles that well and keeps it off the NUC's Arc iGPU, which stays uncontended for the LLM (image-description responsiveness was the explicit priority — see Phase 7). Once each x86 machine ended up hosting exactly one workload, virtualization stopped earning its keep: no more workloads to multiplex, no LXC/VM passthrough layer to fight with, no hypervisor to patch. Snapshots/PBS were already deprioritized in favor of raw LLM performance, so the usual everyday case for keeping a hypervisor didn't apply here either — both x86 machines run **plain Debian 13**. A Google Coral Dual Edge TPU M.2 module exists but has no compatible slot on any of the three machines and is **out of scope** for this rebuild.

## Shared facts (referenced by every phase)

| Item | Value |
|---|---|
| **NUC** — old Proxmox host (pre-Phase 2 only) | `192.168.68.100`. Retired once Phase 2 wipes it. |
| Old HAOS VM (NUC) | VMID ≈100, IP `.101` — ceases to exist when Phase 2 wipes the disk, replaced by the Pi 5 |
| Old Frigate container (NUC) | IP `.102`, 4 cameras — ceases to exist when Phase 2 wipes the disk, replaced by the HP Mini |
| Coding VM (NUC, pre-Phase 2 only) | VMID 102, IP `.180`, Debian 13 — build vehicle, ceases to exist in Phase 2 (not individually destroyed — the whole host disk is wiped) |
| **NUC** — bare-metal (post-Phase 2) | `192.168.68.103`, Debian 13, no VMID — this is the whole machine now |
| **Raspberry Pi 5** — HAOS | `.101` (reused), bare-metal appliance, no VMID |
| **HP Mini** — bare-metal | `.102` (reused), Debian 13 + Docker, no VMID/CTID |
| State directory | `~/rebuild/` — `~/rebuild/salvage/` (backups, configs) and `~/rebuild/reports/phase-N.md` (handoff reports). Lives on the Coding VM through Phase 1, then must be salvaged off-NUC before Phase 2, then restored onto the new NUC install in Phase 2/7. |
| Salvage destination during Phase 1 | Off-NUC storage that survives the Phase 2 wipe — this repo's GitHub remote for anything non-sensitive, plus Lawrie's new laptop (or a USB drive) for `~/.claude/`, SSH keys, MCP tokens, and other sensitive/bulky data. Decided live in Phase 1 — see that file. |
| DHCP reservations | Made by Lawrie in the router app. Claude provides the MAC address and waits for confirmation. |
| Zigbee coordinator | SLZB-type, network-attached at `tcp://192.168.68.98:6638` — no USB/serial passthrough needed anywhere. |

## Ongoing maintenance (Phase 9)

Phases 0-8 are one-time build/migration steps. Phase 9 is different: it sets up a **recurring weekly cron job** on the NUC that keeps the fleet patched going forward — OS security patches fully automatic everywhere (`unattended-upgrades`), routine HACS/Frigate-patch bumps auto-applied by a scoped, unattended Claude Code run, and anything riskier (HA Core/Supervisor/OS updates, Frigate major versions, **Zigbee/ESPHome device firmware — never auto-applied**) just reported and pushed as a phone notification for Lawrie to approve manually. See that file for the full design reasoning.

## Handoff protocol

At the end of every phase, Claude writes `~/rebuild/reports/phase-N.md` recording: what was done, actual values discovered (MACs, storage layout, paths of backups), anything that deviated from the prompt, and open issues. **Every phase begins by reading the previous reports.** If a report is missing, the session must stop and ask Lawrie rather than assume.

## Safety rules baked into every prompt

- Destructive or irreversible steps (wiping a disk, `rm` of backups, a host reboot) always require Lawrie's explicit yes, immediately before execution, naming the exact target.
- Backups are verified to exist (file present, size > 0, readable) before the thing they back up is touched.
- If a verification check fails, the phase stops and reports; it does not improvise forward.
