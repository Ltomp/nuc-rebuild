# Phase 8 — Final verification + decommission of the Coding VM

You are Claude Code (Sonnet) running on the **new workstation VM** (192.168.68.103, VMID 112) — NOT the old Coding VM. Verify this first (below). This final phase proves the whole rebuild end-to-end, raises the workstation to its final 16GB, retires the old Coding VM (192.168.68.180, VMID 102), and closes out.

## Preconditions — confirm where you are

```bash
hostname && hostname -I    # must show workstation / 192.168.68.103
ls ~/rebuild/reports/      # phase-0 .. phase-7 reports must all be present (migrated in Phase 7)
ssh -o BatchMode=yes root@192.168.68.100 'echo host-ok'
```

If any of these fail — wrong machine, missing reports, no host access — **stop and ask Lawrie.** Read `phase-7.md` for what was migrated and `phase-2.md`/`phase-5.md` for guest IDs.

## Steps

### 1. End-to-end system test

- **Walk test**: Lawrie walks in front of a camera → confirm the chain: Frigate event (UI at `http://192.168.68.102:5000`, inference still ~15–25 ms on GPU) → HA sensor flips → phone notification with snapshot arrives.
- **Host GPU**: `ssh root@192.168.68.100 'timeout 10 intel_gpu_top -o -'` shows video-engine load while cameras stream.
- **Resource headroom**: `ssh root@192.168.68.100 'free -h && uptime'` — with 22GB allocated (6+4+4+8) the host should have several GB free and low load.

### 2. Host reboot / autostart test — GATED, drops this session

Ask Lawrie:

> Testing that everything survives a power cycle. The host will reboot; HAOS, Frigate and this workstation all restart automatically (order 1→2→3). **This session drops** — RDP back in after ~5 minutes, restart a session, paste this Phase 8 prompt again and continue from step 3. Reboot the host now? (yes/no)

On "yes": update `~/rebuild/reports/phase-8-progress.md` ("rebooting for autostart test; resume at step 3"), then `ssh root@192.168.68.100 'reboot'`.

### 3. Post-reboot checks (resumed session starts here if the progress file says so)

```bash
ssh root@192.168.68.100 'uptime && qm list && pct list'
```

All three guests running, started automatically in order. Re-verify: HA at `http://192.168.68.101:8123` responds, all 4 cameras live in Frigate, MQTT connected (Frigate UI shows no MQTT errors), and this workstation came back on `.103`.

### 4. Final unmigrated-data check on the Coding VM — GATED destroy

The Coding VM (VMID 102, `.180`) should still be running. Sweep it for anything Phase 7 missed:

```bash
ssh <user>@192.168.68.180 'ls -la ~; crontab -l 2>/dev/null; npm ls -g --depth=0 2>/dev/null'
```

Compare against the Phase 7 migration list. If anything of value is unmigrated, rsync it over now and note it. Then present the gate to Lawrie:

> Final check complete — nothing unmigrated remains on the Coding VM (VMID 102, .180): `<summary of the sweep>`. Its Phase 1 vzdump backup `<file, size, date>` still exists on the host as a last resort. About to permanently destroy VMID 102. Confirm? (yes/no)

Only after "yes":

```bash
ssh root@192.168.68.100 'qm stop 102 && qm destroy 102 --purge --destroy-unreferenced-disks 1'
```

Verify gone from `qm list`.

### 5. Raise the workstation to its final 16GB — brief self-interruption

With the Coding VM gone, allocation is 4+4+16 = 24GB (~8GB host headroom) — the designed end state.

```bash
ssh root@192.168.68.100 'qm set 112 --memory 16384'
```

The change needs a power-off/on of this VM. Warn Lawrie ("this session and RDP drop for ~1 minute"), then:

```bash
ssh root@192.168.68.100 'nohup bash -c "qm shutdown 112 --timeout 60 && sleep 5 && qm start 112" >/dev/null 2>&1 &'
```

After reconnecting (new session, this prompt, resume here): `free -h` shows ≈16GB. Also ask Lawrie to retire the `.180` DHCP reservation in the router app — it's now orphaned.

### 6. Backup retention + closeout

- List the Phase 1 archives: `ssh root@192.168.68.100 'ls -lh <dump-dir>'`. Recommend to Lawrie: **keep all three vzdumps until they've been happy with the system for a couple of weeks**, then delete to reclaim space (that deletion is theirs to trigger — do not delete now).
- Remind Lawrie of the deferred items from PLAN.md: NAS build (scheduled backups + Frigate media off the NVMe) and the OpenVINO **NPU** detector experiment now that the GPU baseline is proven.
- Write the final report `~/rebuild/reports/phase-8.md`: walk test result, autostart test result, final resource table (guest/RAM/vCPU/IP), Coding VM destroy confirmation, backup retention decision, open items. Delete `phase-8-progress.md`.

## Verification checklist

- [ ] Walk test passed end-to-end after the host reboot.
- [ ] All guests autostart in order 1→2→3.
- [ ] Coding VM destroyed (after Lawrie's explicit yes); `.180` reservation retired.
- [ ] Workstation at 16GB / 6 vCPU; host shows ~8GB free headroom.
- [ ] vzdump archives retained; Lawrie owns their eventual deletion.
- [ ] `~/rebuild/reports/phase-8.md` written.

## Rollback

Until step 4's destroy, everything is reversible. After it, the Coding VM exists only as its Phase 1 vzdump (`qmrestore` brings it back if something surfaces later) — which is why that archive is kept for weeks, not deleted at closeout.

## Exit criteria

Rebuild complete: HAOS + Frigate (GPU) + workstation, all autostarting, verified end-to-end, old guests gone, backups retained. 🎉
