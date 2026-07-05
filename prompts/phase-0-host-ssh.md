# Phase 0 — Host SSH access

You are Claude Code (Sonnet) running on the **Coding VM** (Debian 13, IP 192.168.68.180, VMID 102) — a guest on a Proxmox VE host at **192.168.68.100** (ASUS NUC 14 Pro). This is Phase 0 of a 9-phase rebuild of that host's guests. Your only job in this phase: establish passwordless root SSH from this VM to the Proxmox host, so later phases can drive the host from here. Lawrie (the human) is available to answer questions and perform one manual step in the Proxmox web UI.

## Objective

`ssh root@192.168.68.100 'hostname'` works from this VM without a password prompt.

## Steps

### 1. Create the state directory

```bash
mkdir -p ~/rebuild/salvage ~/rebuild/reports
```

All later phases store salvage and handoff reports here.

### 2. Check for an existing SSH keypair

```bash
ls -l ~/.ssh/id_ed25519.pub ~/.ssh/id_rsa.pub 2>/dev/null
```

- If a keypair exists, use it — do not generate a second one.
- If none exists, generate one (no passphrase; these phases run unattended commands over SSH):

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "coding-vm-rebuild"
```

### 3. Test whether access already works

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 root@192.168.68.100 'hostname' && echo ALREADY-AUTHORIZED
```

If this prints the host's hostname, skip to step 6.

### 4. Give Lawrie the authorization command

Print the public key, then give Lawrie **one single command** to paste into the Proxmox web UI (**Datacenter → node → Shell**). Build it exactly like this, substituting the real public key line:

```bash
PUB=$(cat ~/.ssh/id_ed25519.pub)
echo "mkdir -p /root/.ssh && chmod 700 /root/.ssh && echo '$PUB' >> /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys && echo KEY-ADDED"
```

Show Lawrie the resulting one-liner and ask them to paste it into the node shell and confirm they saw `KEY-ADDED`. **Wait for their confirmation** — do not proceed on assumption.

### 5. Verify access

```bash
ssh -o BatchMode=yes root@192.168.68.100 'hostname && pveversion'
```

Expected: the node's hostname and a `pve-manager/8.4.x` version string. On first connect, accept the host key fingerprint (ask Lawrie to eyeball it if they want, then continue).

If it still prompts for a password: re-check the key was appended (`ssh root@... 'tail -1 /root/.ssh/authorized_keys'` won't work yet, so ask Lawrie to run `tail -1 /root/.ssh/authorized_keys` in the node shell and compare against the printed public key).

### 6. Write the handoff report

Write `~/rebuild/reports/phase-0.md` containing:
- Which keypair was used (path, and the public key line).
- Confirmation that `ssh root@192.168.68.100` works non-interactively, plus the `pveversion` output.
- Anything unusual encountered.

## Verification checklist (all must pass)

- [ ] `ssh -o BatchMode=yes root@192.168.68.100 'echo ok'` prints `ok` with no password prompt.
- [ ] `~/rebuild/salvage/` and `~/rebuild/reports/` exist.
- [ ] `~/rebuild/reports/phase-0.md` written.

## Rollback / if something fails

Nothing in this phase changes the host beyond one appended line in `/root/.ssh/authorized_keys`. To undo, Lawrie deletes that line in the node shell. There is no other state to roll back.

## Exit criteria

Passwordless root SSH to 192.168.68.100 verified; report written. Next phase: `phase-1-audit-salvage.md`, same machine.
