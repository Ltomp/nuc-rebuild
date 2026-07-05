# Phase 1 — Audit + salvage (before touching anything)

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with passwordless root SSH to the Proxmox host **192.168.68.100** (established in Phase 0). This phase is **strictly read-and-copy**: audit the host, and salvage everything needed to rebuild Home Assistant and Frigate, before anything is deleted in Phase 2. **You must not stop, reconfigure, or delete any guest in this phase.**

## Context

- Old HAOS VM: VMID ≈100, IP `.101` (confirm the real VMID in the audit).
- Old Frigate: a container at `.102` running 4 cameras (confirm CTID and whether it's an LXC running Docker).
- Coding VM: VMID 102 — the VM you are on.
- Salvage destinations: `/root/rebuild-salvage/` on the host and `~/rebuild/salvage/` here.

## Preconditions

- Read `~/rebuild/reports/phase-0.md`. If missing, stop and ask Lawrie.
- `ssh -o BatchMode=yes root@192.168.68.100 'echo ok'` prints `ok`.

## Steps

### 1. Host audit

Run over SSH and save all output to `~/rebuild/reports/phase-1-audit.txt`:

```bash
ssh root@192.168.68.100 '
  pveversion -v | head -5
  echo ---; qm list
  echo ---; pct list
  echo ---; pvesm status
  echo ---; lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
  echo ---; df -h
  echo ---; ls -l /dev/dri/
  echo ---; lspci -nnk | grep -A3 -i vga
  echo ---; cat /etc/apt/sources.list /etc/apt/sources.list.d/*.list 2>/dev/null
  echo ---; free -h
' | tee ~/rebuild/reports/phase-1-audit.txt
```

From the output, record: **real VMID of the old HAOS VM**, **real CTID of the Frigate container**, storage names (e.g. `local`, `local-lvm`) and free space, and whether `/dev/dri/renderD128` exists on the host. Also dump each guest's config:

```bash
ssh root@192.168.68.100 'for id in $(qm list | awk "NR>1{print \$1}"); do echo "== VM $id =="; qm config $id; done; for id in $(pct list | awk "NR>1{print \$1}"); do echo "== CT $id =="; pct config $id; done' | tee -a ~/rebuild/reports/phase-1-audit.txt
```

### 2. Salvage Home Assistant (full backup)

Preferred route — HA CLI via the VM console is fiddly; use the API or ask Lawrie:

1. Ask Lawrie to trigger **Settings → System → Backups → Create backup (full)** in the HA UI at `http://192.168.68.101:8123`, or do it yourself if a long-lived access token is available.
2. Once complete, download it. Easiest: Lawrie downloads from the HA UI and you fetch it, or pull over Samba/SSH add-on if configured. If no path works, ask Lawrie how they'd like to hand you the file. The goal is non-negotiable: **a full-backup `.tar` copied to `~/rebuild/salvage/` AND to `/root/rebuild-salvage/` on the host.**
3. Verify: file exists in both places, size > 10MB, and `tar -tf` on it lists contents without error.

### 3. Salvage Frigate config

The Frigate config contains the camera RTSP URLs, credentials, zones and masks — the single most important salvage item.

```bash
# Find it (adjust CTID from the audit; config is typically bind-mounted or inside the CT)
ssh root@192.168.68.100 'pct exec <CTID> -- find / -name "config.yml" -path "*frigate*" 2>/dev/null'
ssh root@192.168.68.100 'pct exec <CTID> -- cat <path-to-config.yml>' > ~/rebuild/salvage/frigate-config.yml
```

Also record current media usage inside the CT (`du -sh` on the media dir) and copy `docker-compose.yml`/run command if present. Copy `frigate-config.yml` to the host too:

```bash
scp ~/rebuild/salvage/frigate-config.yml root@192.168.68.100:/root/rebuild-salvage/
```

Verify the salvaged config actually contains 4 cameras and their `rtsp://` URLs (grep it; do not print full credentials into the chat, just confirm presence).

### 4. vzdump backups (insurance)

First check free space on the target storage (from the audit — use the storage with the most free space, typically `local`):

```bash
ssh root@192.168.68.100 'df -h /var/lib/vz'
```

Rule of thumb: you need at least the sum of the three guests' used disk. If space is insufficient, stop and discuss options with Lawrie. Then back up **all three** guests — old HAOS, old Frigate, **and this Coding VM** (it carries all of Lawrie's Claude data through the risky Phase 3 host upgrade):

```bash
ssh root@192.168.68.100 'mkdir -p /root/rebuild-salvage && vzdump <HAOS_VMID> <FRIGATE_CTID> 102 --mode snapshot --compress zstd --storage <storage> --notes-template "pre-rebuild {{guestname}}"'
```

Notes: `--mode snapshot` keeps guests running (the Coding VM can back itself up this way). If snapshot mode fails for a guest, fall back to `--mode suspend` for the old guests only — never suspend VMID 102 (you are inside it).

Verify: `ssh root@192.168.68.100 'ls -lh /var/lib/vz/dump/'` (or the chosen storage's dump dir) shows three fresh `vzdump-*.vma.zst` / `.tar.zst` files with plausible sizes.

### 5. Handoff report

Write `~/rebuild/reports/phase-1.md` with:
- Real VMIDs/CTIDs, storage names + free space, `/dev/dri` status, PVE version.
- Full paths + sizes of: HA backup tar (both copies), frigate-config.yml (both copies), three vzdump archives.
- Camera count and stream types found in the Frigate config (no credentials).
- Any deviations or concerns.

## Verification checklist

- [ ] Audit file saved; real VMIDs recorded.
- [ ] HA full backup tar in `~/rebuild/salvage/` and `/root/rebuild-salvage/`, `tar -tf` clean.
- [ ] `frigate-config.yml` salvaged to both locations, contains 4 cameras.
- [ ] Three vzdump archives on host storage, sizes plausible.
- [ ] `~/rebuild/reports/phase-1.md` written.

## Rollback

Nothing was modified — this phase only reads and copies. Worst case: delete the copies.

## Exit criteria

All salvage verified in two places, vzdumps done, report written. **Cameras and HA are still running.** Next phase: `phase-2-shrink-teardown.md`, same machine.
