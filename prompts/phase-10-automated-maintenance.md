# Phase 10 — Automated maintenance (ongoing, recurring)

You are Claude Code (Sonnet) running on the **workstation VM** (192.168.68.103), after Phase 9 has fully completed. Unlike every phase before it, this one doesn't migrate or build anything one-time — it sets up a **recurring weekly job** that keeps the fleet patched, and hands you (a future, unattended instance of yourself) a safe, pre-scoped way to do it without a human watching every run.

## Design — read this before building anything

There are five machines/guests in play now: the NUC Proxmox host, the HP Mini Proxmox host, the workstation VM, the LLM LXC, and the Frigate LXC — plus the Pi 5, which is **not** a general Debian box you `apt` into; HAOS is a locked-down appliance managed through Home Assistant's own Supervisor API.

Updates split into two tiers by risk, and **only one tier runs unattended**:

**Tier 1 — fully automatic, no LLM needed at all.** OS-level security patches (`unattended-upgrades`) on every Debian-based host/guest: NUC host, HP Mini host, workstation VM, LLM LXC, Frigate LXC. This is a well-established, boring, battle-tested mechanism — don't reinvent it with an agent loop.

**Tier 2 — checked weekly by a scheduled Claude Code run; only HACS updates auto-apply, everything else held for Lawrie's explicit approval.** This covers: HA Core/Supervisor/OS/add-on updates, HACS integration updates, Frigate's version, the LLM serving stack's (IPEX-LLM/Ollama) version, and Proxmox point-releases.
- **Auto-apply**: HACS integration updates only — genuinely low blast radius, easy to roll back via HACS itself.
- **Never auto-apply, always just report + notify**: HA Core/Supervisor/OS updates (breaking-change history on this exact install matters — 1,752 entities, 28 automations riding on it), Proxmox point/major releases (we just did the careful `pve8to9`-checklist upgrade by hand for a reason), **Frigate updates of any size** (it has a real history of config-schema changes — retention config, `go2rtc` integration — shipping inside releases that look like routine version bumps; never assume a patch-looking version number means a safe patch), the LLM stack's version, and **Zigbee coordinator / ESPHome device firmware** (a botched auto-update here can brick hardware requiring a physical re-flash — this category is flag-only, forever, no exceptions).

### Checking software that HA doesn't know about

HA's `update.*` entities only cover HA Core/Supervisor/OS, add-ons, and HACS. **Frigate and the LLM stack are neither** — they're raw software running in an LXC, invisible to HA's update system entirely. The pattern for anything in this situation: compare the *currently running* version against the *latest published* version from the project's own release channel.

For Frigate, both halves are cheap API calls:

```bash
# what's actually running right now
curl -s http://192.168.68.102:5000/api/version

# what's the latest published release
curl -s https://api.github.com/repos/blakeblackshear/frigate/releases/latest | jq -r '.tag_name, .published_at, .body'
```

If they differ, **read the `.body` release-notes text** (that's what the GitHub API call above already fetches) and summarize it in the report — don't try to auto-classify by version number alone, since Frigate's version numbers don't reliably signal blast radius. This is real work worth having an LLM do (reading unstructured prose for "breaking"/"migration required" language), even though the action taken afterward is always "report, never auto-apply."

For the LLM stack (IPEX-LLM/Ollama), the same pattern applies: check its own version output against its upstream GitHub releases page, same reasoning, same "report only" handling.

If a future piece of software doesn't publish to GitHub Releases, the fallback is comparing the local Docker image's digest against the registry's current manifest digest for the same tag (`docker manifest inspect <image>:<tag>` or the `crane`/`skopeo` tools) — cruder (no changelog text, no semver), but still tells you *whether* something changed even when you can't easily tell *what*.

**Why a cron job + headless Claude CLI, not Anthropic's cloud scheduler**: this needs to run *on your LAN*, with the SSH keys and HA token that already live on the workstation, reaching `192.168.68.x` addresses directly. A cloud-scheduled agent wouldn't have that network path. So this phase sets up a plain Linux cron job on the workstation that invokes `claude` in headless/print mode against a maintenance prompt — the same mechanism you'd use to script any CLI tool, just running the actual Claude Code binary unattended.

**Why the unattended run can't ask permission mid-flight**: nobody's watching a cron job. So the *safe* actions (Tier 1 via unattended-upgrades, Tier 2's auto-apply subset) must be pre-scoped into an allowlist ahead of time — not approved live — and everything else must be "check, describe, notify, take no action" rather than "ask and wait."

## Preconditions

- Read `~/rebuild/reports/phase-9.md` — confirms the rebuild is complete and stable.
- This phase runs entirely from the workstation VM.

## Steps

### 1. Tier 1 — unattended-upgrades on every Debian host/guest

```bash
for host in "root@192.168.68.100" "root@192.168.68.110"; do
  ssh $host 'apt install -y unattended-upgrades && dpkg-reconfigure -f noninteractive unattended-upgrades'
done
ssh root@192.168.68.100 "pct exec 113 -- bash -c 'apt install -y unattended-upgrades && dpkg-reconfigure -f noninteractive unattended-upgrades'"
ssh root@192.168.68.110 "pct exec 111 -- bash -c 'apt install -y unattended-upgrades && dpkg-reconfigure -f noninteractive unattended-upgrades'"
sudo apt install -y unattended-upgrades && sudo dpkg-reconfigure -f noninteractive unattended-upgrades   # the workstation itself
```

Confirm each `/etc/apt/apt.conf.d/50unattended-upgrades` only enables the `-security` and `-updates` origins (not arbitrary version jumps), and `/etc/apt/apt.conf.d/20auto-upgrades` has both `Update-Package-Lists` and `Unattended-Upgrade` set to `"1"`.

### 2. Home Assistant long-lived access token

Ask Lawrie to create one: HA UI → profile (bottom left) → **Long-Lived Access Tokens** → Create Token. Store it on the workstation, not in this repo:

```bash
mkdir -p ~/.config && echo "<token>" > ~/.config/ha-maintenance-token && chmod 600 ~/.config/ha-maintenance-token
```

Verify it works:

```bash
curl -s -H "Authorization: Bearer $(cat ~/.config/ha-maintenance-token)" http://192.168.68.101:8123/api/ | head
```

### 3. Scope a permissions allowlist for the unattended run

Use the `update-config` skill to build a `.claude/settings.json` allowlist scoped to exactly what the weekly run needs and nothing more: read-only `apt list --upgradable` checks everywhere, read-only `curl` GETs to HA's REST API and to `api.github.com` (release checks), the `update.install` service call (HACS only), and read-only `curl` to Frigate's and the LLM stack's own version endpoints. Do **not** allowlist `docker compose pull`/`up`, `qm`/`pct destroy`, `reboot`, or anything else destructive — Frigate and the LLM stack are report-only in this design, so the job never needs write access to them at all.

### 4. Write the maintenance prompt

Create `prompts/maintenance-weekly.md` in this repo — this is the prompt the cron job feeds to `claude` headlessly each week. It should instruct the (future, unattended) Claude session to:

1. Check `apt list --upgradable` on the NUC host, HP Mini host, workstation, LLM LXC, and Frigate LXC — Tier 1 already handles security patches automatically, so this is really just a sanity check that `unattended-upgrades` is actually running (check `/var/log/unattended-upgrades/` timestamps) rather than a thing to act on directly.
2. Query HA's `update.*` entities via the REST API (`GET /api/states` filtered to the `update.` domain) for Core, Supervisor, OS, each add-on, and each HACS integration. For each one showing an update available, fetch the release notes URL (the entity's `release_url` attribute) and summarize whether it looks like a breaking/major change or a routine patch.
3. **Frigate** (not in HA's update system): `curl http://192.168.68.102:5000/api/version` for the running version, `curl https://api.github.com/repos/blakeblackshear/frigate/releases/latest` for the latest published one. If they differ, read the release's `.body` text and summarize what changed — never classify by version number alone.
4. **LLM stack** (also not in HA's update system): same pattern — check the running IPEX-LLM/Ollama version against its upstream GitHub releases.
5. For HACS integration updates identified as routine: apply them (`update.install` service call).
6. For everything else with an update available (HA Core/Supervisor/OS, Proxmox, **Frigate of any version size**, the LLM stack, any Zigbee/ESPHome firmware notice surfaced via HA): take **no action** — just collect it into the report with the summarized release notes.
7. Write `~/maintenance/reports/<date>.md` with what was auto-applied and what's waiting, and send a summary via HA's existing notify service:

```bash
curl -s -X POST -H "Authorization: Bearer $(cat ~/.config/ha-maintenance-token)" \
  -H "Content-Type: application/json" \
  -d '{"message": "Weekly maintenance: <N> auto-applied, <M> waiting for your review — see report.", "title": "Home lab maintenance"}' \
  http://192.168.68.101:8123/api/services/notify/mobile_app_<lawries_phone>
```

### 5. Schedule it

```bash
mkdir -p ~/maintenance/reports
crontab -e
# add:
0 9 * * 1  cd ~/claude/nuc-rebuild && claude -p "$(cat prompts/maintenance-weekly.md)" >> ~/maintenance/cron.log 2>&1
```

(Monday 9am — adjust to taste. Check the current Claude Code CLI docs for the exact non-interactive/print-mode invocation flag if `-p` has changed.)

### 6. Dry-run it

Trigger the cron command manually once, watch it end-to-end, and confirm: Tier 1 status check works, HA API calls succeed, a report gets written, and the phone notification arrives. Don't wait a week to find out the token expired or a path was wrong.

### 7. Handoff report

Write `~/rebuild/reports/phase-10.md`: allowlist scope applied, HA token created (not the token itself), cron schedule, dry-run result, and a reminder of what's permanently excluded from auto-apply (Zigbee/ESPHome firmware, Proxmox major versions, HA Core major versions, Frigate major versions).

## Verification checklist

- [ ] `unattended-upgrades` active and logging on all five Debian hosts/guests.
- [ ] HA long-lived token stored (mode 600) and confirmed working against the REST API.
- [ ] Permissions allowlist scoped narrowly — no destructive commands included.
- [ ] `prompts/maintenance-weekly.md` written and dry-run successfully end-to-end.
- [ ] Cron job installed; phone notification confirmed received.
- [ ] `~/rebuild/reports/phase-10.md` written.

## Rollback

Remove the crontab entry to stop the recurring job at any time — nothing it does is hard to undo, since by design its unattended actions are limited to patch-level bumps on integrations/containers. `unattended-upgrades` can be disabled per-host via `dpkg-reconfigure unattended-upgrades` if ever needed.

## Exit criteria

The fleet patches its own OS-level security updates automatically, and gets a weekly triage pass that safely absorbs routine integration/container bumps while surfacing anything riskier for Lawrie's explicit call — without ever touching Proxmox major versions, HA Core majors, or device firmware on its own.
