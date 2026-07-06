# Phase 6 — HP Mini: Frigate LXC on the UHD 770

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180), with root SSH to the HP Mini's Proxmox host **192.168.68.110** (Phase 5). This phase builds an unprivileged Debian 13 LXC running Frigate in Docker, with **VAAPI decode and OpenVINO GPU detection on the Intel UHD 770**, and brings the 4 cameras back online. The salvaged old config is at `~/rebuild/salvage/frigate-config.yml` (from Phase 1, on the Coding VM); MQTT now lives on the **Pi 5** at `.101` (Phase 4), not the NUC.

## Target

| Item | Value |
|---|---|
| CTID | **111** (confirm free; else next free, record it) |
| Type | Debian 13 LXC, **unprivileged**, nesting on |
| Resources | **8GB RAM / 6 vCPU / 100GB rootfs** + dedicated media volume — the HP Mini has real headroom (32GB total, nothing else running on it), so there's no need to size this tightly the way the original NUC-hosted design had to. |
| GPU | `/dev/dri/renderD128` passed via `dev0:` entry (UHD 770) |
| IP | `192.168.68.102` via DHCP reservation (reused) |

## Preconditions

- Read `~/rebuild/reports/phase-1.md` (salvaged config), `phase-4.md` (Pi 5 / MQTT broker healthy), `phase-5.md` (HP Mini Proxmox + iGPU verified).
- `ssh root@192.168.68.110 'ls -l /dev/dri/renderD128'` exists.
- Salvaged `frigate-config.yml` present and contains the 4 cameras.
- MQTT credentials from the original HA setup — check `~/rebuild/salvage/` for a saved `mqtt-frigate-credentials.txt` from prior work, or ask Lawrie to create a fresh `frigate` MQTT user in HA (now on the Pi 5) at Settings → People → Users if one doesn't already exist post-restore.

## Steps

### 1. Create the container

```bash
ssh root@192.168.68.110 '
  pveam update && pveam available --section system | grep debian-13
  pveam download local debian-13-standard_<latest>_amd64.tar.zst
  pct create 111 local:vztmpl/debian-13-standard_<latest>_amd64.tar.zst \
    --hostname frigate --unprivileged 1 --features nesting=1,keyctl=1 \
    --memory 8192 --cores 6 --rootfs local-lvm:100 \
    --net0 name=eth0,bridge=vmbr0,ip=dhcp --ostype debian \
    --onboot 1 --startup order=1,up=20
'
```

(Adjust the storage name from the Phase 5 audit if it isn't `local-lvm`.) Add a dedicated media volume — with the HP Mini's headroom, be generous; check available space first and size it to leave meaningful retention (discuss exact GB with Lawrie based on what's actually free):

```bash
ssh root@192.168.68.110 'pvesm status'   # check free space, then:
ssh root@192.168.68.110 'pct set 111 --mp0 local-lvm:<SIZE_GB>,mp=/media/frigate,backup=0'
```

### 2. GPU passthrough into the unprivileged CT

Start the CT once, find the `render` group gid **inside** it, then wire the device with that gid:

```bash
ssh root@192.168.68.110 'pct start 111 && sleep 5 && pct exec 111 -- getent group render'
# e.g. "render:x:104:" → gid 104
ssh root@192.168.68.110 'pct set 111 --dev0 /dev/dri/renderD128,gid=<render_gid>'
ssh root@192.168.68.110 'pct reboot 111 && sleep 5 && pct exec 111 -- ls -ln /dev/dri/'
```

Expected: `renderD128` exists inside the CT, group-owned by the render gid.

### 3. DHCP reservation — needs Lawrie

`ssh root@192.168.68.110 'pct config 111 | grep net0'` → give Lawrie the MAC (`hwaddr=`), ask for a reservation to **192.168.68.102** (reused from the original Frigate container), then `pct reboot 111` and confirm the CT gets `.102`.

### 4. Docker inside the CT

```bash
ssh root@192.168.68.110 'pct exec 111 -- bash -c "
  apt update && apt install -y ca-certificates curl
  install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
  echo \"deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable\" > /etc/apt/sources.list.d/docker.list
  apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
  docker run --rm hello-world
"'
```

### 5. Frigate compose + config

Read the salvaged `~/rebuild/salvage/frigate-config.yml` for camera names, RTSP URLs/credentials (main + substream), zones and masks. Build the new config properly rather than copying the old one verbatim.

`/opt/frigate/docker-compose.yml` inside the CT:

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    shm_size: "256mb"
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    volumes:
      - /opt/frigate/config:/config
      - /media/frigate:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1g
    ports:
      - "5000:5000"    # UI/API
      - "8554:8554"    # go2rtc RTSP
      - "8971:8971"    # authenticated UI (0.14+)
    environment:
      - TZ=<Lawrie's timezone — confirm>
```

`/opt/frigate/config/config.yml` skeleton — **MQTT now points at the Pi 5**, not the old NUC HAOS VM:

```yaml
mqtt:
  host: 192.168.68.101      # the Pi 5, not .100 or a NUC guest
  user: frigate
  password: "<frigate MQTT user password>"

go2rtc:
  streams:
    cam1:      ["rtsp://<user>:<pass>@<cam1-ip>/<main-path>"]
    cam1_sub:  ["rtsp://<user>:<pass>@<cam1-ip>/<sub-path>"]
    # cam2..cam4 same pattern — ONE connection per camera stream

ffmpeg:
  hwaccel_args: preset-vaapi

detectors:
  ov:
    type: openvino
    device: GPU

model:
  width: 300
  height: 300
  input_tensor: nhwc
  input_pixel_format: bgr
  path: /openvino-model/ssdlite_mobilenet_v2.xml
  labelmap_path: /openvino-model/coco_91cl_bkgr.txt

record:
  enabled: true
  retain:
    days: <compute from the actual media volume size and camera count; the HP Mini's headroom means this can likely be more generous than the original NUC-constrained plan — discuss with Lawrie>
    mode: motion
  alerts: { retain: { days: 14 } }
  detections: { retain: { days: 14 } }

snapshots:
  enabled: true
  retain: { default: 14 }

cameras:
  cam1:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/cam1_sub
          roles: [detect]
        - path: rtsp://127.0.0.1:8554/cam1
          roles: [record]
    detect: { width: 1280, height: 720, fps: 5 }
    # port over this camera's zones/masks from the salvaged config here
  # cam2..cam4 same pattern
```

Then:

```bash
ssh root@192.168.68.110 'pct exec 111 -- bash -c "cd /opt/frigate && docker compose up -d && sleep 20 && docker logs frigate 2>&1 | tail -40"'
```

Logs must show go2rtc connecting to all 4 cameras, VAAPI in use, and the OpenVINO detector starting on GPU — no crash loops.

### 6. Verify GPU acceleration end-to-end

```bash
ssh root@192.168.68.110 'timeout 10 intel_gpu_top -o -'   # run on the HP Mini HOST
```

Expected: non-zero **Video** engine load (decode) and periodic **Render** activity (OpenVINO inference). In the Frigate UI (`http://192.168.68.102:5000` → System metrics): detector inference speed — UHD 770 is a solid media/decode iGPU; note the actual figure rather than assuming NUC-Arc-class numbers, since Quick Sync media decode is excellent but raw compute is more modest than Arc's Xe-LPG. All 4 cameras streaming, low CT CPU. Ask Lawrie to eyeball the four live views and confirm zones/masks look right.

### 7. Handoff report

Write `~/rebuild/reports/phase-6.md`: CTID, render gid used, media volume size + retention chosen, camera names mapped old→new, inference speed observed, `intel_gpu_top` observations, anything ported vs. deferred (masks/zones), UI URL, MQTT connectivity confirmed to the Pi 5.

## Verification checklist

- [ ] CT 111 unprivileged, nesting, `renderD128` visible inside with correct gid, on the HP Mini (not the NUC).
- [ ] CT on `.102`; onboot enabled.
- [ ] All 4 cameras live in Frigate UI; go2rtc one-connection-per-camera confirmed in logs.
- [ ] OpenVINO/GPU detector running on the UHD 770; `intel_gpu_top` shows video+render load; CT CPU modest.
- [ ] MQTT: Frigate connects to the **Pi 5** (`.101`), confirmed via logs and `mosquitto_sub -h 192.168.68.101 ... -t 'frigate/available'` showing `online`.
- [ ] `~/rebuild/reports/phase-6.md` written.

## Rollback

`pct stop 111 && pct destroy 111 --purge` and re-run the phase. The salvaged config is never modified in place — work on copies.

## Exit criteria

Cameras recording again on the HP Mini, GPU-accelerated, publishing to the Pi 5's MQTT broker. Next phase: `phase-7-ha-frigate-integration.md`, back on the Coding VM, orchestrating both the Pi 5 (HA) and this box.
