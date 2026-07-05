# Phase 5 — New Frigate LXC on the Arc iGPU

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180), with root SSH to the Proxmox host **192.168.68.100** (PVE 9). This phase builds an unprivileged Debian 13 LXC running Frigate in Docker, with **VAAPI decode and OpenVINO GPU detection on the Arc iGPU**, and brings the 4 cameras back online. The salvaged old config is at `~/rebuild/salvage/frigate-config.yml`; MQTT credentials from Phase 4 are in `~/rebuild/salvage/mqtt-frigate-credentials.txt`.

## Target

| Item | Value |
|---|---|
| CTID | **111** (confirm free; else next free, record it) |
| Type | Debian 13 LXC, **unprivileged**, nesting on |
| Resources | 4GB RAM / 4 vCPU / 32GB rootfs + dedicated media volume |
| GPU | `/dev/dri/renderD128` passed via `dev0:` entry |
| IP | `192.168.68.102` via DHCP reservation |

## Preconditions

- Read reports for phases 1, 3, 4. `phase-4.md` must confirm MQTT works.
- `ssh root@192.168.68.100 'ls -l /dev/dri/renderD128'` exists (Phase 3 verified).
- Salvaged `frigate-config.yml` present and contains the 4 cameras.

## Steps

### 1. Create the container

```bash
ssh root@192.168.68.100 '
  pveam update && pveam available --section system | grep debian-13
  pveam download local debian-13-standard_<latest>_amd64.tar.zst
  pct create 111 local:vztmpl/debian-13-standard_<latest>_amd64.tar.zst \
    --hostname frigate --unprivileged 1 --features nesting=1,keyctl=1 \
    --memory 4096 --cores 4 --rootfs <STORE>:32 \
    --net0 name=eth0,bridge=vmbr0,ip=dhcp --ostype debian \
    --onboot 1 --startup order=2,up=20
'
```

Add a dedicated media volume (size it from Phase 1's audit of old media usage + free space — e.g. 300–500G on the 2TB NVMe; discuss with Lawrie if unsure):

```bash
ssh root@192.168.68.100 'pct set 111 --mp0 <STORE>:<SIZE_GB>,mp=/media/frigate,backup=0'
```

### 2. GPU passthrough into the unprivileged CT

Start the CT once, find the `render` group gid **inside** it, then wire the device with that gid:

```bash
ssh root@192.168.68.100 'pct start 111 && sleep 5 && pct exec 111 -- getent group render'
# e.g. "render:x:104:" → gid 104
ssh root@192.168.68.100 'pct set 111 --dev0 /dev/dri/renderD128,gid=<render_gid>'
ssh root@192.168.68.100 'pct reboot 111 && sleep 5 && pct exec 111 -- ls -ln /dev/dri/'
```

Expected: `renderD128` exists inside the CT, group-owned by the render gid. (The `dev0:` mechanism does the idmap for you — no manual `lxc.cgroup2.devices.allow` needed on PVE 9.)

### 3. DHCP reservation — needs Lawrie

`ssh root@192.168.68.100 'pct config 111 | grep net0'` → give Lawrie the MAC (`hwaddr=`), ask for a reservation to **192.168.68.102** (replacing the old Frigate one), then `pct reboot 111` and confirm the CT gets .102.

### 4. Docker inside the CT

```bash
ssh root@192.168.68.100 'pct exec 111 -- bash -c "
  apt update && apt install -y ca-certificates curl
  install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
  echo \"deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable\" > /etc/apt/sources.list.d/docker.list
  apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
  docker run --rm hello-world
"'
```

`hello-world` must succeed (nesting=1 makes this work unprivileged).

### 5. Frigate compose + config

Read the salvaged `~/rebuild/salvage/frigate-config.yml` for camera names, RTSP URLs/credentials (main + substream), zones and masks. Build the new config **properly** rather than copying the old one verbatim.

`/opt/frigate/docker-compose.yml` inside the CT:

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    shm_size: "256mb"            # ample for 4 cameras detecting at ~720p
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
      - TZ=<Lawrie's timezone, e.g. Australia/Brisbane — confirm>
```

`/opt/frigate/config/config.yml` skeleton — one entry per camera, repeated 4×:

```yaml
mqtt:
  host: 192.168.68.101
  user: frigate
  password: "<from mqtt-frigate-credentials.txt>"

go2rtc:
  streams:
    cam1:      ["rtsp://<user>:<pass>@<cam1-ip>/<main-path>"]
    cam1_sub:  ["rtsp://<user>:<pass>@<cam1-ip>/<sub-path>"]
    # cam2..cam4 same pattern — ONE connection per camera stream, everything else pulls from go2rtc

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
    days: <size from media volume: roughly days = 0.8 * volume_GB / (cams * GB_per_cam_per_day); continuous ~10–20GB/cam/day at 1080p — compute and confirm with Lawrie>
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

Notes: keep camera names from the old config so restored HA entities line up where possible; port zones/masks coordinates only if detect resolution matches the old config, otherwise rescale or redraw later in the UI (note which in the report). Then:

```bash
ssh root@192.168.68.100 'pct exec 111 -- bash -c "cd /opt/frigate && docker compose up -d && sleep 20 && docker logs frigate 2>&1 | tail -40"'
```

Logs must show go2rtc connecting to all 4 cameras, VAAPI in use, and the OpenVINO detector starting on GPU — no crash loops.

### 6. Verify GPU acceleration end-to-end

```bash
ssh root@192.168.68.100 'timeout 10 intel_gpu_top -o -'   # run on the HOST
```

Expected: non-zero **Video** engine load (decode) and periodic **Render/Compute** activity (OpenVINO inference). In the Frigate UI (`http://192.168.68.102:5000` → System metrics): detector inference speed **~15–25 ms** (vs 100ms+ CPU), detector CPU low, all 4 cameras streaming. Ask Lawrie to eyeball the four live views and confirm zones/masks look right.

### 7. Handoff report

Write `~/rebuild/reports/phase-5.md`: CTID, render gid used, media volume size + retention math, camera names mapped old→new, inference speed observed, `intel_gpu_top` observations, anything ported vs. deferred (masks/zones), UI URL.

## Verification checklist

- [ ] CT 111 unprivileged, nesting, `renderD128` visible inside with correct gid.
- [ ] CT on `.102`; onboot order=2.
- [ ] All 4 cameras live in Frigate UI; go2rtc one-connection-per-camera confirmed in logs.
- [ ] Inference on OpenVINO/GPU at ~15–25 ms; `intel_gpu_top` shows video+render load; CT CPU modest.
- [ ] MQTT: Frigate shows connected (logs), and `frigate/available` topic publishes `online` (verify with `mosquitto_sub` from this VM).
- [ ] `~/rebuild/reports/phase-5.md` written.

## Rollback

`pct stop 111 && pct destroy 111 --purge` and re-run the phase. The salvaged config is never modified in place — work on copies.

## Exit criteria

Cameras recording again, GPU-accelerated, publishing to MQTT. Next phase: `phase-6-ha-frigate.md`, same machine.
