# NUC 14 Pro rebuild — phase prompts for Claude Code (Sonnet)

These files implement `PLAN.md` one phase at a time. Each file is a **complete, self-contained prompt** for a fresh Claude Code session running **Sonnet**. The session has none of the planning context, so every prompt repeats the facts it needs.

## How to use

1. Start a new Claude Code session (Sonnet) on the machine listed in the table below.
2. Paste the entire contents of the phase file as your first message (or, if this repo is checked out on that machine, tell Claude: *"Read prompts/phase-N-….md and follow it exactly."*).
3. Let the phase run to completion. Claude will stop and ask you before anything destructive — **read those confirmations carefully before saying yes.**
4. Do not start the next phase until the current phase's **Exit criteria** are all met and its report file exists.

| Phase | File | Run the session on | Interruption? |
|---|---|---|---|
| 0 | `phase-0-host-ssh.md` | Coding VM (.180) | No |
| 1 | `phase-1-audit-salvage.md` | Coding VM | No |
| 2 | `phase-2-shrink-teardown.md` | Coding VM | No (config only) |
| 3 | `phase-3-host-upgrade.md` | Coding VM | **Yes — host reboot drops the session once.** Restart the session, paste the same prompt again; it is written to resume. |
| 4 | `phase-4-haos.md` | Coding VM | No |
| 5 | `phase-5-frigate.md` | Coding VM | No |
| 6 | `phase-6-ha-frigate.md` | Coding VM | No |
| 7 | `phase-7-workstation.md` | Coding VM | No |
| 8 | `phase-8-verify-decommission.md` | **workstation VM (.103)** | Brief RDP drop when workstation RAM is raised; host reboot test drops the session once. |

## Shared facts (referenced by every phase)

| Item | Value |
|---|---|
| Proxmox host | `192.168.68.100`, ASUS NUC 14 Pro, Core Ultra 5 125H (Meteor Lake), 32GB DDR5, 2TB NVMe, Intel Arc iGPU. PVE 8.4.11 before Phase 3, PVE 9.x after. |
| Old HAOS VM | VMID ≈100, IP `.101` — destroyed in Phase 2 |
| Old Frigate container | IP `.102`, 4 cameras — destroyed in Phase 2 |
| Coding VM | VMID 102, IP `.180`, Debian 13 — build vehicle, destroyed in Phase 8 |
| New HAOS VM | VMID **110**, `.101`, 4GB / 2 vCPU / 48GB |
| New Frigate LXC | CTID **111**, `.102`, 4GB / 4 vCPU / 32GB root + media mount |
| New workstation VM | VMID **112**, `.103`, **8GB** (→16GB in Phase 8) / 6 vCPU / 150GB |
| State directory | `~/rebuild/` on the Coding VM: `~/rebuild/salvage/` (backups, configs) and `~/rebuild/reports/phase-N.md` (handoff reports). Migrated to the workstation in Phase 7. |
| Host salvage dir | `/root/rebuild-salvage/` on the Proxmox host |
| RAM rule | Guest allocation must never exceed **~22GB** while the Coding VM still exists (6 + 4 + 4 + 8). The workstation goes to 16GB only after the Coding VM is destroyed. |
| DHCP reservations | Made by Lawrie in the router app (192.168.68.x network). Claude provides the MAC address and waits for confirmation. |

## Handoff protocol

At the end of every phase, Claude writes `~/rebuild/reports/phase-N.md` recording: what was done, actual values discovered (VMIDs, MACs, storage names, paths of backups), anything that deviated from the prompt, and open issues. **Every phase begins by reading the previous reports.** If a report is missing, the session must stop and ask Lawrie rather than assume.

## Safety rules baked into every prompt

- Destructive commands (`qm destroy`, `pct destroy`, `rm` of backups, host `reboot`) always require Lawrie's explicit yes, immediately before execution, naming the exact target.
- Backups are verified to exist (file present, size > 0) before the thing they back up is touched.
- If a verification check fails, the phase stops and reports; it does not improvise forward.
