# Phase 8 — NUC: workstation VM + LLM LXC

You are Claude Code (Sonnet) on the **Coding VM** (192.168.68.180, VMID 102), with root SSH to the NUC's Proxmox host **192.168.68.100** (now PVE 9, Arc iGPU verified in Phase 3). This phase builds the two permanent guests the NUC keeps for good: the **workstation VM** (Claude Code + XFCE + Chrome + Claude Desktop) and a new **LLM LXC** using the Arc iGPU, **uncontended** since Frigate and HAOS both moved off the NUC in earlier phases. The Coding VM keeps running (it gets destroyed in Phase 9, not now).

**Why an LXC for the LLM, not a VM**: an LXC shares the host kernel directly — GPU device access via `dev0` is a straight passthrough, no hypervisor translation layer, so it performs essentially at bare-metal speed for both GPU and CPU-bound inference. This was the explicit design choice after Lawrie prioritized LLM responsiveness (especially for camera image description) over anything else.

## Target

| Item | Value |
|---|---|
| **workstation** VMID | **112** |
| **workstation** resources | **8GB RAM** (→16GB in Phase 9) / 6 vCPU / 150GB disk. 8GB is deliberate: with the Coding VM (6GB) + LLM LXC (8GB staged) still running, this keeps total NUC guest allocation at **22GB** on the 32GB host. **Do not create it with 16GB — Phase 9 raises it after the Coding VM is destroyed.** |
| **workstation** IP | `192.168.68.103` (**not** .180 — the Coding VM still owns that) |
| **LLM** CTID | **113** |
| **LLM** resources | **8GB RAM** (→12GB in Phase 9) / 4 vCPU / 100GB disk, unprivileged LXC |
| **LLM** IP | `192.168.68.104` |
| **LLM** GPU | Arc iGPU via `dev0: /dev/dri/renderD128,gid=<render gid in CT>` — same pattern as the Frigate LXC, just on the NUC instead of the HP Mini |

## Preconditions

- Read `~/rebuild/reports/phase-3.md` (storage names, confirms **Xe driver** in use) and confirm phases 4, 5, 6, 7 reports exist (Pi 5 HA, HP Mini bootstrap, HP Mini Frigate, HA↔Frigate integration all healthy).
- Free space check: `ssh root@192.168.68.100 'pvesm status'` — 150GB (workstation) + 100GB (LLM) must fit comfortably.

---
# Part 1 — Workstation VM

## 1. Create the VM and install Debian 13

```bash
ssh root@192.168.68.100 '
  cd /var/lib/vz/template/iso && wget -q https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.*-amd64-netinst.iso || ls debian-13*.iso
  qm create 112 --name workstation --memory 8192 --cores 6 --cpu host \
    --machine q35 --bios ovmf --efidisk0 <STORE>:1,efitype=4m \
    --scsihw virtio-scsi-single --scsi0 <STORE>:150,discard=on,ssd=1 \
    --net0 virtio,bridge=vmbr0 --ostype l26 --agent enabled=1 \
    --ide2 local:iso/<the-iso>,media=cdrom --boot order="scsi0;ide2" \
    --vga std --onboot 1 --startup order=1,up=30
  qm start 112
'
```

The Debian installer is interactive — ask Lawrie to open the VM's **Console** in the Proxmox web UI and walk them through it: language/locale/keyboard; hostname `workstation`; user `lawrie` (+ root password or sudo-via-first-user); guided partitioning, whole disk, ext4, no LVM complications needed; **software selection: deselect every desktop environment, select only "SSH server" and "standard system utilities"** (XFCE is installed minimal in the next step). After install completes and the VM reboots, detach the ISO: `ssh root@192.168.68.100 'qm set 112 --ide2 none --boot order=scsi0'`.

## 2. DHCP reservation — needs Lawrie

`ssh root@192.168.68.100 'qm config 112 | grep net0'` → MAC to Lawrie → reservation for **192.168.68.103** → reboot the VM (`qm reboot 112`) → confirm `ping -c1 192.168.68.103` and `ssh lawrie@192.168.68.103` works (copy this VM's SSH key over first: `ssh-copy-id lawrie@192.168.68.103`, password auth once).

All remaining workstation steps run over SSH from here: `ssh lawrie@192.168.68.103` (use `sudo` as needed; if sudo isn't set up, `su -c 'apt install -y sudo && usermod -aG sudo lawrie'` via root once, then re-login).

## 3. Guest agent, XFCE (minimal), xrdp

```bash
sudo apt update && sudo apt install -y qemu-guest-agent
sudo apt install -y --no-install-recommends xfce4 xfce4-terminal xfce4-goodies dbus-x11
sudo apt install -y xrdp
echo xfce4-session > ~/.xsession
sudo adduser xrdp ssl-cert
sudo systemctl enable --now xrdp
```

Verify: `ss -tlnp | grep 3389` listening. Ask Lawrie to RDP to `192.168.68.103` from their machine (any RDP client), log in as `lawrie`, and confirm an XFCE desktop appears.

## 4. Chrome

```bash
sudo curl -fsSLo /usr/share/keyrings/google-chrome.asc https://dl.google.com/linux/linux_signing_key.pub
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome.asc] https://dl.google.com/linux/chrome/deb/ stable main" | sudo tee /etc/apt/sources.list.d/google-chrome.list
sudo apt update && sudo apt install -y google-chrome-stable
```

## 5. Claude Desktop (Linux beta) — the primary Claude app

Debian 13 x86_64 meets the beta requirements (Debian 12+). Install from Anthropic's apt repo so updates come with `apt upgrade`:

```bash
sudo curl -fsSLo /usr/share/keyrings/claude-desktop-archive-keyring.asc https://downloads.claude.ai/claude-desktop/key.asc
gpg --show-keys /usr/share/keyrings/claude-desktop-archive-keyring.asc   # fingerprint must be 31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/claude-desktop-archive-keyring.asc] https://downloads.claude.ai/claude-desktop/apt/stable stable main" | sudo tee /etc/apt/sources.list.d/claude-desktop.list
sudo apt update && sudo apt install -y claude-desktop
```

**Verify the fingerprint matches before installing.** Known Linux-beta limits to tell Lawrie: no Computer Use (screen control), no in-app dictation; Chat, Cowork, and Code tabs all work, including parallel sessions, diff review, integrated terminal/editor.

## 6. Node.js + Claude Code CLI (kept alongside Desktop — covers headless/SSH use)

```bash
sudo apt install -y nodejs npm git   # Debian 13 ships Node ≥ 20; verify: node --version
curl -fsSL https://claude.ai/install.sh | bash   # native installer; ensure ~/.local/bin on PATH
claude --version
```

If the Coding VM instead used an npm-global install (`npm ls -g` here will tell you), mirror whichever setup the Coding VM has, so migrated config behaves identically.

## 7. Migrate Lawrie's data from the Coding VM

Run from the **Coding VM** (here), pushing to the workstation. Migrate, then verify each item:

```bash
rsync -aP ~/claude/     lawrie@192.168.68.103:~/claude/
rsync -aP ~/.claude/    lawrie@192.168.68.103:~/.claude/      # memory, settings, plans, projects
rsync -aP ~/.claude.json lawrie@192.168.68.103:~/ 2>/dev/null || true
rsync -aP ~/awa/        lawrie@192.168.68.103:~/awa/
rsync -aP ~/.gitconfig  lawrie@192.168.68.103:~/
rsync -aP ~/.git-credentials lawrie@192.168.68.103:~/ 2>/dev/null || true
rsync -aP ~/.ssh/       lawrie@192.168.68.103:~/.ssh/          # includes host-authorized keys from Phases 0 and 5
rsync -aP ~/rebuild/    lawrie@192.168.68.103:~/rebuild/       # salvage + phase reports — Phase 9 runs from there
```

Then sweep for stragglers and migrate anything relevant you find: `npm ls -g --depth=0` (recreate globals on the workstation), MCP server configs and tokens (check `~/.claude.json`, `~/.config/`, project `.mcp.json` files), crontabs (`crontab -l`), shell rc customisations (`~/.bashrc` additions), and anything else in `~` that isn't OS noise (`ls -la ~`). Fix perms on arrival: `ssh lawrie@192.168.68.103 'chmod 700 ~/.ssh && chmod 600 ~/.ssh/*'`.

Verify from the workstation: `ssh lawrie@192.168.68.103 'ssh -o BatchMode=yes root@192.168.68.100 echo nuc-ok && ssh -o BatchMode=yes root@192.168.68.110 echo hpmini-ok && ls ~/rebuild/reports/'` — the migrated key must reach **both** Proxmox hosts (NUC and HP Mini) and all phase reports must be present.

## 8. Lawrie's sign-in session (RDP)

Ask Lawrie to RDP into `192.168.68.103` and:
1. Launch **Claude Desktop** (app menu or `claude-desktop`), sign in with the claude.ai account (Desktop doesn't take API keys).
2. Sign into **Chrome**, install the **Claude in Chrome** extension, and connect it.
3. Open a terminal, run `claude` in `~/claude/<a-project>`, sign in if prompted, and confirm memory/settings carried over (e.g. `/config` shows migrated settings).

---
# Part 2 — LLM LXC (Arc iGPU, IPEX-LLM)

## 9. Create the container

```bash
ssh root@192.168.68.100 '
  pct create 113 local:vztmpl/debian-13-standard_<latest>_amd64.tar.zst \
    --hostname llm --unprivileged 1 --features nesting=0 \
    --memory 8192 --cores 4 --rootfs <STORE>:100 \
    --net0 name=eth0,bridge=vmbr0,ip=dhcp --ostype debian \
    --onboot 1 --startup order=2,up=20
'
```

(Reuse the same `debian-13-standard` template already downloaded for other LXCs if present; `pveam download local debian-13-standard_<latest>_amd64.tar.zst` if not.) No Docker/nesting needed here — the serving stack runs natively in the LXC for one less layer between it and the GPU.

## 10. GPU passthrough

```bash
ssh root@192.168.68.100 'pct start 113 && sleep 5 && pct exec 113 -- getent group render'
# e.g. "render:x:104:" → gid 104
ssh root@192.168.68.100 'pct set 113 --dev0 /dev/dri/renderD128,gid=<render_gid>'
ssh root@192.168.68.100 'pct reboot 113 && sleep 5 && pct exec 113 -- ls -ln /dev/dri/'
```

Expected: `renderD128` visible inside the CT, group-owned by the render gid.

## 11. DHCP reservation — needs Lawrie

`ssh root@192.168.68.100 'pct config 113 | grep net0'` → MAC to Lawrie → reservation for **192.168.68.104** → `pct reboot 113` → confirm.

## 12. Intel GPU compute runtime

Before any LLM serving stack, the container needs the Intel compute runtime (separate from the display/decode drivers Frigate uses — this is the OpenCL/Level-Zero compute path):

```bash
ssh root@192.168.68.100 'pct exec 113 -- bash -c "
  apt update && apt install -y wget gnupg
  apt install -y intel-opencl-icd intel-level-zero-gpu level-zero clinfo
  clinfo | grep -A5 \"Device Name\"
"'
```

Expected: `clinfo` lists the Arc iGPU as an OpenCL/Level-Zero compute device. If it doesn't appear, double check the `dev0` passthrough (step 10) and that the host's Xe driver (verified in Phase 3) is actually the one bound to the GPU.

## 13. IPEX-LLM + Ollama

Intel's **ipex-llm** project provides an Ollama-compatible portable build optimized for Arc's Xe architecture via oneAPI/SYCL — this is what gives the meaningful speed advantage over a generic Vulkan/CPU backend. Package names and exact install steps shift over time, so **check the current quickstart at the ipex-llm GitHub repo** (`intel/ipex-llm`, "Ollama" quickstart doc) rather than trusting a stale command verbatim; as of this rebuild's design, the pattern is:

```bash
ssh root@192.168.68.100 'pct exec 113 -- bash -c "
  cd /opt && wget <current ipex-llm[cpp] portable tgz URL from the quickstart docs>
  tar -xzf ipex-llm-*.tar.gz && cd ipex-llm-*
  # the portable build ships an ollama binary pre-built with IPEX-LLM SYCL support
  ./ollama serve &
"'
```

Set it up as a systemd service (not a backgrounded shell job) so it survives reboots — write a unit file, `systemctl enable --now`. Confirm the service binds on `0.0.0.0:11434` (or reverse-proxy/firewall it to be reachable from the LAN, not just localhost) since both the Pi 5 (for `llmvision`) and the workstation (for chat/coding) need to reach it over the network.

Verify GPU is actually being used, not silently falling back to CPU:

```bash
ssh root@192.168.68.100 'timeout 15 intel_gpu_top -o -' &
ssh root@192.168.68.100 "pct exec 113 -- curl -s http://localhost:11434/api/generate -d '{\"model\":\"<test-model>\",\"prompt\":\"hello\"}'"
```

`intel_gpu_top` should show Render/Compute engine activity while the request runs.

## 14. Pull the models

**Vision model** (fast, small, for camera captioning — responsiveness is the priority here):

```bash
ssh root@192.168.68.100 'pct exec 113 -- ./ollama pull moondream'   # or qwen2-vl:2b if preferred — check current Ollama library for the best small vision model available
```

**General chat/coding model** (larger, latency less critical):

```bash
ssh root@192.168.68.100 'pct exec 113 -- ./ollama pull qwen2.5:14b'   # or llama3.1:8b / qwen2.5-coder:7b — pick based on what fits comfortably in the 8-12GB budget once loaded
```

Keep the vision model **warm** so camera-event captioning has no cold-load penalty:

```bash
ssh root@192.168.68.100 'pct exec 113 -- bash -c "echo OLLAMA_KEEP_ALIVE=-1 >> /etc/environment"'   # or pass keep_alive:-1 per-request from the llmvision integration config
```

Given the 8-12GB RAM budget, don't keep both models warm simultaneously unless testing shows it fits — prioritize the vision model staying resident since that's the latency-sensitive path; the chat/coding model can tolerate a load delay.

## 15. Point HA's llmvision integration at this endpoint

The `llmvision` custom integration is currently **not installed** (an empty leftover folder was found in the pre-rebuild audit) — install it fresh via HACS on the Pi 5's HA instance (Settings → Devices & services → Add integration → LLM Vision, or via HACS if it's not in HA's default integration list — check the project's current install docs). Configure its provider as a local Ollama endpoint pointing at `http://192.168.68.104:11434`, model `moondream` (or whichever vision model was pulled). Leave the general chat/coding model unconfigured in HA — that one's for Lawrie's direct use from the workstation, not a HA integration.

## 16. Handoff report

Write `~/rebuild/reports/phase-8.md`: workstation VMID/MAC, migration byte counts, sweep findings, Desktop/CLI versions, sign-in confirmation; LLM LXC CTID/MAC, render gid, `clinfo`/`intel_gpu_top` confirmation of GPU use, models pulled and their sizes, `llmvision` HA configuration confirmed, current RAM/disk usage on both new guests as a baseline.

## Verification checklist

- [ ] VM 112 (workstation): 8GB/6vCPU/150GB, `.103`, agent responding, RDP → XFCE works, Chrome + extension working, Claude Desktop signed in, `claude` CLI works with migrated config.
- [ ] Both Proxmox hosts (NUC `.100`, HP Mini `.110`) reachable via the migrated SSH key from the workstation.
- [ ] CT 113 (LLM): 8GB/4vCPU/100GB, `.104`, `renderD128` visible with correct gid.
- [ ] `clinfo` shows the Arc iGPU as a compute device; `intel_gpu_top` shows load during inference — confirms GPU, not CPU fallback.
- [ ] Vision model responds to a test image description request; general model responds to a test chat/coding prompt.
- [ ] `llmvision` installed in HA and pointed at `http://192.168.68.104:11434`.
- [ ] Coding VM untouched and still running.
- [ ] `~/rebuild/reports/phase-8.md` written.

## Rollback

The Coding VM remains fully intact — nothing is deleted in this phase. Workstation: `qm stop 112 && qm destroy 112 --purge` and re-run. LLM LXC: `pct stop 113 && pct destroy 113 --purge` and re-run — no other guest depends on it yet.

## Exit criteria

Workstation fully operational and data-complete; LLM LXC serving both a warm vision model and a general model on the Arc iGPU, reachable from both HA and the workstation; Coding VM now redundant but alive. Next phase: `phase-9-verify-decommission.md` — **run that session from the workstation VM, not from here.**
