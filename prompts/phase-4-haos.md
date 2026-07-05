# Phase 4 — New HAOS VM (built first: it hosts the MQTT broker)

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180), with root SSH to the Proxmox host **192.168.68.100** (now PVE 9). This phase builds a fresh Home Assistant OS VM, restores the Phase 1 backup, and stands up the Mosquitto MQTT broker that Frigate (Phase 5) will connect to.

## Target

| Item | Value |
|---|---|
| VMID | **110** (confirm free in `qm list` first; if taken, pick the next free ≥110 and record it) |
| Name | `haos` |
| Resources | 4GB RAM / 2 vCPU / 48GB disk, CPU type `host`, virtio net |
| Firmware | **UEFI (OVMF) — mandatory for HAOS**, machine `q35` |
| IP | `192.168.68.101` via router DHCP reservation |

## Preconditions

- Read `~/rebuild/reports/phase-1.md` (HA backup path!) and `phase-3.md`. Both must exist.
- Verify the HA backup tar is still present and readable: `tar -tf ~/rebuild/salvage/<ha-backup>.tar > /dev/null && echo OK`.
- `ssh root@192.168.68.100 'pveversion'` shows 9.x.

## Steps

### 1. Download the latest HAOS image (on the host)

Find the newest stable release at https://github.com/home-assistant/operating-system/releases — you want the `haos_ova-<version>.qcow2.xz` asset.

```bash
ssh root@192.168.68.100 '
  cd /root && wget -q https://github.com/home-assistant/operating-system/releases/download/<VER>/haos_ova-<VER>.qcow2.xz
  unxz -f haos_ova-<VER>.qcow2.xz && ls -lh haos_ova-<VER>.qcow2
'
```

### 2. Create the VM and import the disk

Use the storage name from the Phase 1 audit (e.g. `local-lvm`) as `<STORE>`:

```bash
ssh root@192.168.68.100 '
  qm create 110 --name haos --memory 4096 --cores 2 --cpu host \
    --machine q35 --bios ovmf --efidisk0 <STORE>:1,efitype=4m \
    --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single \
    --agent enabled=1 --ostype l26 --tablet 0
  qm importdisk 110 /root/haos_ova-<VER>.qcow2 <STORE>
  qm set 110 --scsi0 <STORE>:vm-110-disk-1,discard=on,ssd=1
  qm set 110 --boot order=scsi0
  qm resize 110 scsi0 48G
  qm set 110 --onboot 1 --startup order=1,up=30
  qm config 110
'
```

Check `qm config 110` output carefully: ovmf + efidisk present, scsi0 is the imported disk, boot order scsi0, agent enabled. (HAOS will not boot on SeaBIOS — if it drops to a UEFI shell or no bootable device, the efidisk/bios settings are wrong.)

### 3. DHCP reservation — needs Lawrie

Get the MAC: `ssh root@192.168.68.100 'qm config 110 | grep net0'`. Ask Lawrie to set a DHCP reservation for that MAC to **192.168.68.101** in the router app (the old HAOS reservation may need updating to the new MAC, or deleting + recreating). **Wait for confirmation before booting**, so HAOS comes up on .101 first try.

### 4. First boot

```bash
ssh root@192.168.68.100 'qm start 110'
```

HAOS takes several minutes on first boot. Poll: `ping -c1 192.168.68.101` and then `curl -s -o /dev/null -w "%{http_code}" http://192.168.68.101:8123` until it returns `200` (onboarding page). Also confirm the guest agent responds: `ssh root@192.168.68.100 'qm agent 110 ping'`.

### 5. Restore the Phase 1 full backup

On the onboarding screen there is an **"Restore from backup"** option — this is the cleanest path:

1. Ask Lawrie to open `http://192.168.68.101:8123`, choose restore-from-backup, and upload the salvaged tar (tell them its exact path; if they're on another machine, serve it briefly: `cd ~/rebuild/salvage && python3 -m http.server 8000` and give them `http://192.168.68.180:8000/<file>` — stop the server afterwards).
2. Full restore reboots HA and takes a while. Poll `:8123` until the real login page returns; Lawrie logs in with their old credentials.
3. Confirm with Lawrie: integrations, devices, automations, and add-ons are back.

### 6. Mosquitto broker + Frigate MQTT user

- If the Mosquitto broker add-on was in the backup it's already installed — verify it's **started** and set to start on boot (Settings → Add-ons → Mosquitto broker). Otherwise ask Lawrie to install it from the official add-on store and start it.
- Create the MQTT user for Frigate: Settings → People → Users → Add user, username `frigate`, a strong generated password, *local access only* is fine. (Mosquitto add-on authenticates against HA users.)
- Record the credentials in `~/rebuild/salvage/mqtt-frigate-credentials.txt` with mode 600 — Phase 5 needs them.
- Test from this VM (install client if needed, `sudo apt install -y mosquitto-clients`):

```bash
mosquitto_sub -h 192.168.68.101 -p 1883 -u frigate -P '<password>' -t 'test/#' -C 1 -W 5 &
mosquitto_pub -h 192.168.68.101 -p 1883 -u frigate -P '<password>' -t test/hello -m ok
```

The subscriber must print `ok`.

### 7. Cleanup + handoff report

`ssh root@192.168.68.100 'rm /root/haos_ova-<VER>.qcow2'` (image no longer needed; the vzdump archives stay). Write `~/rebuild/reports/phase-4.md`: actual VMID, HAOS version, restore result (what came back, anything missing), Mosquitto status, MQTT credential file path, MAC + reservation confirmed.

## Verification checklist

- [ ] VM 110: ovmf/q35, 4GB/2vCPU/48GB, onboot order=1, agent responding.
- [ ] HA reachable at `http://192.168.68.101:8123`, restored config confirmed by Lawrie.
- [ ] Mosquitto running; `frigate` user pub/sub test passed from this VM.
- [ ] Credentials saved (mode 600) for Phase 5.
- [ ] `~/rebuild/reports/phase-4.md` written.

## Rollback

`qm stop 110 && qm destroy 110 --purge` and start the phase over — nothing else depends on it yet. The HA backup tar is read-only salvage; never modify or move the originals.

## Exit criteria

HA restored and healthy on .101 with a working MQTT broker + `frigate` user. Next phase: `phase-5-frigate.md`, same machine.
