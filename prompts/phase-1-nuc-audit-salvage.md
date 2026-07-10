# Phase 1 — NUC audit + salvage (before touching anything)

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with passwordless root SSH to the Proxmox host **192.168.68.100** (established in Phase 0). This phase is **strictly read-and-copy**: audit the host, and salvage everything needed to rebuild Home Assistant and Frigate elsewhere — **and everything needed to rebuild this Coding VM's own Claude Code setup** — before Phase 2 wipes this entire host from Proxmox to bare-metal Debian. **You must not stop, reconfigure, or delete any guest in this phase, and you must not begin the wipe.**

## Why this phase matters more than a normal "before we delete some VMs" salvage

Every previous phase in this rebuild destroyed individual guests one at a time, with the host and this session surviving. **Phase 2 is different: it wipes the entire physical disk**, taking the current Proxmox install, the old HAOS VM, the old Frigate container, and this Coding VM (and this very session) with it, all at once, permanently. There is no picking through the wreckage afterward. Anything not copied off this host before Phase 2 begins is gone. Treat this phase's verification steps as non-negotiable gates, not formalities.

## Context

- Old HAOS VM: VMID ≈100, IP `.101` (confirm the real VMID in the audit).
- Old Frigate: a container at `.102` running 4 cameras (confirm CTID and whether it's an LXC running Docker).
- Coding VM: VMID 102 — the VM you are on. **This also does not survive Phase 2** and must be salvaged like the other two.
- Salvage destinations: `/root/rebuild-salvage/` on the host (survives only until Phase 2 — this is *not* an off-NUC copy, just a staging convenience) and `~/rebuild/salvage/` here (same caveat). The real off-NUC destinations are established in step 4 below.

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

1. Ask Lawrie to trigger **Settings → System → Backups → Create backup (full)** in the HA UI at `http://192.168.68.101:8123`, or do it yourself if a long-lived access token is available. **It must be a Full backup, not a partial/config-only one** — this HA install runs Zigbee2MQTT, Matter Server, ESPHome, and the voice assist add-ons, and only a Full backup captures their add-on data (Zigbee2MQTT's paired-device database, Matter fabric certificates, ESPHome secrets) alongside the core config. A partial backup would silently lose all of that, forcing a re-pair of 51 Zigbee devices and re-commissioning of Matter devices later.
2. Once complete, download it. Easiest: Lawrie downloads from the HA UI and you fetch it, or pull over Samba/SSH add-on if configured. If no path works, ask Lawrie how they'd like to hand you the file. The goal is non-negotiable: **a full-backup `.tar` copied to `~/rebuild/salvage/`, then out to the off-NUC destination chosen in step 4.** Expect this to be large (the recorder DB alone was ~420MB at last check, still growing) — budget transfer time and disk space accordingly, and don't be surprised by a multi-hundred-MB to multi-GB file.
3. Verify: file exists, size > 100MB (a suspiciously small file likely means a partial backup was taken by mistake — stop and redo it as Full), and `tar -tf` on it lists contents without error.

### 3. Salvage Frigate config

The Frigate config contains the camera RTSP URLs, credentials, zones and masks — the single most important salvage item for the camera rebuild.

```bash
# Find it (adjust CTID from the audit; config is typically bind-mounted or inside the CT)
ssh root@192.168.68.100 'pct exec <CTID> -- find / -name "config.yml" -path "*frigate*" 2>/dev/null'
ssh root@192.168.68.100 'pct exec <CTID> -- cat <path-to-config.yml>' > ~/rebuild/salvage/frigate-config.yml
```

Also record current media usage inside the CT (`du -sh` on the media dir) and copy `docker-compose.yml`/run command if present.

Verify the salvaged config actually contains 4 cameras and their `rtsp://` URLs (grep it; do not print full credentials into the chat, just confirm presence).

### 4. Salvage this Coding VM itself — the new, critical step

**This VM does not survive Phase 2.** Everything below must land somewhere that isn't this host, before the wipe.

First, ask Lawrie where salvage should go. Two reasonable options — pick based on what's actually available right now:
- **Lawrie's new laptop**, if it's on the same LAN and reachable via SSH/`scp` (simplest if available — ask for its IP and confirm reachability: `ping -c1 <laptop-ip>`).
- **A USB drive**, physically attached to whatever machine Lawrie is using to interact with this session, if the laptop route isn't available.

Whichever is chosen, this repo (`~/nuc-rebuild`) is **already partly covered** — verify it's fully pushed rather than re-copying it:

```bash
cd ~/nuc-rebuild && git status -sb && git log origin/main -1
```

If `git status` shows anything not pushed, commit and push it now.

For everything else, build a single archive and copy it to the chosen destination:

```bash
mkdir -p ~/rebuild/salvage/coding-vm
tar czf ~/rebuild/salvage/coding-vm/coding-vm-home.tar.gz \
  -C ~ \
  .claude \
  claude \
  awa \
  .ssh \
  .gitconfig \
  .git-credentials \
  .claude.json \
  .npmrc \
  .config \
  rebuild \
  2>&1 | grep -v "Removing leading"   # some of these paths may not all exist — that's fine, tar skips missing ones
```

Also capture things that live outside a simple tar of `~`:

```bash
npm ls -g --depth=0 > ~/rebuild/salvage/coding-vm/npm-globals.txt 2>&1
crontab -l > ~/rebuild/salvage/coding-vm/crontab.txt 2>/dev/null || echo "no crontab" > ~/rebuild/salvage/coding-vm/crontab.txt
claude --version > ~/rebuild/salvage/coding-vm/claude-version.txt 2>&1
```

**This archive contains SSH private keys, git credentials, and MCP tokens.** Copy it only to the destination Lawrie chose, over a channel you both trust (SSH/scp to the laptop is fine on a home LAN; do not push it to GitHub or any other third-party service, even a private repo).

```bash
# example if the laptop route was chosen:
scp ~/rebuild/salvage/coding-vm/coding-vm-home.tar.gz lawrie@<laptop-ip>:~/nuc-rebuild-salvage/
scp ~/rebuild/salvage/coding-vm/npm-globals.txt ~/rebuild/salvage/coding-vm/crontab.txt lawrie@<laptop-ip>:~/nuc-rebuild-salvage/
```

Verify the copy landed and is intact:

```bash
ssh lawrie@<laptop-ip> 'ls -lh ~/nuc-rebuild-salvage/ && tar tzf ~/nuc-rebuild-salvage/coding-vm-home.tar.gz > /dev/null && echo TAR-OK'
```

Do not proceed to Phase 2 until `TAR-OK` prints. This is the single most important verification in this whole phase.

### 5. vzdump backups (belt-and-braces, but not the real safety net for Phase 2)

These vzdump archives live on the *same disk* Phase 2 wipes — they protect against a mistake *within* this phase (e.g. destroying the wrong guest by accident, which no longer happens in the new plan, but keep the habit), not against the wipe itself. The real safety net is step 4's off-NUC copy.

First check free space on the target storage (from the audit — use the storage with the most free space, typically `local`):

```bash
ssh root@192.168.68.100 'df -h /var/lib/vz'
```

Then back up all three guests:

```bash
ssh root@192.168.68.100 'mkdir -p /root/rebuild-salvage && vzdump <HAOS_VMID> <FRIGATE_CTID> 102 --mode snapshot --compress zstd --storage <storage> --notes-template "pre-wipe {{guestname}}"'
```

`--mode snapshot` keeps guests running (the Coding VM can back itself up this way).

Verify: `ssh root@192.168.68.100 'ls -lh /var/lib/vz/dump/'` shows three fresh archives with plausible sizes. **These are for reference only if something looks wrong before you commit to Phase 2 — do not treat them as recoverable after the wipe.**

### 6. Handoff report

Write `~/rebuild/reports/phase-1.md` with:
- Real VMIDs/CTIDs, storage names + free space, `/dev/dri` status, PVE version.
- Full paths + sizes of: HA backup tar, frigate-config.yml, the Coding VM archive — **and confirmation each one landed on the chosen off-NUC destination, not just this host.**
- Camera count and stream types found in the Frigate config (no credentials).
- The off-NUC destination chosen (laptop IP or USB) — Phase 2's pre-flight check re-verifies this.
- Any deviations or concerns.

## Verification checklist

- [ ] Audit file saved; real VMIDs recorded.
- [ ] HA full backup tar salvaged, `tar -tf` clean, size plausible.
- [ ] `frigate-config.yml` salvaged, contains 4 cameras.
- [ ] Coding VM archive built, copied to the off-NUC destination, and verified intact there (`TAR-OK`).
- [ ] `nuc-rebuild` git repo fully pushed (`git status` clean against `origin/main`).
- [ ] Three vzdump archives on host storage, sizes plausible (belt-and-braces only).
- [ ] `~/rebuild/reports/phase-1.md` written, including the off-NUC destination used.

## Rollback

Nothing was modified — this phase only reads and copies. Worst case: delete the copies and redo them.

## Exit criteria

All salvage verified **off this host**, report written. **Cameras and HA are still running; this Coding VM is still running.** Next phase: `phase-2-nuc-bare-metal-install.md` — **this is the point of no return for this host.**
