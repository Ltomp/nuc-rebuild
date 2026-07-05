# NUC 14 Pro clean rebuild: Proxmox + HAOS + Frigate (iGPU) + Claude workstation

> **Execution:** each phase has a self-contained implementation prompt in [`prompts/`](prompts/00-README.md), written for Claude Code sessions running Sonnet. Start with `prompts/00-README.md`.

## Context

The ASUS NUC 14 Pro (Core Ultra 5 125H, 32GB DDR5, 2TB NVMe, Arc iGPU) runs Proxmox VE at **192.168.68.100**. Current guests were built ad-hoc: HAOS VM (VMID ~100, IP .101), Frigate container (IP .102, 4 cameras), and the **Coding VM (VMID 102, IP .180)** — the Debian 13 VM this Claude Code session runs in. Everything works but not correctly/optimally.

Decision: **teardown and clean rebuild.** Old HAOS VM and Frigate container get deleted (configs salvaged first). A fresh HAOS VM, a fresh Frigate LXC (accelerated on the Arc iGPU), and a new permanent **workstation VM** (Claude Code + XFCE + Chrome for Claude's Chrome integration) get built. The Coding VM is only the build vehicle — it gets shrunk for the rebuild and **deleted at the end** once its data is migrated to the new workstation VM. Backups/NAS are a separate future project; vzdump archives on the host's own disk are the safety net during this work.

## Target end state (32GB total, 24GB allocated, ~8GB host headroom)

| Guest | Type | IP | Resources | Role |
|---|---|---|---|---|
| HAOS | VM | .101 | 4GB / 2 vCPU / 48GB disk | Home Assistant OS + Mosquitto add-on |
| Frigate | LXC (Debian 13 + Docker) | .102 | 4GB / 4 vCPU / 32GB root + media mount | 4 cameras, VAAPI decode + OpenVINO GPU detection |
| workstation | VM | **.103** | 16GB (**8GB until the Coding VM is decommissioned**) / 6 vCPU / 150GB disk | Debian 13 + XFCE + xrdp + Chrome + Claude Desktop + Claude Code (migrated from Coding VM) |

All guests: CPU type `host`, virtio disk/net, qemu-guest-agent (VMs), `onboot` enabled with start order HAOS → Frigate → workstation. Static DHCP reservations for all three plus the 4 cameras.

**RAM rule during the build:** while the Coding VM (6GB) still exists, guest allocation must stay ≤ ~22GB (6+4+4+8) — hence the workstation is created at 8GB and only raised to 16GB in Phase 8. The workstation gets **.103** (not .180) because it runs alongside the Coding VM until then; the .180 reservation is retired at decommission.

## Phase 0 — Host SSH access (one manual step for Lawrie)

1. Check/generate SSH keypair on Coding VM (`~/.ssh/`).
2. Lawrie pastes one provided command into Proxmox web UI → node → Shell to authorize the key in `/root/.ssh/authorized_keys`.
3. Verify `ssh root@192.168.68.100` from here. Everything after this is driven from this session.

## Phase 1 — Audit + salvage (before touching anything)

- Host audit: `pveversion`, `qm list` / `pct list` (confirm real VMIDs), `pvesm status` + storage layout of the 2TB NVMe, per-guest configs, `/dev/dri` on host, repo/update state.
- **Salvage HA**: trigger a full HA backup (web UI or `ha backups new` via console), copy the `.tar` to host storage and to this VM.
- **Salvage Frigate**: copy `config.yml` (camera URLs/credentials, zones/masks) to host storage and this VM. Note current media usage.
- `vzdump` one-shot backup of the old HAOS VM, the Frigate container, **and the Coding VM** (it carries all the Claude data through the risky Phase 3 host upgrade) to host storage — after checking free space. Insurance that survives their deletion.

## Phase 2 — Shrink Coding VM + teardown

- **Configure** the Coding VM down to 6GB RAM / 4 vCPU (enough to drive the rebuild; stays headless — Chrome lives in the new workstation VM). The change applies at the next power-off/on — i.e. the Phase 3 host reboot — so no reboot happens here and there is **one session interruption total** across Phases 2–3. Disk allocation stays as-is (shrinking virtual disks is risky and pointless for a VM being deleted later).
- Stop and **destroy old HAOS VM and old Frigate container** (vzdump archives from Phase 1 remain on host). Confirm with Lawrie immediately before each destroy. Cameras go dark from here until Phase 5.

## Phase 3 — Proxmox host upgrade + optimisation

Host is on **PVE 8.4.11** (Debian 12 base, 6.8 kernel) — a major version behind. With the old guests gone and only the shrunk Coding VM left, this is the ideal window for the in-place upgrade to **PVE 9.x** (Debian 13 base, 6.14+ kernel — notably better Meteor Lake / Arc iGPU support):

- Run the official `pve8to9 --full` checklist and resolve any blockers.
- Switch repos to PVE 9 no-subscription, `apt dist-upgrade` per the official upgrade path, then reboot the host. **The reboot drops this session** (the Coding VM restarts, applying the Phase 2 shrink in the same interruption); Lawrie restarts the session afterwards.
- Then optimise: `intel-microcode`, `intel-gpu-tools`, full update.
- Verify iGPU on host: `/dev/dri/renderD128`, i915/xe driver loaded, `vainfo` clean.
- Confirm governor (powersave + intel_pstate is correct for an always-on Meteor Lake box).

## Phase 4 — New HAOS VM (built first — it hosts the MQTT broker)

- Import latest HAOS qcow2 as new VM (4GB / 2 vCPU, virtio, qemu agent, static reservation on .101).
- Restore the Phase 1 HA backup → automations/integrations/devices return.
- Install/verify **Mosquitto broker add-on**; create `frigate` MQTT user.

## Phase 5 — New Frigate LXC on the Arc iGPU

- Debian 13 LXC (unprivileged) with `/dev/dri/renderD128` device passthrough (`dev0: /dev/dri/renderD128,gid=<render group gid inside the CT>`) + nesting for Docker; static reservation (reuse .102).
- Docker + Frigate (compose), `shm_size` sized for 4 cameras, media on a dedicated mount from the NVMe with retention sized to available space.
- Rebuild `config.yml` from salvaged camera credentials, done properly:
  - `go2rtc` restreams — one connection per camera; detect on ~720p substreams, record main streams.
  - `ffmpeg: hwaccel_args: preset-vaapi` (Arc decodes all 4 streams trivially).
  - Detector `openvino` on `GPU` — expect ~15–25 ms inference vs 100 ms+ on CPU.
  - MQTT → HAOS Mosquitto; sensible record/snapshot retention; keep salvaged zones/masks.
- Verify: `intel_gpu_top` shows video+render load; Frigate UI inference speed; low CPU.

## Phase 6 — HA ↔ Frigate integration

- HACS + Frigate integration in HA → cameras, events, sensors.
- Camera dashboard + one worked example automation (person detected → phone notification with snapshot).

## Phase 7 — New workstation VM (permanent Claude home)

- Debian 13 VM: **8GB** (→16GB in Phase 8) / 6 vCPU / 150GB on **.103**, XFCE (minimal, `--no-install-recommends`), xrdp, Google Chrome (Google apt repo).
- **Claude Desktop for Linux (beta)** from Anthropic's apt repo (Debian 12+ x86_64 qualifies) — Chat/Cowork/Code tabs, parallel sessions, diff review, integrated terminal/editor; updates via `apt upgrade`. Sign-in with the claude.ai account (no API keys). Linux-beta limits: no Computer Use, no in-app dictation. Node.js + Claude Code CLI installed alongside for headless/SSH use.
- Migrate from Coding VM: `~/claude/` projects, `~/.claude/` (memory, settings, plans), `~/awa`, npm global setup, `.gitconfig`/credentials, MCP server configs/tokens, SSH keys (incl. the host-authorized key), plus the `~/rebuild/` salvage + reports.
- Lawrie RDPs in, signs into Claude Desktop and Chrome, installs the **Claude in Chrome** extension.

## Phase 8 — Verification + decommission

- End-to-end: walk-test triggers Frigate detection on GPU → HA notification with snapshot; all guests auto-start after a host reboot test; RAM/CPU headroom sane under load.
- New session from the workstation VM confirms Claude Code + Chrome integration works there.
- **Only then**: final check that nothing on the Coding VM is unmigrated → destroy Coding VM (Lawrie confirms; done from workstation or web UI, not from inside it).
- After the destroy: **raise the workstation to 16GB** (quick power-cycle) and retire the `.180` DHCP reservation → final state 24GB allocated / ~8GB headroom.
- vzdump archives (old HAOS, old Frigate, Coding VM) kept until Lawrie is happy, then removed to reclaim space.

## Deferred

- NAS build → scheduled Proxmox backups + Frigate recordings off the NVMe.
- NPU detection experiment (OpenVINO `NPU` device) after the GPU baseline is proven.
