# Phase 7 — HA ↔ Frigate integration

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180). Home Assistant runs at `http://192.168.68.101:8123` on the **Raspberry Pi 5** (Phase 4, restored from backup) and Frigate at `http://192.168.68.102:5000` on the **HP Mini** (Phase 6, GPU-accelerated on the UHD 770, publishing to HA's Mosquitto). This phase connects them: HACS → Frigate integration → camera dashboard → one worked notification automation. Most steps happen in the HA UI, so this phase is **you guiding Lawrie click-by-click and verifying results over MQTT/API**, not running many commands yourself.

## Preconditions

- Read `~/rebuild/reports/phase-4.md` (Pi 5 / HA) and `phase-6.md` (HP Mini / Frigate).
- Verify Frigate is publishing: `mosquitto_sub -h 192.168.68.101 -u frigate -P '<frigate MQTT password>' -t 'frigate/available' -C 1 -W 10` prints `online`. (MQTT broker is on the Pi 5 at `.101` — the old NUC-hosted broker no longer exists.)

## Steps

### 1. HACS

Check first — the Phase 4 restore may have brought HACS back (ask Lawrie to look for "HACS" in the sidebar / Settings → Integrations). If present, skip to step 2.

If absent, guide Lawrie through the official install (https://hacs.xyz/docs/use/download/download/): the usual route is the **Get HACS** add-on or the terminal one-liner via the Advanced SSH & Web Terminal add-on, then restart HA, then Settings → Devices & services → Add integration → HACS, and authorise with GitHub. Follow the current official docs if they differ.

### 2. Frigate integration

1. HACS → search "Frigate" → Download; restart HA when prompted.
2. Settings → Devices & services → Add integration → **Frigate**; URL: `http://192.168.68.102:5000`.
3. Expected result: one device per camera plus per-camera entities — `camera.<name>`, occupancy sensors (`binary_sensor.<name>_person_occupancy` etc.), count sensors, and motion sensors. If camera names were kept from the old config (per the Phase 6 report), previously-restored dashboards/automations referencing them may just start working again — check.

Optionally also suggest the **Frigate Card** (HACS → Frontend → "Frigate card") — the best dashboard card for this setup.

### 3. Camera dashboard

Guide Lawrie to create a dashboard (or repair the restored one): a view with the 4 camera feeds (Frigate card in `live` view or standard picture-glance cards), plus each camera's person-occupancy sensor. Keep it simple; polish is Lawrie's call.

### 4. Worked example automation: person → phone notification with snapshot

Confirm the HA companion app is installed on Lawrie's phone (`notify.mobile_app_*` service exists — check in Developer tools → Actions). Then create this automation (Settings → Automations → new → edit in YAML), one trigger entity per camera:

```yaml
alias: "Person detected — notify with snapshot"
mode: parallel
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.cam1_person_occupancy
      - binary_sensor.cam2_person_occupancy
      - binary_sensor.cam3_person_occupancy
      - binary_sensor.cam4_person_occupancy
    to: "on"
actions:
  - action: notify.mobile_app_<lawries_phone>
    data:
      message: "Person detected: {{ trigger.to_state.attributes.friendly_name }}"
      data:
        image: >-
          http://192.168.68.102:5000/api/{{ trigger.to_state.entity_id
          | replace('binary_sensor.','') | replace('_person_occupancy','') }}/latest.jpg
        clickAction: "/lovelace/cameras"
```

Adjust entity ids to the real ones from step 2. (This is the simple robust version; fancier event-based notifications via the Frigate notification blueprint can come later.)

### 5. Live test

Ask Lawrie to walk in front of one camera. Verify the full chain:
1. Frigate UI shows the person event.
2. `mosquitto_sub ... -t 'frigate/events' -C 1 -W 60` shows the event JSON.
3. The occupancy sensor flips `on` in HA (Developer tools → States).
4. Lawrie's phone gets the notification **with the snapshot image**.

If a link in the chain fails, debug that link only (integration config, sensor name, notify service) — the earlier phases proved the rest.

### 6. Handoff report

Write `~/rebuild/reports/phase-7.md`: HACS status (restored vs fresh), integration entity naming, dashboard state, the automation's final YAML, and the live-test result.

## Verification checklist

- [ ] Frigate integration loaded; entities exist for all 4 cameras.
- [ ] Dashboard shows all 4 live feeds.
- [ ] Walk test: Frigate event → MQTT → HA sensor → phone notification with snapshot, end to end.
- [ ] `~/rebuild/reports/phase-7.md` written.

## Rollback

Everything here is additive inside HA (integration, dashboard, automation) — remove via the HA UI if needed. Nothing outside HA changes.

## Exit criteria

The security stack is fully rebuilt and verified across two machines (Pi 5 + HP Mini). Next phase: `phase-8-nuc-workstation-llm.md`, back to the NUC.
