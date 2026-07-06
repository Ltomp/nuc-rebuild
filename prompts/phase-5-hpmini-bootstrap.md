# Phase 5 — HP Pro Mini 400 G9: Proxmox bootstrap

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180), with root SSH already working to the NUC's Proxmox host. This phase brings up a **second, independent Proxmox VE host** on the HP Pro Mini 400 G9 (i7-12700T, 32GB DDR4, Intel UHD 770 iGPU, NVMe + SATA bay) and establishes the same passwordless-root-SSH pattern used for the NUC in Phase 0. This machine will run Frigate (Phase 6), nothing else in this rebuild.

This is **not a Proxmox cluster** with the NUC — two independent standalone Proxmox hosts. No shared storage, no corosync, no justification for that complexity with only one guest planned here.

## Preconditions

- Read `~/rebuild/reports/phase-4.md` — confirms HA is healthy on the Pi 5 (not strictly required for this phase, but establishes rebuild sequencing).
- This phase does not touch the NUC or Pi 5 at all.

## Steps

### 1. Physical install — Lawrie's steps

Ask Lawrie to:
1. Download the Proxmox VE ISO from proxmox.com and write it to a USB installer (e.g. with `balenaEtcher` or `dd`).
2. Boot the HP Mini from the USB installer, run through the Proxmox installer: target disk the NVMe, set a root password, and — important — set the network configuration with a **static IP of 192.168.68.110** during install (simpler than a DHCP reservation for a Proxmox host, consistent with how the NUC's Proxmox host is addressed).
3. Confirm once the install finishes and the host reboots into the Proxmox web UI (`https://192.168.68.110:8006`).

### 2. Confirm reachability

```bash
ping -c1 192.168.68.110
curl -sk -o /dev/null -w "%{http_code}" https://192.168.68.110:8006
```

Expected: ping succeeds, `200` (or a redirect code) from the web UI port.

### 3. SSH key authorization (same pattern as Phase 0)

```bash
ls -l ~/.ssh/id_ed25519.pub   # reuse the same keypair already generated in Phase 0 — do not create a second one
ssh -o BatchMode=yes -o ConnectTimeout=5 root@192.168.68.110 'hostname' && echo ALREADY-AUTHORIZED
```

If not already authorized, print the same one-liner pattern as Phase 0 and give it to Lawrie to paste into the HP Mini's Proxmox web UI **Datacenter → node → Shell**:

```bash
PUB=$(cat ~/.ssh/id_ed25519.pub)
echo "mkdir -p /root/.ssh && chmod 700 /root/.ssh && echo '$PUB' >> /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys && echo KEY-ADDED"
```

Wait for Lawrie's confirmation, then verify:

```bash
ssh -o BatchMode=yes root@192.168.68.110 'hostname && pveversion'
```

### 4. Verify the UHD 770 iGPU is visible

```bash
ssh root@192.168.68.110 '
  ls -l /dev/dri/
  lspci -nnk | grep -A3 -i vga
  lsmod | grep -E "^i915|^xe"
'
```

Expected: `/dev/dri/renderD128` present, UHD 770 identified, a driver claimed it (`i915` is normal and fine here — Alder Lake-era UHD 770 doesn't need the newer Xe driver the way Meteor Lake's Arc does; either works for VAAPI/OpenVINO).

### 5. Basic host hygiene

```bash
ssh root@192.168.68.110 '
  apt update && apt -y dist-upgrade
  apt install -y intel-gpu-tools vainfo
  vainfo 2>&1 | head -25
'
```

Confirm `vainfo` lists H264/HEVC decode entrypoints cleanly (UHD 770 has strong Quick Sync media decode — expect this to just work).

### 6. Storage check

```bash
ssh root@192.168.68.110 'pvesm status; lsblk -o NAME,SIZE,TYPE,MOUNTPOINT'
```

Confirm the NVMe is the Proxmox root/VM storage. Note whether the SATA bay has a drive installed — not used in this rebuild (NAS/backups deferred per Lawrie), but worth recording for future reference.

### 7. Handoff report

Write `~/rebuild/reports/phase-5.md`: HP Mini's PVE version, confirmed static IP `.110`, SSH key auth working, UHD 770 driver + vainfo output, storage layout, whether a SATA drive is present.

## Verification checklist

- [ ] Proxmox VE installed and reachable at `192.168.68.110:8006`.
- [ ] Passwordless root SSH working from this VM (same keypair as the NUC).
- [ ] `/dev/dri/renderD128` present; `vainfo` clean.
- [ ] `~/rebuild/reports/phase-5.md` written.

## Rollback

This is a fresh install with nothing else depending on it yet — if something's wrong, Lawrie can just re-run the Proxmox installer.

## Exit criteria

A second, independent Proxmox host is live and reachable at `.110` with working SSH and a verified iGPU. Next phase: `phase-6-hpmini-frigate.md`, same machine (HP Mini), driven from here.
