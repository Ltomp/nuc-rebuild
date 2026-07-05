# Phase 7 — New workstation VM (permanent Claude home)

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with root SSH to the Proxmox host **192.168.68.100**. This phase builds the permanent **workstation VM**: Debian 13 + XFCE + xrdp + Chrome + **Claude Desktop for Linux (beta)** + Claude Code CLI, then migrates all of Lawrie's Claude data off this Coding VM. The Coding VM keeps running (it gets destroyed in Phase 8, not now).

## Target

| Item | Value |
|---|---|
| VMID | **112** (confirm free; else next free, record it) |
| Name | `workstation` |
| Resources | **8GB RAM** / 6 vCPU / 150GB disk — 8GB is deliberate: with the Coding VM (6GB) + HAOS (4GB) + Frigate (4GB) still running, 8GB keeps total guest allocation at 22GB on the 32GB host. **Phase 8 raises it to 16GB after the Coding VM is destroyed. Do not create it with 16GB.** |
| IP | `192.168.68.103` via DHCP reservation (**not** .180 — the Coding VM still owns that) |
| OS | Debian 13 (trixie) netinst, CPU type `host`, virtio disk/net, qemu-guest-agent |

## Preconditions

- Read `~/rebuild/reports/phase-3.md` (storage names) and confirm phases 4–6 reports exist.
- Free space check: `ssh root@192.168.68.100 'pvesm status'` — 150GB must fit comfortably.

## Steps

### 1. Create the VM and install Debian 13

```bash
ssh root@192.168.68.100 '
  cd /var/lib/vz/template/iso && wget -q https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.*-amd64-netinst.iso || ls debian-13*.iso
  qm create 112 --name workstation --memory 8192 --cores 6 --cpu host \
    --machine q35 --bios ovmf --efidisk0 <STORE>:1,efitype=4m \
    --scsihw virtio-scsi-single --scsi0 <STORE>:150,discard=on,ssd=1 \
    --net0 virtio,bridge=vmbr0 --ostype l26 --agent enabled=1 \
    --ide2 local:iso/<the-iso>,media=cdrom --boot order="scsi0;ide2" \
    --vga std --onboot 1 --startup order=3,up=30
  qm start 112
'
```

The Debian installer is interactive — ask Lawrie to open the VM's **Console** in the Proxmox web UI and walk them through it: language/locale/keyboard; hostname `workstation`; user `lawrie` (+ root password or sudo-via-first-user); guided partitioning, whole disk, ext4, no LVM complications needed; **software selection: deselect every desktop environment, select only "SSH server" and "standard system utilities"** (XFCE is installed minimal in the next step). After install completes and the VM reboots, detach the ISO: `ssh root@192.168.68.100 'qm set 112 --ide2 none --boot order=scsi0'`.

### 2. DHCP reservation — needs Lawrie

`ssh root@192.168.68.100 'qm config 112 | grep net0'` → MAC to Lawrie → reservation for **192.168.68.103** → reboot the VM (`qm reboot 112`) → confirm `ping -c1 192.168.68.103` and `ssh lawrie@192.168.68.103` works (copy this VM's SSH key over first: `ssh-copy-id lawrie@192.168.68.103`, password auth once).

All remaining steps run over SSH from here: `ssh lawrie@192.168.68.103` (use `sudo` as needed; if sudo isn't set up, `su -c 'apt install -y sudo && usermod -aG sudo lawrie'` via root once, then re-login).

### 3. Guest agent, XFCE (minimal), xrdp

```bash
sudo apt update && sudo apt install -y qemu-guest-agent
sudo apt install -y --no-install-recommends xfce4 xfce4-terminal xfce4-goodies dbus-x11
sudo apt install -y xrdp
echo xfce4-session > ~/.xsession
sudo adduser xrdp ssl-cert
sudo systemctl enable --now xrdp
```

Verify: `ss -tlnp | grep 3389` listening. Ask Lawrie to RDP to `192.168.68.103` from their machine (any RDP client), log in as `lawrie`, and confirm an XFCE desktop appears.

### 4. Chrome

```bash
sudo curl -fsSLo /usr/share/keyrings/google-chrome.asc https://dl.google.com/linux/linux_signing_key.pub
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome.asc] https://dl.google.com/linux/chrome/deb/ stable main" | sudo tee /etc/apt/sources.list.d/google-chrome.list
sudo apt update && sudo apt install -y google-chrome-stable
```

### 5. Claude Desktop (Linux beta) — the primary Claude app

Debian 13 x86_64 meets the beta requirements (Debian 12+). Install from Anthropic's apt repo so updates come with `apt upgrade`:

```bash
sudo curl -fsSLo /usr/share/keyrings/claude-desktop-archive-keyring.asc https://downloads.claude.ai/claude-desktop/key.asc
gpg --show-keys /usr/share/keyrings/claude-desktop-archive-keyring.asc   # fingerprint must be 31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/claude-desktop-archive-keyring.asc] https://downloads.claude.ai/claude-desktop/apt/stable stable main" | sudo tee /etc/apt/sources.list.d/claude-desktop.list
sudo apt update && sudo apt install -y claude-desktop
```

**Verify the fingerprint matches before installing.** Known Linux-beta limits to tell Lawrie: no Computer Use (screen control), no in-app dictation; Chat, Cowork, and Code tabs all work, including parallel sessions, diff review, integrated terminal/editor.

### 6. Node.js + Claude Code CLI (kept alongside Desktop — covers headless/SSH use)

```bash
sudo apt install -y nodejs npm git   # Debian 13 ships Node ≥ 20; verify: node --version
curl -fsSL https://claude.ai/install.sh | bash   # native installer; ensure ~/.local/bin on PATH
claude --version
```

If the Coding VM instead used an npm-global install (`npm ls -g` here will tell you), mirror whichever setup the Coding VM has, so migrated config behaves identically.

### 7. Migrate Lawrie's data from the Coding VM

Run from the **Coding VM** (here), pushing to the workstation. Migrate, then verify each item:

```bash
rsync -aP ~/claude/     lawrie@192.168.68.103:~/claude/
rsync -aP ~/.claude/    lawrie@192.168.68.103:~/.claude/      # memory, settings, plans, projects
rsync -aP ~/.claude.json lawrie@192.168.68.103:~/ 2>/dev/null || true
rsync -aP ~/awa/        lawrie@192.168.68.103:~/awa/
rsync -aP ~/.gitconfig  lawrie@192.168.68.103:~/
rsync -aP ~/.git-credentials lawrie@192.168.68.103:~/ 2>/dev/null || true
rsync -aP ~/.ssh/       lawrie@192.168.68.103:~/.ssh/          # includes the host-authorized key from Phase 0
rsync -aP ~/rebuild/    lawrie@192.168.68.103:~/rebuild/       # salvage + phase reports — Phase 8 runs from there
```

Then sweep for stragglers and migrate anything relevant you find: `npm ls -g --depth=0` (recreate globals on the workstation), MCP server configs and tokens (check `~/.claude.json`, `~/.config/`, project `.mcp.json` files), crontabs (`crontab -l`), shell rc customisations (`~/.bashrc` additions), and anything else in `~` that isn't OS noise (`ls -la ~`). Fix perms on arrival: `ssh lawrie@192.168.68.103 'chmod 700 ~/.ssh && chmod 600 ~/.ssh/*'`.

Verify from the workstation: `ssh lawrie@192.168.68.103 'ssh -o BatchMode=yes root@192.168.68.100 echo host-ok && ls ~/rebuild/reports/'` — the migrated key must reach the Proxmox host and all phase reports must be present.

### 8. Lawrie's sign-in session (RDP)

Ask Lawrie to RDP into `192.168.68.103` and:
1. Launch **Claude Desktop** (app menu or `claude-desktop`), sign in with the claude.ai account (Desktop doesn't take API keys).
2. Sign into **Chrome**, install the **Claude in Chrome** extension, and connect it.
3. Open a terminal, run `claude` in `~/claude/<a-project>`, sign in if prompted, and confirm memory/settings carried over (e.g. `/config` shows migrated settings).

### 9. Handoff report

Write `~/rebuild/reports/phase-7.md` (it syncs to the workstation — rsync `~/rebuild/` again after writing): VMID, MAC, sizing (8GB now, 16GB in Phase 8), everything migrated with byte counts, sweep findings, Desktop/CLI versions, what Lawrie confirmed in the sign-in session, anything intentionally left on the Coding VM.

## Verification checklist

- [ ] VM 112: 8GB/6vCPU/150GB, `.103`, agent responding, onboot order=3.
- [ ] RDP → XFCE works; Chrome + extension working.
- [ ] Claude Desktop installed from the apt repo (key fingerprint verified), signed in, Code tab opens a migrated project.
- [ ] `claude` CLI runs with migrated `~/.claude/` config.
- [ ] Migration sweep done; `~/rebuild/` present on the workstation including this phase's report.
- [ ] Coding VM untouched and still running.

## Rollback

The Coding VM remains fully intact — nothing is deleted in this phase, so rollback is simply `qm stop 112 && qm destroy 112 --purge` and re-run.

## Exit criteria

Workstation fully operational and data-complete; Coding VM now redundant but alive. Next phase: `phase-8-verify-decommission.md` — **run that session from the workstation VM, not from here.**
