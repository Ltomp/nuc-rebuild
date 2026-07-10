# Phase 3 — HP Pro Mini 400 G9: bare-metal Debian 13 + Docker

You are Claude Code (Sonnet) running directly on the **NUC** (192.168.68.103, Debian 13, bare-metal — installed in Phase 2). This phase brings up the HP Pro Mini 400 G9 (i7-12700T, 32GB DDR4, Intel UHD 770 iGPU, NVMe + SATA bay) as a second bare-metal Debian machine — no hypervisor, first install (Proxmox was never put on this hardware, so there's nothing to wipe). It will run Frigate (Phase 5), nothing else in this rebuild.

## Preconditions

- Read `~/rebuild/reports/phase-2.md` — confirms you're on the new bare-metal NUC with a working `claude` CLI and restored SSH keys.
- This phase does not touch the Pi 5 or anything already done.

## Steps

### 1. Physical install — Lawrie's steps

Ask Lawrie to:
1. Download the Debian 13 netinst ISO and write it to a USB installer.
2. Boot the HP Mini from the USB installer, run the installer:
   - Hostname: `hpmini`.
   - Network: **static IP `192.168.68.102`** (reused from the original Frigate container's address — this machine's whole job is Frigate, so it gets that identity directly now, no separate host-vs-guest IP split).
   - User: `lawrie`, sudo or root login (match whatever was chosen for the NUC in Phase 2, for consistency).
   - Partitioning: guided, entire NVMe, single ext4 partition.
   - Software selection: SSH server + standard system utilities only, no desktop.
3. Confirm once it's installed and reachable.

### 2. Confirm reachability and SSH key auth

```bash
ping -c1 192.168.68.102
ssh lawrie@192.168.68.102 'echo ok'   # password auth, first connection
ssh-copy-id lawrie@192.168.68.102     # reuse the same keypair already on this NUC
ssh -o BatchMode=yes lawrie@192.168.68.102 'echo KEY-OK'
```

### 3. Basic hygiene + verify the UHD 770 iGPU

```bash
ssh lawrie@192.168.68.102 '
  sudo apt update && sudo apt -y dist-upgrade
  sudo apt install -y intel-gpu-tools vainfo git curl
  ls -l /dev/dri/
  lspci -nnk | grep -A3 -i vga
  vainfo 2>&1 | head -25
'
```

Expected: `/dev/dri/renderD128` present, UHD 770 identified, `vainfo` lists H264/HEVC decode entrypoints cleanly (UHD 770 has strong Quick Sync media decode — this should just work with the in-kernel `i915` driver; unlike the NUC's Arc iGPU, UHD 770 doesn't need the newer Xe driver).

Check the render group's gid on this host — Docker (unlike an unprivileged LXC) usually just needs the container's runtime user to be in the host's `render` group, or the compose file needs `group_add`, so record it now for Phase 5:

```bash
ssh lawrie@192.168.68.102 'getent group render'
```

### 4. Install Docker

```bash
ssh lawrie@192.168.68.102 '
  sudo install -m 0755 -d /etc/apt/keyrings
  sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
  echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable" | sudo tee /etc/apt/sources.list.d/docker.list
  sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
  sudo usermod -aG docker lawrie
  sudo docker run --rm hello-world
'
```

(Log out/in — or just prefix `sudo` for the rest of this rebuild's Docker commands — for the group membership to take effect in this session.)

### 5. Storage check

```bash
ssh lawrie@192.168.68.102 'lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE; df -h /'
```

Confirm the NVMe is the root filesystem with plenty of free space for Frigate media. Note whether the SATA bay has a drive installed — not used in this rebuild (NAS/backups deferred), but worth recording.

### 6. Handoff report

Write `~/rebuild/reports/phase-3.md`: confirmed static IP `.102`, SSH key auth working, root-vs-sudo choice, UHD 770 driver + `vainfo` output, `render` group gid, Docker version, storage layout, SATA bay drive presence.

## Verification checklist

- [ ] HP Mini installed, bare-metal Debian 13, reachable at `192.168.68.102`.
- [ ] Passwordless SSH working from the NUC (same keypair).
- [ ] `/dev/dri/renderD128` present; `vainfo` clean.
- [ ] Docker installed and `hello-world` ran successfully.
- [ ] `~/rebuild/reports/phase-3.md` written.

## Rollback

Fresh install with nothing else depending on it yet — if something's wrong, Lawrie can just re-run the Debian installer.

## Exit criteria

A second bare-metal machine is live and reachable at `.102` with working SSH, Docker, and a verified iGPU. Next phase: `phase-4-pi5-haos.md` — different machine (the Raspberry Pi 5), driven from here over SSH once flashed.
