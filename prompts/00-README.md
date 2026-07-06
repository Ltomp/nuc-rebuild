# Home lab rebuild — phase prompts for Claude Code (Sonnet)

These files implement `PLAN.md` one phase at a time, across **three physical machines**. Each file is a **complete, self-contained prompt** for a fresh Claude Code session running **Sonnet**. The session has none of the planning context, so every prompt repeats the facts it needs.

## How to use

1. Start a new Claude Code session (Sonnet) on the machine listed in the table below.
2. Paste the entire contents of the phase file as your first message (or, if this repo is checked out on that machine, tell Claude: *"Read prompts/phase-N-….md and follow it exactly."*).
3. Let the phase run to completion. Claude will stop and ask you before anything destructive — **read those confirmations carefully before saying yes.**
4. Do not start the next phase until the current phase's **Exit criteria** are all met and its report file exists.

| Phase | File | Run the session on | Interruption? |
|---|---|---|---|
| 0 | `phase-0-nuc-host-ssh.md` | Coding VM (.180) | No |
| 1 | `phase-1-nuc-audit-salvage.md` | Coding VM | No |
| 2 | `phase-2-nuc-shrink-teardown.md` | Coding VM | No (config only) |
| 3 | `phase-3-nuc-host-upgrade.md` | Coding VM | **Yes — NUC host reboot drops the session once.** Restart, paste the same prompt; it resumes. |
| 4 | `phase-4-pi5-haos.md` | Coding VM (drives the Pi 5 over SSH once flashed) | No, but needs Lawrie to physically flash/boot the Pi 5 |
| 5 | `phase-5-hpmini-bootstrap.md` | Coding VM (drives the HP Mini over SSH once Proxmox is installed) | No, but needs Lawrie to physically install Proxmox VE on the HP Mini first |
| 6 | `phase-6-hpmini-frigate.md` | Coding VM (drives the HP Mini over SSH) | No |
| 7 | `phase-7-ha-frigate-integration.md` | Coding VM | No |
| 8 | `phase-8-nuc-workstation-llm.md` | Coding VM (drives the NUC over SSH) | No |
| 9 | `phase-9-verify-decommission.md` | **workstation VM (.103, on the NUC)** | Brief RDP drop when workstation/LLM RAM is raised; each host's reboot test drops the session once. |

## The three machines

| Machine | Hardware | Role | Hypervisor |
|---|---|---|---|
| **NUC 14 Pro** | Core Ultra 5 125H, 32GB DDR5, 2TB NVMe, **Arc iGPU** | workstation VM (Claude Code/Chrome/RDP) + LLM LXC (Arc iGPU, uncontended) | Proxmox VE (existing, upgraded to 9.x in Phase 3) |
| **Raspberry Pi 5** | 8GB RAM, NVMe (upgraded from microSD) | Home Assistant OS, bare-metal — Zigbee2MQTT, Mosquitto, Matter, ESPHome, voice (Whisper/Piper), Samba, Tailscale | None — HAOS runs directly on hardware |
| **HP Pro Mini 400 G9** | i7-12700T (12C/20T), 32GB DDR4, **UHD 770 iGPU**, NVMe + SATA bay | Frigate LXC (GPU-accelerated camera detection) | Proxmox VE (new install, Phase 5) |

**Why split this way:** the Zigbee coordinator is a network device (`tcp://192.168.68.98:6638`), not USB, so HA's physical location doesn't matter to the mesh — moving it to the Pi 5 frees the NUC's RAM/CPU budget entirely for workstation + LLM. Frigate needs a capable iGPU for VAAPI decode + OpenVINO detection; the HP Mini's UHD 770 handles that well and keeps it off the NUC's Arc iGPU, which is reserved uncontended for the LLM (image-description responsiveness was the explicit priority — see Phase 8). A Google Coral Dual Edge TPU M.2 module exists but has no compatible slot on any of the three machines and is **out of scope** for this rebuild.

## Shared facts (referenced by every phase)

| Item | Value |
|---|---|
| **NUC** — Proxmox host | `192.168.68.100`. PVE 8.4.11 before Phase 3, PVE 9.x after. |
| Old HAOS VM (NUC) | VMID ≈100, IP `.101` — destroyed in Phase 2, replaced by the Pi 5 |
| Old Frigate container (NUC) | IP `.102`, 4 cameras — destroyed in Phase 2, replaced by the HP Mini LXC |
| Coding VM (NUC) | VMID 102, IP `.180`, Debian 13 — build vehicle, destroyed in Phase 9 |
| **Raspberry Pi 5** — HAOS | `.101` (reused), bare-metal, no VMID |
| **HP Mini** — Proxmox host | New static IP `.110` |
| Frigate LXC (HP Mini) | CTID **111**, `.102` (reused), 8GB / 6 vCPU / 100GB root + media mount (HP Mini has real headroom, no need to be stingy) |
| workstation VM (NUC) | VMID **112**, `.103`, **8GB** (→16GB in Phase 9) / 6 vCPU / 150GB |
| LLM LXC (NUC) | CTID **113**, `.104`, **8GB** (→12GB in Phase 9) / 4 vCPU / 100GB, Arc iGPU passthrough via `dev0`, served with IPEX-LLM |
| State directory | `~/rebuild/` on the Coding VM: `~/rebuild/salvage/` (backups, configs) and `~/rebuild/reports/phase-N.md` (handoff reports). Migrated to the workstation in Phase 8. |
| Host salvage dir | `/root/rebuild-salvage/` on the NUC |
| NUC RAM rule | Guest allocation must never exceed **~22GB** while the Coding VM still exists (6 + 8 workstation + 8 LLM). Final state after Coding VM is destroyed: 16 + 12 = 28GB, ~4GB headroom. |
| DHCP reservations | Made by Lawrie in the router app. Claude provides the MAC address and waits for confirmation. |
| Zigbee coordinator | SLZB-type, network-attached at `tcp://192.168.68.98:6638` — no USB/serial passthrough needed anywhere. |

## Handoff protocol

At the end of every phase, Claude writes `~/rebuild/reports/phase-N.md` recording: what was done, actual values discovered (VMIDs, MACs, storage names, paths of backups), anything that deviated from the prompt, and open issues. **Every phase begins by reading the previous reports.** If a report is missing, the session must stop and ask Lawrie rather than assume.

## Safety rules baked into every prompt

- Destructive commands (`qm destroy`, `pct destroy`, `rm` of backups, host `reboot`) always require Lawrie's explicit yes, immediately before execution, naming the exact target.
- Backups are verified to exist (file present, size > 0) before the thing they back up is touched.
- If a verification check fails, the phase stops and reports; it does not improvise forward.
