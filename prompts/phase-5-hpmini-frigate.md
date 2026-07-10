# Phase 5 — HP Mini: Frigate on the UHD 770

You are Claude Code (Sonnet) running on the **NUC** (192.168.68.103), with SSH to the HP Mini (**192.168.68.102**, bare-metal Debian 13 + Docker, set up in Phase 3). This phase runs Frigate directly in Docker on the host — no LXC, no VM, no passthrough config to fight: `/dev/dri/renderD128` is just the HP Mini's own device, handed to the container with a normal Docker `--device` flag. Brings the 4 cameras back online with **VAAPI decode and OpenVINO GPU detection on the Intel UHD 770**. The salvaged old config is at `~/rebuild/salvage/frigate-config.yml` (restored from Phase 1's archive onto this NUC in Phase 2); MQTT now lives on the **Pi 5** at `.101` (Phase 4), not the NUC.

## Preconditions

- Read `~/rebuild/reports/phase-1.md` (salvaged config), `phase-3.md` (HP Mini Docker + iGPU + `render` group gid), `phase-4.md` (Pi 5 / MQTT broker healthy).
- `ssh lawrie@192.168.68.102 'ls -l /dev/dri/renderD128'` exists.
- Salvaged `~/rebuild/salvage/frigate-config.yml` present on this NUC and contains the 4 cameras.
- MQTT credentials: check `~/rebuild/salvage/` for a saved `mqtt-frigate-credentials.txt` from prior work, or ask Lawrie to create a fresh `frigate` MQTT user in HA (now on the Pi 5) at Settings → People → Users if one doesn't already exist post-restore.

## Steps

### 1. Directories and permissions on the HP Mini

```bash
ssh lawrie@192.168.68.102 '
  sudo mkdir -p /opt/frigate/config /media/frigate
  sudo chown -R lawrie:lawrie /opt/frigate /media/frigate
'
```

Check available space and size the media volume against it (the HP Mini has real headroom — 32GB RAM, and check actual free disk — no need to size tightly the way the original NUC-hosted design had to):

```bash
ssh lawrie@192.168.68.102 'df -h /media'
```

### 2. GPU device access

Docker on a bare host is simpler than the LXC pattern this design used elsewhere — no unprivileged-container gid remapping needed. Two options, pick whichever works cleanly:

```bash
# confirm the render group's gid (recorded in phase-3.md too)
ssh lawrie@192.168.68.102 'getent group render'
```

Pass the device straight through in the compose file (below) via `devices:` — if the container's internal user can't access it, add `group_add: ["<render_gid>"]` to the compose service (this is the Docker-native equivalent of the LXC's gid-mapped `dev0` passthrough, and is normally all that's needed).

### 3. Frigate compose + config

Read the salvaged `~/rebuild/salvage/frigate-config.yml` for camera names, RTSP URLs/credentials (main + substream), zones and masks. Build the new config properly rather than copying the old one verbatim, and copy it to the HP Mini:

`/opt/frigate/docker-compose.yml` on the HP Mini:

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    shm_size: "256mb"
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    group_add:
      - "<render_gid from step 2, if needed>"
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
  host: 192.168.68.101      # the Pi 5
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
    days: <compute from the actual media volume size and camera count; the HP Mini's headroom means this can likely be generous — discuss with Lawrie>
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

Copy the config and bring it up:

```bash
scp /opt/frigate/docker-compose.yml lawrie@192.168.68.102:/opt/frigate/docker-compose.yml   # if built locally first, else write directly on the host
ssh lawrie@192.168.68.102 'cd /opt/frigate && docker compose up -d && sleep 20 && docker logs frigate 2>&1 | tail -40'
```

Logs must show go2rtc connecting to all 4 cameras, VAAPI in use, and the OpenVINO detector starting on GPU — no crash loops. If the container can't open `/dev/dri/renderD128` (permission denied), that's the `group_add` gid — fix and `docker compose up -d` again.

### 4. Verify GPU acceleration end-to-end

```bash
ssh lawrie@192.168.68.102 'timeout 10 intel_gpu_top -o -'   # run on the HP Mini HOST
```

Expected: non-zero **Video** engine load (decode) and periodic **Render** activity (OpenVINO inference). In the Frigate UI (`http://192.168.68.102:5000` → System metrics): detector inference speed — UHD 770 is a solid media/decode iGPU; note the actual figure rather than assuming Arc-class numbers, since Quick Sync media decode is excellent but raw compute is more modest than Arc's Xe-LPG. All 4 cameras streaming, low host CPU. Ask Lawrie to eyeball the four live views and confirm zones/masks look right.

### 5. Handoff report

Write `~/rebuild/reports/phase-5.md`: media volume size + retention chosen, camera names mapped old→new, inference speed observed, `intel_gpu_top` observations, anything ported vs. deferred (masks/zones), UI URL, MQTT connectivity confirmed to the Pi 5.

## Verification checklist

- [ ] `renderD128` accessible inside the Frigate container (no permission errors in logs).
- [ ] Frigate reachable at `192.168.68.102:5000`; onboot restart confirmed (`restart: unless-stopped`).
- [ ] All 4 cameras live in Frigate UI; go2rtc one-connection-per-camera confirmed in logs.
- [ ] OpenVINO/GPU detector running on the UHD 770; `intel_gpu_top` shows video+render load; host CPU modest.
- [ ] MQTT: Frigate connects to the **Pi 5** (`.101`), confirmed via logs and `mosquitto_sub -h 192.168.68.101 ... -t 'frigate/available'` showing `online`.
- [ ] `~/rebuild/reports/phase-5.md` written.

## Rollback

`docker compose down` and re-run. The salvaged config is never modified in place — work on copies.

## Exit criteria

Cameras recording again on the HP Mini, GPU-accelerated, publishing to the Pi 5's MQTT broker. Next phase: `phase-6-ha-frigate-integration.md`, back on the NUC session, orchestrating both the Pi 5 (HA) and this box.
