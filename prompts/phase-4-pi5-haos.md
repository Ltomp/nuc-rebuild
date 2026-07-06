# Phase 4 — Raspberry Pi 5: Home Assistant OS (bare-metal)

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180), with root SSH to the NUC's Proxmox host **192.168.68.100**. This phase moves Home Assistant off the NUC entirely, onto a **Raspberry Pi 5 running HAOS bare-metal** — no Proxmox, no VM, direct on the hardware. The Zigbee coordinator is network-attached (`tcp://192.168.68.98:6638`), so this move has zero impact on the Zigbee mesh. You'll drive the Pi 5 over SSH once Lawrie has it flashed and booted; the physical steps are his.

## Why bare-metal, not a VM

This is a genuine, first-class, extremely common Home Assistant deployment pattern — HAOS ships an official Raspberry Pi 5 image. No virtualization tax, no passthrough complexity, and it physically decouples HA/Zigbee/voice from whatever's happening to the NUC or HP Mini during their rebuild work.

## Context — what this HA instance actually is

A live audit before this rebuild found: **144 devices, 1,752 entities, 58 integrations, 28 automations**, a **51-device Zigbee mesh** (17 end devices + 34 routers + 8 groups) via the networked SLZB coordinator, plus HACS, Matter Server, ESPHome (2 devices), local voice assist (Whisper + Piper + assist_microphone), Samba, Tailscale, and a code-server (VS Code) add-on. The recorder database was **419MB and growing**, with no `recorder:` tuning in `configuration.yaml`. This is a real, heavily-loaded home automation hub — treat the restore accordingly.

**The Pi 5 has a fixed 8GB RAM ceiling** — unlike a VM on a 32GB host, there's no borrowing more if it runs tight. Recorder tuning (step 6) is not optional polish here; it's load-bearing.

## Preconditions

- Read `~/rebuild/reports/phase-1.md` — confirms the **Full** HA backup (not partial) is salvaged at `~/rebuild/salvage/<file>.tar`, ideally several hundred MB+ given the recorder size.
- Verify it's still readable: `tar -tf ~/rebuild/salvage/<ha-backup>.tar > /dev/null && echo OK`.
- Confirm Phase 2 already destroyed the old HAOS VM on the NUC (read `phase-2.md`) — HA should currently be offline.

## Steps

### 1. Physical setup — Lawrie's steps

Ask Lawrie to:
1. Upgrade the Pi 5 from microSD to NVMe (via an NVMe HAT/M.2 board) if not already done — strongly recommended given the recorder's write volume; microSD wears out under this load.
2. Download the official **Home Assistant OS Raspberry Pi 5** image from the [HAOS releases page](https://github.com/home-assistant/operating-system/releases) (the `haos_rpi5-64-<version>.img.xz` asset) and flash it to the NVMe using `balenaEtcher`, the Raspberry Pi Imager, or `dd` from another machine.
3. Insert the NVMe, connect Ethernet (not WiFi — this is a home-automation hub, wired is non-negotiable for reliability), power it on.
4. Confirm when it's booted — wait a few minutes for first boot.

### 2. DHCP reservation — needs Lawrie

Once booted, find its MAC address (Lawrie can check the router's DHCP client list, or you can try `nmap -sn 192.168.68.0/24` from this VM and look for a new host). Ask Lawrie to set a DHCP reservation for that MAC to **192.168.68.101** (reused from the old HAOS VM). Wait for confirmation, then reboot the Pi if it already grabbed a different address (`ssh` in isn't possible yet at this stage — this is done via the HAOS onboarding web UI, not SSH; HAOS doesn't expose OS-level SSH by default).

### 3. Verify it's reachable and onboard

```bash
curl -s -o /dev/null -w "%{http_code}" http://192.168.68.101:8123
```

Expected: `200` (onboarding page). This is a **fresh HAOS instance** — do not click through onboarding yet; the restore-from-backup option appears during onboarding, which is the path you want.

### 4. Restore the Phase 1 full backup

1. Ask Lawrie to open `http://192.168.68.101:8123`, and on the onboarding screen choose **restore from backup**, uploading the salvaged tar. Tell them its exact path; if they're on another machine, serve it briefly from here:

```bash
cd ~/rebuild/salvage && python3 -m http.server 8000
# give Lawrie: http://192.168.68.180:8000/<file>.tar — stop the server (Ctrl-C) once downloaded
```

2. Given the backup's size, this restore will take a while — poll `curl -s -o /dev/null -w "%{http_code}" http://192.168.68.101:8123` until it returns to `200` with the real login page (not onboarding) rather than assuming a fixed wait time.
3. Lawrie logs in with the old credentials. Confirm together: automations (should be 28), Zigbee2MQTT devices (should be ~51, reconnecting to the coordinator automatically — no re-pairing needed since it's the same network coordinator), Matter Server, ESPHome (2 devices), HACS, and the voice assist add-ons are all present. Anything missing → check it was a Full backup (Phase 1 step 1 note), not partial.
4. **Tailscale**: may not reconnect automatically even with a full restore, since node identity can tie to the physical install. Ask Lawrie to check the Tailscale add-on's status and re-authenticate if needed (a fresh auth link, same as a new node join).

### 5. Enable SSH for future phases

The Advanced SSH & Web Terminal add-on (used for the earlier audit) should come back from the restore. Verify: `ssh <user>@192.168.68.101 'echo ok'` (password auth, per its config — or set up key auth now for consistency with the rest of this rebuild, at your discretion).

### 6. Recorder tuning (not optional — fixed 8GB ceiling)

The current install has no `recorder:` block, meaning **every one of 1,752 entities'** history is kept at the 10-day default — this produced a 419MB, still-growing database with no bound. Add one to `configuration.yaml`, tuned rather than guessed — ask Lawrie which domains/entities are noise (typically: per-device signal-strength/LQI sensors, battery-percentage sensors already tracked by `battery_notes`, and any high-frequency numeric sensors not needed historically). A reasonable starting point:

```yaml
recorder:
  purge_keep_days: 7
  exclude:
    domains:
      - update
    entity_globs:
      - "sensor.*_linkquality"
      - "sensor.*_battery"
```

Restart HA, confirm no errors in the log, and monitor `du -sh /homeassistant/home-assistant_v2.db` over the next few days (note in the report to check back).

### 7. Drop code-server (VS Code add-on) — recommended

Now that a full workstation VM exists on the NUC for real dev work (Phase 8), the code-server add-on on an always-on home-automation appliance is unneeded surface area. Confirm with Lawrie, then remove it from Settings → Add-ons if agreed.

### 8. Handoff report

Write `~/rebuild/reports/phase-4.md`: Pi 5 storage used (NVMe confirmed), HAOS version, MAC + reservation confirmed, restore result (what came back, what needed manual fixing — especially Tailscale), recorder config applied, code-server decision, current `free -h` / `df -h` on the Pi 5 to establish a baseline.

## Verification checklist

- [ ] Pi 5 boots from NVMe (not microSD), reachable at `192.168.68.101`.
- [ ] Full restore confirmed by Lawrie: 28 automations, ~51 Zigbee devices reconnected, Matter/ESPHome/HACS present.
- [ ] Tailscale re-authenticated if needed.
- [ ] `recorder:` block added and HA restarts cleanly.
- [ ] SSH access confirmed for future phases.
- [ ] `~/rebuild/reports/phase-4.md` written.

## Rollback

The salvaged backup tar is read-only — re-flash the Pi 5 and restore again from scratch if something goes wrong early on. Once Lawrie has done meaningful post-restore configuration (recorder tuning, Tailscale re-auth), treat that as new state worth its own backup before further changes.

## Exit criteria

Home Assistant fully restored and running on the Pi 5 at `.101`, Zigbee/Matter/ESPHome/voice all confirmed working, recorder tuned for the fixed 8GB ceiling. Next phase: `phase-5-hpmini-bootstrap.md` — **different machine again: the HP Pro Mini 400 G9**, driven from here once Lawrie has Proxmox installed on it.
