# Phase 3 — NUC Proxmox host upgrade PVE 8 → 9 + iGPU optimisation

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with root SSH to the Proxmox host **192.168.68.100**. The host runs **PVE 8.4.x** (Debian 12 base). This phase upgrades it in place to **PVE 9.x** (Debian 13 base, 6.14+ kernel — materially better Meteor Lake / Arc iGPU support), reboots it once, then installs iGPU tooling and verifies the GPU. The only other guest was already destroyed in Phase 2, so this is the safest possible window. **This matters beyond Frigate now**: the LLM LXC built in Phase 8 runs on this same Arc iGPU, and the modern **Xe driver** (vs. legacy i915) that comes with the 6.14+ kernel has meaningfully better Arc compute support — worth getting right here.

**This phase is resumable.** The host reboot kills this session (the Coding VM restarts with it, also applying its Phase 2 shrink to 6GB/4 vCPU). Lawrie restarts a session and pastes this same prompt; step 0 routes you to the right place.

## Preconditions / Step 0 — figure out where you are

- Read `~/rebuild/reports/phase-0.md`, `phase-1.md`, `phase-2.md`. All must exist.
- Check host version: `ssh root@192.168.68.100 'pveversion'`
  - `pve-manager/8.x` → fresh run, start at step 1.
  - `pve-manager/9.x` → the reboot already happened; skip to step 5.
- If `~/rebuild/reports/phase-3-progress.md` exists, read it and resume from the recorded step. **Append to this progress file after every completed step** — it is what makes resumption safe.

## Steps

### 1. Pre-upgrade checks

```bash
ssh root@192.168.68.100 'pve8to9 --full'
```

Read every WARN/FAIL. Typical items and required actions:
- Repos still pointing at enterprise without subscription → will be fixed in step 2.
- Old kernels or low `/boot` space → `apt autoremove --purge` old kernels.
- Anything you don't recognise: **fetch the official guide** (https://pve.proxmox.com/wiki/Upgrade_from_8_to_9) and follow it; do not improvise. FAILs must be resolved before continuing; re-run `pve8to9 --full` until no FAILs remain (WARNs: understand each, fix or consciously accept, and note them in the progress file).

Also bring PVE 8 fully current first:

```bash
ssh root@192.168.68.100 'apt update && apt -y dist-upgrade'
```

### 2. Switch repositories to PVE 9 / Debian trixie (no-subscription)

```bash
ssh root@192.168.68.100 '
  sed -i "s/bookworm/trixie/g" /etc/apt/sources.list
  for f in /etc/apt/sources.list.d/*.list; do [ -f "$f" ] && sed -i "s/bookworm/trixie/g" "$f"; done
  # ensure no enterprise repo is active without a subscription
  grep -rn "enterprise.proxmox.com" /etc/apt/ && echo "DISABLE THESE" || true
  cat > /etc/apt/sources.list.d/pve-no-subscription.list <<EOF
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
EOF
'
```

Disable (comment out) any `pve-enterprise` entries found. Note: PVE 9 may use `.sources` (deb822) files — if the official guide's current instructions differ, follow the guide.

### 3. The upgrade

Run it detached on the host so a dropped SSH connection can't kill the upgrade mid-flight:

```bash
ssh root@192.168.68.100 'apt update && screen -dmS upgrade bash -c "DEBIAN_FRONTEND=noninteractive apt -y dist-upgrade 2>&1 | tee /root/pve9-upgrade.log"'
```

(If `screen` is absent, `apt install -y screen` first, or use `tmux`/`nohup`.) Poll progress with `ssh root@192.168.68.100 'tail -5 /root/pve9-upgrade.log; screen -ls'` every few minutes until the screen session ends. Then confirm: `pveversion` shows 9.x packages installed (still on the old kernel until reboot) and the log has no fatal errors.

### 4. Host reboot — GATED, drops this session

Update `~/rebuild/reports/phase-3-progress.md` FIRST: "upgrade complete, rebooting host; on resume start at step 5." Then ask Lawrie:

> Host upgrade to PVE 9 is installed; a host reboot is required. **This session will drop** (the Coding VM restarts too, coming back with its new 6GB/4 vCPU sizing). After the host is back, restart a Claude session and paste this same Phase 3 prompt — it will resume automatically. Reboot now? (yes/no)

On "yes": `ssh root@192.168.68.100 'reboot'`. The session ends here.

### 5. Post-reboot verification + iGPU optimisation (resumed session starts here)

```bash
ssh root@192.168.68.100 'pveversion && uname -r && uptime'
```

Expected: `pve-manager/9.x`, kernel ≥ 6.14. Confirm the Coding VM shrink applied — locally run `free -h` (≈6GB total) and `nproc` (4).

Install tooling and verify the Arc iGPU:

```bash
ssh root@192.168.68.100 '
  apt update && apt install -y intel-microcode intel-gpu-tools vainfo
  ls -l /dev/dri/            # expect card0/card1 + renderD128
  lsmod | grep -E "^i915|^xe" # note which driver claimed the iGPU
  vainfo 2>&1 | head -25      # expect iHD driver, a list of H264/HEVC decode profiles, no errors
  cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor && cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
'
```

- `vainfo` must list decode entrypoints (VLD) for at least H264 and HEVC. If it errors, install `intel-media-va-driver-non-free` and retest.
- Governor: `powersave` with driver `intel_pstate` is **correct** for this always-on Meteor Lake box (hardware-managed P-states) — do not "fix" it to performance.
- Record which of `i915`/`xe` is driving the GPU in the report; Phase 8's LLM LXC needs `renderD128` to exist and the **Xe** driver ideally in use (better Arc compute path than legacy i915) — if `i915` claimed it instead, note that in the report so Phase 8 can investigate.

Finally: `apt -y autoremove --purge` on the host to clear leftover Debian 12 packages.

### 6. Handoff report

Write `~/rebuild/reports/phase-3.md`: PVE version before/after, kernel, pve8to9 items resolved, driver in use (i915/xe), vainfo summary, governor confirmation, Coding VM new sizing confirmed, path of `/root/pve9-upgrade.log`. Delete `phase-3-progress.md` (superseded).

## Verification checklist

- [ ] `pveversion` = 9.x, kernel ≥ 6.14, host stable after reboot.
- [ ] `pve8to9` (rerun post-upgrade if available) shows no FAILs.
- [ ] `/dev/dri/renderD128` present; `vainfo` clean with H264+HEVC decode.
- [ ] Coding VM now 6GB / 4 vCPU.
- [ ] `~/rebuild/reports/phase-3.md` written.

## Rollback / if something fails

There is no clean downgrade from PVE 9 → 8. If the upgrade fails mid-way: **stop, do not reboot**, capture `/root/pve9-upgrade.log`, and work the error against the official upgrade wiki (usually a repo or held-package issue fixed by `apt -f install` / correcting sources and re-running `dist-upgrade`). The vzdump archives from Phase 1 protect the *guests*, not the host — treat host errors carefully and involve Lawrie before any drastic action.

## Exit criteria

Host on PVE 9, iGPU verified, report written. Next phase: `phase-4-pi5-haos.md` — **different machine: the Raspberry Pi 5**, physically flashed and driven from here over SSH.
