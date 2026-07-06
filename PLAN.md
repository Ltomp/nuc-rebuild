# Home lab clean rebuild: NUC (Proxmox + workstation + LLM) + Pi 5 (HAOS) + HP Mini (Proxmox + Frigate)

> **Execution:** each phase has a self-contained implementation prompt in [`prompts/`](prompts/00-README.md), written for Claude Code sessions running Sonnet. Start with `prompts/00-README.md`.

## Context

The ASUS NUC 14 Pro (Core Ultra 5 125H, 32GB DDR5, 2TB NVMe, Arc iGPU) runs Proxmox VE at **192.168.68.100**. Current guests were built ad-hoc: HAOS VM (VMID ~100, IP .101), Frigate container (IP .102, 4 cameras), and the **Coding VM (VMID 102, IP .180)** — the Debian 13 VM this Claude Code session runs in.

A live audit of the current HAOS instance (SSH into the add-on, real device/entity/integration counts, `docker stats`) found it's a much bigger install than a fresh-rebuild plan should assume: 144 devices, 1,752 entities, 58 integrations, a 51-device Zigbee mesh, local voice assist (Whisper/Piper), Matter, ESPHome, HACS, and a 419MB recorder database — currently running on a VM allocated 15GB RAM / **1 vCPU**, with 2GB in swap. The original single-NUC rebuild plan (4GB/2vCPU for a new HAOS VM) was significantly undersized for this.

Two more machines became available during planning — a **Raspberry Pi 5 (8GB)** and an **HP Pro Mini 400 G9 (i7-12700T, 32GB DDR4, UHD 770 iGPU, NVMe + SATA bay)** — plus a spare **Google Coral Dual Edge TPU M.2 module** (no compatible slot on any of the three machines; out of scope). Given Lawrie also wants a **local LLM** for camera image description (via the currently-dormant `llmvision` integration), general chat, and coding experimentation — with image-description **responsiveness** as the explicit priority — the rebuild was redesigned across all three machines rather than cramming everything onto one host:

- **The Zigbee coordinator is network-attached** (`tcp://192.168.68.98:6638`, not USB), so Home Assistant's physical location doesn't affect the mesh at all — moving it to the Pi 5 was free.
- **Frigate's GPU needs and the LLM's GPU needs compete for the same Arc iGPU** if both stay on the NUC. Since LLM responsiveness was prioritized over anything else, Frigate moved to the HP Mini (UHD 770 — plenty capable for VAAPI decode + OpenVINO detection), leaving the NUC's Arc iGPU **uncontended** for the LLM.
- **LXC, not a VM, for GPU work**: an LXC container shares the host kernel directly (device passthrough via `dev0`, no hypervisor translation layer), so it gets essentially bare-metal GPU/CPU performance under Proxmox — there's no performance reason to abandon Proxmox on either host, as long as GPU-bound workloads (Frigate, the LLM) are LXCs rather than VMs.
- **Snapshots/PBS were explicitly deprioritized** by Lawrie in favor of raw LLM performance — the "Deferred" NAS/backup item stays deferred; vzdump archives on the NUC's own disk remain the interim safety net.

Decision: **teardown and clean rebuild across three machines.** Old HAOS VM and Frigate container on the NUC get deleted (configs salvaged first). Home Assistant moves to the Pi 5 (bare-metal). Frigate moves to a new Proxmox install on the HP Mini (LXC, UHD 770). The NUC keeps Proxmox and hosts a permanent **workstation VM** (Claude Code + XFCE + Chrome + Claude Desktop) and a new **LLM LXC** (Arc iGPU, IPEX-LLM). The Coding VM is only the build vehicle — it gets shrunk for the rebuild and **deleted at the end** once its data is migrated to the new workstation VM.

## Target end state

| Machine | Guest | Type | IP | Resources | Role |
|---|---|---|---|---|---|
| Raspberry Pi 5 | — | bare-metal | .101 | 8GB RAM (fixed), NVMe | Home Assistant OS: Zigbee2MQTT, Mosquitto, Matter, ESPHome, voice (Whisper/Piper), Samba, Tailscale |
| HP Pro Mini 400 G9 | Frigate | LXC | .102 | 8GB / 6 vCPU / 100GB root + media mount | 4 cameras, VAAPI decode + OpenVINO GPU detection on UHD 770 |
| NUC 14 Pro | workstation | VM | .103 | 16GB (**8GB until Coding VM decommissioned**) / 6 vCPU / 150GB | Debian 13 + XFCE + xrdp + Chrome + Claude Desktop + Claude Code (migrated from Coding VM) |
| NUC 14 Pro | LLM | LXC | .104 | 12GB (**8GB until Coding VM decommissioned**) / 4 vCPU / 100GB | Arc iGPU (uncontended), IPEX-LLM: small vision model kept warm for `llmvision` camera captioning + a 7-14B model for chat/coding |

All Proxmox guests: CPU type `host`, virtio disk/net, qemu-guest-agent (VMs), `onboot` enabled. Static DHCP reservations for all guests plus the 4 cameras and both Proxmox hosts themselves (NUC `.100`, HP Mini `.110`).

**NUC RAM rule during the build:** while the Coding VM (6GB) still exists, guest allocation must stay ≤ ~22GB (6 + 8 workstation + 8 LLM). Final state after Coding VM is destroyed: 16 + 12 = 28GB, ~4GB headroom on the NUC's 32GB.

**HP Mini RAM:** 8GB for Frigate leaves ~24GB of its 32GB genuinely spare — no NAS/PBS role assigned to it in this rebuild (deferred, per Lawrie), but there's ample headroom if that changes later.

## Phase 0 — NUC host SSH access (one manual step for Lawrie)

1. Check/generate SSH keypair on Coding VM (`~/.ssh/`).
2. Lawrie pastes one provided command into Proxmox web UI → NUC node → Shell to authorize the key in `/root/.ssh/authorized_keys`.
3. Verify `ssh root@192.168.68.100` from here. Everything after this is driven from this session.

## Phase 1 — Audit + salvage (before touching anything)

- Host audit: `pveversion`, `qm list` / `pct list` (confirm real VMIDs), `pvesm status` + storage layout, per-guest configs, `/dev/dri` on host.
- **Salvage HA**: trigger a **full** backup (not partial — a full backup includes add-on data: Zigbee2MQTT's paired device database, Matter fabric certificates, ESPHome secrets, voice assist config, everything). Copy the `.tar` to host storage and to this VM. This backup will be substantial (recorder DB alone is ~400MB+) — budget transfer time accordingly.
- **Salvage Frigate**: copy `config.yml` (camera URLs/credentials, zones/masks) to host storage and this VM. Note current media usage.
- `vzdump` one-shot backup of the old HAOS VM, the Frigate container, **and the Coding VM** (it carries all the Claude data through the risky Phase 3 host upgrade) to host storage — after checking free space.

## Phase 2 — Shrink Coding VM + teardown

- **Configure** the Coding VM down to 6GB RAM / 4 vCPU (enough to drive the rebuild; stays headless — Chrome lives in the new workstation VM). The change applies at the next power-off/on — i.e. the Phase 3 host reboot — so no reboot happens here and there is **one session interruption total** across Phases 2–3.
- Stop and **destroy old HAOS VM and old Frigate container** on the NUC (vzdump archives from Phase 1 remain on host). Confirm with Lawrie immediately before each destroy. Cameras and HA go dark from here until Phases 4/6.

## Phase 3 — NUC Proxmox host upgrade + optimisation

Host is on **PVE 8.4.11** (Debian 12 base, 6.8 kernel). With the old guests gone and only the shrunk Coding VM left, this is the ideal window for the in-place upgrade to **PVE 9.x** (Debian 13 base, 6.14+ kernel — better Meteor Lake / Arc iGPU support, and notably the modern **Xe driver** the LLM LXC benefits from):

- Run `pve8to9 --full`, resolve blockers, switch repos, `apt dist-upgrade`, reboot. **The reboot drops this session** (the Coding VM restarts, applying the Phase 2 shrink in the same interruption); Lawrie restarts the session afterwards.
- Optimise: `intel-microcode`, `intel-gpu-tools`, full update.
- Verify iGPU: `/dev/dri/renderD128`, **Xe driver** loaded (not legacy i915), `vainfo` clean.
- Confirm governor (powersave + intel_pstate is correct for an always-on box).

## Phase 4 — Raspberry Pi 5: Home Assistant OS (bare-metal)

- Lawrie physically upgrades the Pi 5 from microSD to NVMe (recommended given the recorder's write volume), flashes the official HAOS Raspberry Pi 5 image, boots it.
- Static DHCP reservation on **.101** (reused).
- Restore the Phase 1 **full** backup → Zigbee2MQTT (reconnects to the network coordinator automatically, no re-pairing), Matter, ESPHome, voice assist, Samba, automations, HACS all return. Tailscale may need manual re-auth (node identity ties to the physical install).
- Given the Pi 5's **fixed 8GB ceiling** (no borrowing from a bigger pool like a VM could), add a `recorder:` block with sensible `exclude`/`purge_keep_days` tuning — the current install's unbounded recorder growth (419MB and climbing) is exactly the kind of thing that will hurt on fixed memory.
- Consider dropping the VS Code/code-server add-on now that a full workstation VM exists for real dev work.

## Phase 5 — HP Pro Mini 400 G9: Proxmox bootstrap

- Lawrie installs Proxmox VE fresh on the HP Mini (USB installer), gives it a static IP (**.110**).
- Same SSH-key-authorization pattern as Phase 0, establishing root SSH from the Coding VM to the HP Mini.

## Phase 6 — HP Mini: Frigate LXC on the UHD 770

- Debian 13 LXC (unprivileged) with `/dev/dri/renderD128` passthrough (`dev0: /dev/dri/renderD128,gid=<render gid in CT>`) + nesting for Docker; static reservation **.102** (reused).
- Docker + Frigate (compose); HP Mini has real headroom so sizing can be generous rather than tight.
- Rebuild `config.yml` from the salvaged camera credentials properly: `go2rtc` one connection per camera; `ffmpeg: hwaccel_args: preset-vaapi`; detector `openvino` on `GPU` (UHD 770); MQTT → the Pi 5's Mosquitto at **.101**; keep salvaged zones/masks.
- Verify: `intel_gpu_top` shows video+render load; Frigate UI inference speed; low CPU.

## Phase 7 — HA ↔ Frigate integration

- HACS + Frigate integration in HA (now on the Pi 5) pointed at the Frigate LXC (now on the HP Mini) → cameras, events, sensors.
- Camera dashboard + one worked example automation (person detected → phone notification with snapshot).

## Phase 8 — NUC: workstation VM + LLM LXC

- **Workstation VM**: Debian 13, **8GB** (→16GB in Phase 9) / 6 vCPU / 150GB on **.103**, XFCE (minimal), xrdp, Google Chrome, **Claude Desktop for Linux (beta)** from Anthropic's apt repo alongside the Claude Code CLI. Migrate `~/claude/`, `~/.claude/`, `~/awa`, npm globals, git config/credentials, MCP configs, SSH keys, and `~/rebuild/` from the Coding VM.
- **LLM LXC**: unprivileged, `dev0` passthrough to the Arc iGPU (`renderD128`), **8GB** (→12GB in Phase 9) / 4 vCPU / 100GB on **.104**. Serve via **IPEX-LLM** (Intel's Xe-optimized inference stack — meaningfully faster than a generic Vulkan/CPU backend on Arc). Two models: a small vision-capable model (e.g. Moondream2 or Qwen2-VL-2B) **kept warm** (no cold-load penalty) for `llmvision` camera captioning, and a separate 7–14B model for general chat/coding experimentation. LXC (not a VM) is deliberate — near-bare-metal GPU/CPU performance since there's no hypervisor translation layer.

## Phase 9 — Verification + decommission

- End-to-end walk test spanning all three machines: Frigate (HP Mini) detects a person → MQTT to HA (Pi 5) → HA calls `llmvision` → hits the LLM LXC (NUC) for a description → phone notification with snapshot + description.
- All three machines' guests/services auto-start after a reboot test on each host.
- **Only then**: final check that nothing on the Coding VM is unmigrated → destroy Coding VM (Lawrie confirms; done from workstation or web UI, not from inside it).
- After the destroy: **raise the workstation to 16GB and the LLM LXC to 12GB** (quick power-cycles) and retire the `.180` DHCP reservation → final NUC state 28GB allocated / ~4GB headroom.
- vzdump archives (old HAOS, old Frigate, Coding VM) kept until Lawrie is happy, then removed to reclaim space.

## Deferred

- NAS build / Proxmox Backup Server on the HP Mini's free SATA bay — explicitly deferred by Lawrie in favor of LLM performance; vzdump-on-local-disk remains the interim safety net.
- Google Coral Dual Edge TPU M.2 accelerator — no compatible slot on any of the three machines (NUC/HP Mini are slotless SFF designs; Pi 5's PCIe HAT breakout is M-key, not E-key). Revisit if hardware with a genuine E-key slot becomes available.
- NPU detection experiment (OpenVINO `NPU` device on the NUC's Meteor Lake NPU) after the GPU baseline is proven.
