# Phase 2 — NUC: wipe Proxmox, install bare-metal Debian 13

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with root SSH to the Proxmox host **192.168.68.100**. This phase permanently replaces the entire Proxmox install on the NUC with plain Debian 13 — no hypervisor, no VMs, no LXCs. **This is the point of no return for this host and for this session.** Everything past the pre-flight check in this file is either a instruction for Lawrie to carry out physically, or a instruction for the *next* Claude Code session (running on the new install) to pick up.

## Why this phase looks different from every phase before it

Phases 0-1 only read and copied. This phase erases the disk this very session runs on. There is no `qm destroy <single-guest>` gate here — the whole host goes at once, Coding VM included. Treat the pre-flight check below as an absolute gate, not a formality: once Lawrie boots the installer USB, anything not already copied off this host is gone for good.

## Preconditions — hard gate, do not skip

Read `~/rebuild/reports/phase-1.md` in full. It must show, all confirmed:
- [ ] HA full backup tar salvaged and verified (`tar -tf` clean).
- [ ] `frigate-config.yml` salvaged, contains 4 cameras.
- [ ] The Coding VM's own archive (`coding-vm-home.tar.gz`) built, copied to the **off-NUC** destination Lawrie chose (laptop or USB), and verified there with `TAR-OK`.
- [ ] `~/nuc-rebuild` git repo fully pushed to GitHub (`git status` clean against `origin/main`).

If any box isn't checked, or you can't independently re-verify it (re-run the verification commands from Phase 1 yourself — don't just trust the report text), **stop and tell Lawrie exactly what's missing.** Do not proceed to the physical steps.

Once you're satisfied, say so explicitly and confirm with Lawrie:

> Salvage is verified complete and off this host. The next step wipes the NUC's disk and this session will end permanently. Ready to proceed whenever you are — this doesn't have to happen right now.

## Steps

### 1. Physical wipe + install — Lawrie's steps, whenever convenient

Give Lawrie this checklist:

1. Download the Debian 13 netinst ISO (`https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/`) and write it to a USB installer (Rufus, `balenaEtcher`, or `dd` from another machine).
2. Boot the NUC from the USB installer (F10/F2 boot menu depending on BIOS — ASUS NUCs typically use F10 or Esc).
3. Run through the installer:
   - Language/locale/keyboard as preferred.
   - Hostname: `nuc`. Domain: leave blank.
   - Network: configure a **static IP, `192.168.68.103`**, matching gateway/DNS/netmask for the `192.168.68.0/22` LAN (same as the rest of this rebuild).
   - User: `lawrie`, with a real password; allow root login or set up sudo for this user (either is fine — sudo is more conventional, note which was chosen).
   - Partitioning: guided, use entire disk (the 2TB NVMe), single ext4 partition — no LVM complexity needed for a single-workload box.
   - Software selection: **deselect every desktop environment**; select only **SSH server** and **standard system utilities**.
4. Let it install and reboot. Confirm it comes up at `192.168.68.103`.

### 2. Post-install hardening — Lawrie does this at the console, or you drive it once SSH is up

```bash
ssh lawrie@192.168.68.103 'echo ok'   # first connection, password auth
```

Once reachable, from this (soon-to-be-gone) Coding VM, push the same SSH key already used for the NUC/HP Mini Proxmox pattern so the *next* session doesn't have to bootstrap key auth from scratch:

```bash
ssh-copy-id lawrie@192.168.68.103
ssh -o BatchMode=yes lawrie@192.168.68.103 'echo KEY-OK'
```

Basic hygiene:

```bash
ssh lawrie@192.168.68.103 'sudo apt update && sudo apt -y dist-upgrade && sudo apt install -y git curl sudo'
```

(If root login was chosen over sudo during install, adjust these commands accordingly — note which in the report.)

### 3. Bootstrap Claude Code on the new install

```bash
ssh lawrie@192.168.68.103 'curl -fsSL https://claude.ai/install.sh | bash'
ssh lawrie@192.168.68.103 '~/.local/bin/claude --version || claude --version'
```

### 4. Pull the salvage archive back down — this is what makes the new session a continuation, not a fresh start

Ask Lawrie to make the archive from Phase 1 (on the laptop or USB) reachable, then pull it onto the new NUC install:

```bash
# example if the laptop route was used — adjust source per what Phase 1's report recorded
scp lawrie@<laptop-ip>:~/nuc-rebuild-salvage/coding-vm-home.tar.gz lawrie@192.168.68.103:~/
ssh lawrie@192.168.68.103 'tar xzf ~/coding-vm-home.tar.gz -C ~ && rm ~/coding-vm-home.tar.gz'
ssh lawrie@192.168.68.103 'ls -la ~/.claude ~/claude ~/rebuild ~/.ssh 2>&1 | head -30'
```

Fix permissions on the restored keys:

```bash
ssh lawrie@192.168.68.103 'chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_* 2>/dev/null; chmod 644 ~/.ssh/*.pub 2>/dev/null; true'
```

Verify the restored `claude` CLI sees its old identity:

```bash
ssh lawrie@192.168.68.103 'claude --version && ls ~/rebuild/reports/'
```

Expected: `phase-0.md` and `phase-1.md` present. (Phases 3 onward don't exist yet — they haven't happened.) If `claude` prompts for login despite the restored config, that's fine — have Lawrie complete it once; the account/session should otherwise carry over from the restored `~/.claude`/`~/.claude.json`.

### 5. Clone/verify the plan repo on the new install

```bash
ssh lawrie@192.168.68.103 'cd ~ && git clone https://github.com/Ltomp/nuc-rebuild.git nuc-rebuild 2>&1 || (cd nuc-rebuild && git pull)'
```

(This is a fresh clone using the credentials restored in step 4's archive — verify `git log -1` shows the latest commit.)

### 6. Handoff report — written from wherever this session still has write access

If this Coding VM is still alive when you reach this point, write `~/rebuild/reports/phase-2.md` here **and** copy it into the restored `~/rebuild/reports/` on the new NUC install (it won't exist there yet since Phase 1's report predates this). Record: exact install choices (root vs sudo), the new machine's SSH host key fingerprint (for future reference), confirmation of steps 1-5, and this line verbatim: **"This Coding VM (192.168.68.180) is retired as of this report. All future phases run from 192.168.68.103."**

```bash
scp ~/rebuild/reports/phase-2.md lawrie@192.168.68.103:~/rebuild/reports/
```

## Verification checklist

- [ ] NUC reinstalled: Debian 13, bare-metal, no Proxmox, static IP `.103`.
- [ ] SSH key auth working from the salvage archive (or freshly copied in step 2).
- [ ] `claude --version` works on the new install; restored `~/.claude` config confirmed.
- [ ] `~/rebuild/` restored with phase-0 and phase-1 reports present.
- [ ] `nuc-rebuild` repo cloned/pulled successfully on the new install.
- [ ] `~/rebuild/reports/phase-2.md` written and present on the new install.

## Rollback

None past step 1 — this is a destructive OS reinstall by design. Everything needed to continue exists because Phase 1's salvage was verified *before* this phase began. If step 1 needs to be redone (bad install, wrong disk, etc.), simply repeat it; nothing outside the NUC's own disk was at risk.

## Exit criteria

The NUC is running bare-metal Debian 13 at `.103` with Claude Code restored and working, and this is the last report written by/about the old Coding VM. **From here on, start every subsequent phase's session on `192.168.68.103` itself** — there is no more Proxmox host to SSH into, no more Coding VM. Next phase: `phase-3-hpmini-bare-metal-install.md`, run from this new NUC session.
