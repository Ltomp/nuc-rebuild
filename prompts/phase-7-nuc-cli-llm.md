# Phase 7 — NUC: Claude Code CLI (confirm) + IPEX-LLM

You are Claude Code (Sonnet) running directly on the **NUC** (192.168.68.103, Debian 13, bare-metal — installed in Phase 2, Claude Code and your own data already restored there). This phase installs **IPEX-LLM** natively on the Arc iGPU — no LXC, no VM, no passthrough config, `/dev/dri` is just this machine's own device — and brings up two local models: a vision model kept warm for Frigate camera-event captioning via `llmvision`, and a general chat/coding model. The Arc iGPU is **uncontended**: Frigate is on the HP Mini's UHD 770, HA is on the Pi 5 — nothing else on this machine competes for it.

## Preconditions

- Read `~/rebuild/reports/phase-2.md` (confirms Claude Code CLI and your data were restored here) and confirm phases 3-6 reports exist (HP Mini bootstrap, Pi 5 HA, Frigate, HA↔Frigate integration all healthy).
- Quick self-check — this should already be true, not something to redo:

```bash
claude --version
ls ~/rebuild/reports/
```

If `claude` isn't working or the restore looks incomplete, go back to Phase 2's steps 3-4 rather than improvising here.

## Steps

### 1. Intel GPU compute runtime

Before any LLM serving stack, this needs the Intel compute runtime (separate from the display/decode drivers — this is the OpenCL/Level-Zero compute path):

```bash
sudo apt update && sudo apt install -y intel-opencl-icd intel-level-zero-gpu level-zero clinfo
clinfo | grep -A5 "Device Name"
```

Expected: `clinfo` lists the Arc iGPU as an OpenCL/Level-Zero compute device. If it doesn't appear, check `ls -l /dev/dri/` (should already show `renderD128` from Phase 2's install) and confirm the Xe driver is bound: `lsmod | grep -E "^i915|^xe"`.

### 2. IPEX-LLM + Ollama

Intel's **ipex-llm** project provides an Ollama-compatible portable build optimized for Arc's Xe architecture via oneAPI/SYCL — this is what gives the meaningful speed advantage over a generic Vulkan/CPU backend. Package names and exact install steps shift over time, so **check the current quickstart at the ipex-llm GitHub repo** (`intel/ipex-llm`, "Ollama" quickstart doc) rather than trusting a stale command verbatim; as of this rebuild's design, the pattern is:

```bash
cd /opt && sudo wget <current ipex-llm[cpp] portable tgz URL from the quickstart docs>
sudo tar -xzf ipex-llm-*.tar.gz && cd ipex-llm-*
# the portable build ships an ollama binary pre-built with IPEX-LLM SYCL support
sudo ./ollama serve &
```

Set it up as a systemd service (not a backgrounded shell job) so it survives reboots — write a unit file, `sudo systemctl enable --now ollama`. Confirm the service binds on `0.0.0.0:11434` (or firewall/reverse-proxy it to be reachable from the LAN, not just localhost) since both the Pi 5 (for `llmvision`) and Lawrie directly (for chat/coding) need to reach it over the network.

Verify GPU is actually being used, not silently falling back to CPU:

```bash
timeout 15 intel_gpu_top -o - &
curl -s http://localhost:11434/api/generate -d '{"model":"<test-model>","prompt":"hello"}'
```

`intel_gpu_top` should show Render/Compute engine activity while the request runs.

### 3. Pull the models

**Vision model** (fast, small, for camera captioning — responsiveness is the priority here):

```bash
ollama pull moondream   # or qwen2-vl:2b if preferred — check current Ollama library for the best small vision model available
```

**General chat/coding model** (larger, latency less critical):

```bash
ollama pull qwen2.5:14b   # or llama3.1:8b / qwen2.5-coder:7b — pick based on what fits comfortably in RAM once loaded (32GB total, uncontended by any other guest now)
```

Keep the vision model **warm** so camera-event captioning has no cold-load penalty:

```bash
echo 'OLLAMA_KEEP_ALIVE=-1' | sudo tee -a /etc/environment   # or pass keep_alive:-1 per-request from the llmvision integration config
```

With the full 32GB uncontended (no LXC/VM RAM budget to split anymore), both models can likely stay resident simultaneously — but prioritize the vision model staying warm if memory pressure appears, since that's the latency-sensitive path; the chat/coding model can tolerate a load delay.

### 4. Point HA's llmvision integration at this endpoint

The `llmvision` custom integration is currently **not installed** (an empty leftover folder was found in the pre-rebuild audit) — install it fresh via HACS on the Pi 5's HA instance (Settings → Devices & services → Add integration → LLM Vision, or via HACS if it's not in HA's default integration list — check the project's current install docs). Configure its provider as a local Ollama endpoint pointing at `http://192.168.68.103:11434`, model `moondream` (or whichever vision model was pulled). Leave the general chat/coding model unconfigured in HA — that one's for Lawrie's direct use from this NUC, not a HA integration.

### 5. Wire it into the Frigate/HA event pipeline

Extend the notification automation from Phase 6 (or add a parallel one) so that on a Frigate person event, `llmvision` generates a description of the snapshot and includes it in the phone notification, alongside the existing image. Consult `llmvision`'s current documentation for its exact service-call signature (it typically exposes an `llmvision.image_analyzer` or similar action callable from an automation). Test with a live walk-past and confirm the notification text is sensible and arrives promptly.

### 6. Handoff report

Write `~/rebuild/reports/phase-7.md`: `clinfo`/`intel_gpu_top` confirmation of GPU use (not CPU fallback), models pulled and their sizes, systemd service confirmed enabled, `llmvision` HA configuration confirmed, the extended notification automation's YAML, live-test result (description quality + latency), current RAM/disk usage on the NUC as a baseline.

## Verification checklist

- [ ] `clinfo` shows the Arc iGPU as a compute device; `intel_gpu_top` shows load during inference — confirms GPU, not CPU fallback.
- [ ] Ollama running as a systemd service, reachable from the LAN at `192.168.68.103:11434`.
- [ ] Vision model responds to a test image description request; general model responds to a test chat/coding prompt.
- [ ] `llmvision` installed in HA and pointed at `http://192.168.68.103:11434`.
- [ ] Live walk-test: Frigate event → `llmvision` description → phone notification, end to end.
- [ ] `~/rebuild/reports/phase-7.md` written.

## Rollback

`sudo systemctl disable --now ollama` and remove `/opt/ipex-llm-*` to back out the LLM stack entirely — nothing else on this machine depends on it. HA-side: remove the `llmvision` integration and revert the automation to Phase 6's simpler version.

## Exit criteria

IPEX-LLM serving both a warm vision model and a general model on the Arc iGPU, reachable from both HA and this NUC directly, wired into the camera notification pipeline and proven live. Next phase: `phase-8-verify.md`.
