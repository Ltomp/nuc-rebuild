# Phase 2 — Shrink Coding VM (config only) + teardown of old guests

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with root SSH to the Proxmox host **192.168.68.100**. This phase does two things: (a) *configures* — but does not apply — the shrink of this VM, and (b) **destroys the old HAOS VM and old Frigate container** after Lawrie's explicit confirmation. This is the first destructive phase. Backups from Phase 1 are the safety net.

## Context

- The Coding VM shrinks to **6GB RAM / 4 vCPU**. The change is applied by the **host reboot in Phase 3**, not here — do **not** reboot or stop VMID 102 in this phase (you are running inside it; one interruption total is the design).
- After the old guests are destroyed, **the cameras and Home Assistant go dark until Phases 4–5.** Lawrie must acknowledge this.
- Real VMIDs come from the Phase 1 report — never guess them.

## Preconditions

- Read `~/rebuild/reports/phase-0.md` and `phase-1.md`. If phase-1.md is missing or its verification items aren't all confirmed, **stop and ask Lawrie**.
- Re-verify the safety net yourself (do not trust the report blindly):

```bash
ssh root@192.168.68.100 'ls -lh <dump-dir>'   # three vzdump archives present, sizes > 0
ls -lh ~/rebuild/salvage/                      # HA backup tar + frigate-config.yml present
```

If any backup is missing, **stop**. Do not destroy anything.

## Steps

### 1. Configure the Coding VM shrink (no reboot)

```bash
ssh root@192.168.68.100 'qm set 102 --memory 6144 --cores 4'
ssh root@192.168.68.100 'qm config 102 | grep -E "memory|cores"'
```

Expected: config now shows `memory: 6144`, `cores: 4`. The running VM keeps its current allocation until next power-off/on — that's intended. Confirm the VM is still running normally (`free -h` locally still shows the old size — correct at this point).

### 2. Destroy the old HAOS VM — GATED

Present Lawrie with exactly this, filling in real values, and wait for an explicit "yes":

> About to permanently destroy the old Home Assistant VM: **VMID `<HAOS_VMID>`, name `<name>`, IP .101**. Its vzdump backup `<filename, size, date>` exists at `<path>` and the HA full backup tar is salvaged in two places. After this, Home Assistant is offline until Phase 4. Confirm destroy? (yes/no)

Only after "yes":

```bash
ssh root@192.168.68.100 'qm stop <HAOS_VMID> && qm destroy <HAOS_VMID> --purge --destroy-unreferenced-disks 1'
```

Verify: `qm list` no longer shows it; `pvesm status` reflects freed space.

### 3. Destroy the old Frigate container — GATED

Same protocol, second confirmation (one gate per guest — never batch them):

> About to permanently destroy the old Frigate container: **CTID `<CTID>`, IP .102**. Its vzdump backup `<filename, size, date>` exists, and `frigate-config.yml` is salvaged in two places. **All 4 cameras stop recording now until Phase 5.** Confirm destroy? (yes/no)

Only after "yes":

```bash
ssh root@192.168.68.100 'pct stop <CTID>; pct destroy <CTID> --purge --destroy-unreferenced-disks 1'
```

Verify with `pct list`.

### 4. Post-teardown state check

```bash
ssh root@192.168.68.100 'qm list; pct list; pvesm status; free -h'
```

Expected: only VMID 102 remains. Record freed disk space.

### 5. Handoff report

Write `~/rebuild/reports/phase-2.md`: shrink configured (6144/4, pending reboot), which guests were destroyed (IDs, timestamps, Lawrie's confirmations), storage freed, and a reminder line: **"Coding VM shrink applies at the Phase 3 host reboot."**

## Verification checklist

- [ ] `qm config 102` shows `memory: 6144`, `cores: 4`; VM 102 still running untouched.
- [ ] Old HAOS VM and old Frigate CT gone from `qm list` / `pct list`.
- [ ] Both destroys were individually confirmed by Lawrie beforehand.
- [ ] vzdump archives still present on host storage (untouched).
- [ ] `~/rebuild/reports/phase-2.md` written.

## Rollback

- Shrink: `ssh root@192.168.68.100 'qm set 102 --memory <old> --cores <old>'` (old values are in the Phase 1 audit).
- Destroyed guests: restore from vzdump — `qmrestore <archive> <vmid>` / `pct restore <ctid> <archive>`. This is why the archives must never be deleted in this phase.

## Exit criteria

Old guests destroyed, shrink staged, backups intact, report written. Next phase: `phase-3-host-upgrade.md`, same machine — **that phase includes the one planned session interruption.**
