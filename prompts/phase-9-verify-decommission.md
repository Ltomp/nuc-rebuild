# Phase 9 — Final verification + decommission of the Coding VM

You are Claude Code (Sonnet) running on the **new workstation VM** (192.168.68.103, VMID 112, on the NUC) — NOT the old Coding VM. Verify this first (below). This final phase proves the whole three-machine rebuild end-to-end, tests autostart on all three hosts, raises the workstation and LLM LXC to their final sizes, retires the old Coding VM, and closes out.

## Preconditions — confirm where you are

```bash
hostname && hostname -I    # must show workstation / 192.168.68.103
ls ~/rebuild/reports/      # phase-0 .. phase-8 reports must all be present (migrated in Phase 8)
ssh -o BatchMode=yes root@192.168.68.100 'echo nuc-ok'
ssh -o BatchMode=yes root@192.168.68.110 'echo hpmini-ok'
```

If any of these fail — wrong machine, missing reports, no host access — **stop and ask Lawrie.** Read `phase-8.md` for what was migrated, `phase-2.md` for the Coding VM's pre-shrink values, `phase-6.md` for the Frigate CTID, and `phase-8.md` for the workstation/LLM VMID/CTID.

## Steps

### 1. End-to-end system test (all three machines, no reboots yet)

- **Walk test**: Lawrie walks in front of a camera → confirm the full chain: Frigate (HP Mini, `.102`) detects the person → publishes to MQTT on the Pi 5 (`.101`) → HA sensor flips → `llmvision` calls the LLM LXC (NUC, `.104`) for a description → phone notification arrives **with both the snapshot and the generated description text**. This is the payoff of the whole redesign — confirm the description is generated promptly (this was the explicit "highly responsive" priority) and reads as sensible.
- **HP Mini GPU**: `ssh root@192.168.68.110 'timeout 10 intel_gpu_top -o -'` shows video-engine load while cameras stream.
- **NUC GPU**: `ssh root@192.168.68.100 'timeout 10 intel_gpu_top -o -'` shows render/compute load while a description request runs — confirm the LLM is actually hitting the Arc iGPU, not silently on CPU.
- **Resource headroom, NUC**: `ssh root@192.168.68.100 'free -h && uptime'` — with 22GB allocated (Coding VM 6 + workstation 8 + LLM 8) there should be headroom and low load.

### 2. HP Mini reboot / autostart test (does not drop this session)

```bash
ssh root@192.168.68.110 'reboot'
```

Wait ~2 minutes, then:

```bash
ssh root@192.168.68.110 'uptime && pct list'
```

Frigate LXC should be running automatically. Confirm the Frigate UI (`http://192.168.68.102:5000`) comes back and cameras resume streaming without manual intervention.

### 3. Raspberry Pi 5 reboot / autostart test (does not drop this session)

Ask Lawrie to power-cycle the Pi 5 (or, if SSH access to the HAOS host OS is available, `ssh <user>@192.168.68.101 'reboot'` via the SSH add-on). Wait a few minutes, then confirm `http://192.168.68.101:8123` responds and Zigbee2MQTT/Matter/ESPHome/voice add-ons all show running (Settings → Add-ons).

### 4. NUC reboot / autostart test — GATED, drops this session

Ask Lawrie:

> Testing that the workstation and LLM survive a power cycle. The NUC will reboot; the Coding VM, workstation, and LLM LXC all restart automatically. **This session drops** (you're on the workstation, which is rebooting) — RDP back in after a few minutes, restart a session, paste this Phase 9 prompt again and continue from step 5. Reboot the NUC now? (yes/no)

On "yes": update `~/rebuild/reports/phase-9-progress.md` ("NUC rebooting for autostart test; resume at step 5"), then `ssh root@192.168.68.100 'reboot'`.

### 5. Post-reboot checks (resumed session starts here if the progress file says so)

```bash
ssh root@192.168.68.100 'uptime && qm list && pct list'
```

Coding VM, workstation, and LLM LXC all running, started automatically. Re-verify: workstation reachable at `.103`, LLM endpoint responds at `http://192.168.68.104:11434`, and re-run a quick `llmvision` test to confirm it survived the reboot warm-cache-wise (a cold load here is expected and fine — this is a reboot test, not a latency test).

### 6. Final unmigrated-data check on the Coding VM — GATED destroy

The Coding VM (VMID 102, `.180`) should still be running. Sweep it for anything Phase 8 missed:

```bash
ssh <user>@192.168.68.180 'ls -la ~; crontab -l 2>/dev/null; npm ls -g --depth=0 2>/dev/null'
```

Compare against the Phase 8 migration list. If anything of value is unmigrated, rsync it over now and note it. Then present the gate to Lawrie:

> Final check complete — nothing unmigrated remains on the Coding VM (VMID 102, .180): `<summary of the sweep>`. Its Phase 1 vzdump backup `<file, size, date>` still exists on the NUC as a last resort. About to permanently destroy VMID 102. Confirm? (yes/no)

Only after "yes":

```bash
ssh root@192.168.68.100 'qm stop 102 && qm destroy 102 --purge --destroy-unreferenced-disks 1'
```

Verify gone from `qm list`.

### 7. Raise the workstation and LLM LXC to their final sizes — brief self-interruption

With the Coding VM gone, NUC allocation becomes workstation(16) + LLM(12) = 28GB (~4GB host headroom) — the designed end state.

```bash
ssh root@192.168.68.100 'qm set 112 --memory 16384'
ssh root@192.168.68.100 'pct set 113 --memory 12288'
```

The workstation's change needs a power-off/on (this session's own VM — warn Lawrie: "this session and RDP drop for ~1 minute"):

```bash
ssh root@192.168.68.100 'nohup bash -c "qm shutdown 112 --timeout 60 && sleep 5 && qm start 112" >/dev/null 2>&1 &'
```

The LLM LXC's change can apply with a simple restart (does not affect this session):

```bash
ssh root@192.168.68.100 'pct reboot 113'
```

After reconnecting to the workstation (new session, this prompt, resume here): `free -h` shows ≈16GB on the workstation; `ssh root@192.168.68.100 "pct exec 113 -- free -h"` shows ≈12GB on the LLM LXC. Also ask Lawrie to retire the `.180` DHCP reservation in the router app — it's now orphaned.

### 8. Backup retention + closeout

- List the Phase 1 archives: `ssh root@192.168.68.100 'ls -lh <dump-dir>'`. Recommend to Lawrie: **keep all three vzdumps (old HAOS, old Frigate, Coding VM) until they've been happy with the system for a couple of weeks**, then delete to reclaim space (that deletion is theirs to trigger — do not delete now).
- Remind Lawrie of the deferred items from `PLAN.md`: the NAS/Proxmox Backup Server build on the HP Mini's free SATA bay (deliberately deferred in favor of LLM performance), the Google Coral Dual Edge TPU module (no compatible slot on any of the three machines), and the OpenVINO **NPU** detector experiment on the NUC's Meteor Lake NPU.
- Write the final report `~/rebuild/reports/phase-9.md`: walk test result (including the description text quality/latency), autostart test results for all three hosts, final resource table (machine/guest/RAM/vCPU/IP), Coding VM destroy confirmation, backup retention decision, open items. Delete `phase-9-progress.md`.

## Final resource table (expected end state)

| Machine | Guest | RAM | vCPU | IP |
|---|---|---|---|---|
| Raspberry Pi 5 | HAOS (bare-metal) | 8GB (fixed) | 4 cores | .101 |
| HP Mini | Frigate LXC | 8GB | 6 | .102 |
| NUC | workstation VM | 16GB | 6 | .103 |
| NUC | LLM LXC | 12GB | 4 | .104 |

## Verification checklist

- [ ] Walk test passed end-to-end: Frigate → MQTT → HA → llmvision → LLM description → phone notification.
- [ ] HP Mini, Pi 5, and NUC all autostart their services/guests after independent reboot tests.
- [ ] Coding VM destroyed (after Lawrie's explicit yes); `.180` reservation retired.
- [ ] Workstation at 16GB/6vCPU; LLM LXC at 12GB/4vCPU; NUC shows ~4GB free headroom.
- [ ] vzdump archives retained; Lawrie owns their eventual deletion.
- [ ] `~/rebuild/reports/phase-9.md` written.

## Rollback

Until step 6's destroy, everything is reversible. After it, the Coding VM exists only as its Phase 1 vzdump (`qmrestore` brings it back if something surfaces later) — which is why that archive is kept for weeks, not deleted at closeout.

## Exit criteria

Rebuild complete across three machines: HAOS (Pi 5) + Frigate/GPU (HP Mini) + workstation/LLM (NUC), all autostarting, verified end-to-end including the LLM-powered camera description pipeline, old guests gone, backups retained. 🎉
