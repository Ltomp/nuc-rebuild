# Phase 8 — Final verification

You are Claude Code (Sonnet) running directly on the **NUC** (192.168.68.103, Debian 13, bare-metal). This final phase proves the whole three-machine rebuild end-to-end and tests that every machine's services survive a reboot. Unlike the old Proxmox-based design, there's no second guest to decommission or resize here — Phase 2 already retired the old Coding VM by wiping the host it ran on, and this NUC has run as a single bare-metal machine since. This phase is purely verification and closeout.

## Preconditions

```bash
hostname && hostname -I    # must show nuc / 192.168.68.103
ls ~/rebuild/reports/      # phase-0 .. phase-7 reports must all be present
ssh -o BatchMode=yes lawrie@192.168.68.102 'echo hpmini-ok'
curl -s -o /dev/null -w "%{http_code}" http://192.168.68.101:8123
```

If any of these fail — missing reports, no HP Mini access, HA unreachable — **stop and ask Lawrie.** Read `phase-5.md` for the Frigate setup, `phase-7.md` for the LLM stack, and the rest for general context.

## Steps

### 1. End-to-end system test (all three machines, no reboots yet)

- **Walk test**: Lawrie walks in front of a camera → confirm the full chain: Frigate (HP Mini, `.102`) detects the person → publishes to MQTT on the Pi 5 (`.101`) → HA sensor flips → `llmvision` calls Ollama on this NUC (`.103`) for a description → phone notification arrives **with both the snapshot and the generated description text**. This is the payoff of the whole redesign — confirm the description is generated promptly (responsiveness was the explicit priority) and reads as sensible.
- **HP Mini GPU**: `ssh lawrie@192.168.68.102 'timeout 10 intel_gpu_top -o -'` shows video-engine load while cameras stream.
- **NUC GPU**: `timeout 10 intel_gpu_top -o -` (run locally) shows render/compute load while a description request runs — confirm the LLM is actually hitting the Arc iGPU, not silently on CPU.
- **Resource headroom, NUC**: `free -h && uptime` — with nothing else running on this machine, there should be comfortable headroom and low load.

### 2. HP Mini reboot / autostart test (does not drop this session)

```bash
ssh lawrie@192.168.68.102 'sudo reboot'
```

Wait ~2 minutes, then:

```bash
ssh lawrie@192.168.68.102 'uptime && sudo docker ps'
```

Frigate should be running automatically (`restart: unless-stopped` in the compose file). Confirm the Frigate UI (`http://192.168.68.102:5000`) comes back and cameras resume streaming without manual intervention.

### 3. Raspberry Pi 5 reboot / autostart test (does not drop this session)

Ask Lawrie to power-cycle the Pi 5 (or, if SSH access to the HAOS host OS is available, `ssh <user>@192.168.68.101 'reboot'` via the SSH add-on). Wait a few minutes, then confirm `http://192.168.68.101:8123` responds and Zigbee2MQTT/Matter/ESPHome/voice add-ons all show running (Settings → Add-ons).

### 4. NUC reboot / autostart test — GATED, drops this session

Ask Lawrie:

> Testing that this NUC's services (Ollama/IPEX-LLM) survive a power cycle. The NUC will reboot and **this session drops**. SSH back in after a couple of minutes, restart a Claude Code session, paste this Phase 8 prompt again and continue from step 5. Reboot now? (yes/no)

On "yes": update `~/rebuild/reports/phase-8-progress.md` ("NUC rebooting for autostart test; resume at step 5"), then:

```bash
sudo reboot
```

### 5. Post-reboot checks (resumed session starts here if the progress file says so)

```bash
uptime
systemctl is-active ollama
curl -s http://localhost:11434/api/tags | head
```

Ollama should be running automatically via its systemd unit from Phase 7. Re-run a quick `llmvision` test to confirm it survived the reboot warm-cache-wise (a cold load here is expected and fine — this is a reboot test, not a latency test).

### 6. Backup retention + closeout

- List whatever Phase 1 vzdump/archive artifacts still exist on the old Proxmox storage, if reachable at all (they may not be, since that host no longer exists) — this is really just a note in the report that they're gone along with the old install, and that the off-NUC salvage archive (laptop/USB from Phase 1/2) is the thing actually worth keeping around for a while yet.
- Remind Lawrie of the deferred items from `PLAN.md`: the NAS/backup build (deliberately deferred in favor of LLM performance and simplicity), the Google Coral Dual Edge TPU module (no compatible slot on any of the three machines), and the OpenVINO **NPU** detector experiment on the NUC's Meteor Lake NPU.
- Write the final report `~/rebuild/reports/phase-8.md`: walk test result (including the description text quality/latency), autostart test results for all three machines, final resource table (machine/RAM/role/IP), open items. Delete `phase-8-progress.md`.

## Final resource table (expected end state)

| Machine | RAM | Role | IP |
|---|---|---|---|
| Raspberry Pi 5 | 8GB (fixed) | HAOS (bare-metal) | .101 |
| HP Pro Mini 400 G9 | 32GB (uncontended) | Frigate (Docker, bare-metal) | .102 |
| NUC 14 Pro | 32GB (uncontended) | Claude Code CLI + IPEX-LLM (bare-metal) | .103 |

No hypervisors, no VMs, no LXCs anywhere in the final state.

## Verification checklist

- [ ] Walk test passed end-to-end: Frigate → MQTT → HA → llmvision → LLM description → phone notification.
- [ ] HP Mini, Pi 5, and NUC all autostart their services after independent reboot tests.
- [ ] `~/rebuild/reports/phase-8.md` written.

## Rollback

Nothing destructive happens in this phase beyond the reboot tests themselves, which are expected to succeed cleanly given prior phases' verification gates.

## Exit criteria

Rebuild complete across three bare-metal machines: HAOS (Pi 5) + Frigate/GPU (HP Mini) + Claude Code/LLM (NUC), all autostarting, verified end-to-end including the LLM-powered camera description pipeline. Next phase: `phase-9-automated-maintenance.md`, ongoing recurring maintenance setup.
