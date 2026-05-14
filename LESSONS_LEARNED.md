# Ronin 48 — Training Lessons Learned

Dependency conflicts, environment bugs, and hard-won fixes from every training run across
the Ronin 48 model suite. Read this before you touch RunPod.

---

## The Root Cause of Most Errors

The stock RunPod PyTorch template ships with **torch 2.11.0** (or similar dev/nightly
builds). Most pinned ML packages on PyPI were built against torch 2.2.x or earlier.
This causes a cascade of incompatibilities that are not obvious from the error messages.
Every fix below traces back to this root cause.

**The solution:** Use the `ronin48/qlora-training` Docker image as your RunPod template.
It has all of this pre-solved.

If you insist on the stock template, read on.

---

## The Golden Path

If you're starting from scratch on RunPod:

1. Deploy a pod with the `ronin48/qlora-training` Docker image
2. Use a **300 GB network volume**, set `volumeMountPath: "/workspace"` (see Error #21)
3. Open the **web terminal** (not SSH — see Error #9)
4. Run the model's launch script:
   ```bash
   HF_TOKEN=your_token bash <(curl -s https://codeberg.org/Ronin48/<MODEL>/raw/branch/main/scripts/launch.sh)
   ```
5. Watch for `[train] starting QLoRA training...` then GPU spike > 10%
6. Walk away — training takes 8–10 hours

---

## Package Versions That Work Together

As of May 2026 on `runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04`:

| Package | Pin | Reason |
|---|---|---|
| transformers | `>=4.46.0,<4.50.0` | 4.50+ adds MoE custom ops that crash on PyTorch 2.4 |
| trl | `>=0.12.0,<1.0.0` | 1.0+ imports `is_trackio_available` from transformers>=4.50 |
| peft | latest (`--upgrade`) | |
| bitsandbytes | latest (`--upgrade`) | old versions lack CUDA 13 binaries |
| accelerate | latest (`--upgrade`) | |
| torchvision | **uninstalled** | Crashes on torch 2.4+ — remove it |
| triton | torch-managed | Don't pin separately |

These pins are baked into all `scripts/launch.sh` files.

---

## Error Log

### Error #1 — FlashAttention2 ImportError

**Error:**
```
ImportError: FlashAttention2 has been toggled on, but it cannot be used due to the
following error: the package flash_attn seems to be not installed.
```

**Cause:** `configs/training_config.yaml` had `attn_implementation: "flash_attention_2"`.
The stock RunPod template doesn't have `flash_attn` installed.

**Fix:** Set `attn_implementation: "eager"` in `configs/training_config.yaml`.

---

### Error #2 — bitsandbytes CUDA Library Missing

**Error:**
```
RuntimeError: CUDA SETUP ERROR: Missing dependency: libnvJitLink.so.13
```

**Cause:** `libnvJitLink.so.13` ships via pip as part of the `nvidia-cu13` package
but isn't in `LD_LIBRARY_PATH` by default.

**Fix:**
```bash
export LD_LIBRARY_PATH=/usr/local/lib/python3.10/dist-packages/nvidia/cu13/lib:${LD_LIBRARY_PATH:-}
```

See also Error #14 — if using Python 3.11+, derive this path dynamically.

---

### Error #3 — torchvision Crashes on Import

**Error:**
```
RuntimeError: operator torchvision::nms does not exist
```

**Cause:** torchvision built for torch 2.2.0 crashes on torch 2.4+. The crash cascades
up through transformers → bloom model → peft → your training script.

**Fix:** Uninstall torchvision. It's not needed for text-only training.
```bash
pip uninstall torchvision -y
```

---

### Error #4 — bitsandbytes Version Too Old

**Error:**
```
Could not find the bitsandbytes CUDA binary at libbitsandbytes_cuda130.so
```

**Fix:** Don't pin bitsandbytes. Let pip resolve it:
```bash
pip install bitsandbytes
```

---

### Error #5 — triton Version Conflict

**Error:**
```
No module named 'triton.ops'
```

**Cause:** `triton.ops` was removed in triton 3.x. Pinning bitsandbytes pulled in a version
that expected triton 2.x internals.

**Fix:** Don't pin triton or bitsandbytes.

---

### Error #6 — `.to()` Not Supported on 4-bit Models

**Error:**
```
ValueError: `.to` is not supported for `4-bit` or `8-bit` bitsandbytes models.
```

**Cause:** Passing `torch_dtype` or `device_map` to `from_pretrained()` alongside
`quantization_config`. bitsandbytes manages dtype and placement internally.

**Fix:**
```python
# Wrong:
model = AutoModelForCausalLM.from_pretrained(
    model_name, quantization_config=bnb_config,
    device_map="auto", torch_dtype=torch.bfloat16,
)

# Right:
model = AutoModelForCausalLM.from_pretrained(
    model_name, quantization_config=bnb_config,
)
```

---

### Error #7 — peft / transformers Version Mismatch

**Error:**
```
ModuleNotFoundError: Could not import module 'BloomPreTrainedModel'.
```

**Fix:**
```bash
pip install peft --upgrade
```

---

### Error #8 — Disk Quota Exceeded

**Error:**
```
OSError: I/O error: IO Error: Disk quota exceeded (os error 122)
```

**Cause:** Llama-3.3-70B weights are ~140 GB across 30 shards. A 200 GB volume fills
fast once you add dataset, checkpoints, and adapter output.

**Fix:** Use a **300 GB network volume** minimum.

---

### Error #9 — SSH Connection Refused / dockerArgs Ignored

**Symptom:** `ssh: connect to host <ip> port <port>: Connection refused`

**Cause:** The RunPod PyTorch template does not launch sshd. `dockerArgs` and `startupScript`
are silently ignored when a `templateId` is set.

**Fix:** Use the **RunPod web terminal**.
In the RunPod console: **Connect → Web Terminal**. Run training from there.

---

### Error #10 — HuggingFace Weights Downloading to Container Disk

**Symptom:** Disk full errors mid-download even with a large network volume attached.

**Cause:** HuggingFace defaults to `~/.cache/huggingface` — on the 50 GB container disk,
not the network volume.

**Fix:** Always set before running anything:
```bash
export HF_HOME=/workspace/hf_cache
```

This is set automatically by `scripts/launch.sh`.

---

### Error #11 — bitsandbytes Custom Op Schema Error (PyTorch 2.4 + Python 3.11)

**Error:**
```
ValueError: infer_schema(func): Parameter input has unsupported type torch.Tensor.
```

**Cause:** Older bitsandbytes versions used string-quoted type annotations in custom op
registrations. PyTorch 2.4's `infer_schema` rejects string annotations.

**Fix:**
```bash
pip install --upgrade bitsandbytes
```

---

### Error #12 — transformers MoE Custom Op Schema Error (transformers >= 4.50 + PyTorch 2.4)

**Error:**
```
ValueError: infer_schema(func): Parameter input has unsupported type torch.Tensor.
```

**Cause:** transformers 4.50+ added FP8/MoE expert integrations that register custom ops
at import time using string-quoted type annotations that PyTorch 2.4 rejects.

**Fix:**
```bash
pip install "transformers>=4.46.0,<4.50.0"
```

---

### Error #13 — trl / transformers Version Sandwich (PyTorch 2.4)

**Error:**
```
ImportError: cannot import name 'is_trackio_available' from 'transformers'
```

**Cause:** trl>=1.0.0 requires transformers>=4.50, but transformers>=4.50 crashes on
PyTorch 2.4 (see Error #12). The two pins conflict.

**Fix:** Pin both together:
```bash
pip install "transformers>=4.46.0,<4.50.0" "trl>=0.12.0,<1.0.0"
```

---

### Error #14 — LD_LIBRARY_PATH Hardcoded to Python 3.10

**Symptom:** bitsandbytes CUDA library not found on Python 3.11+ images.

**Fix:** Derive the path dynamically:
```bash
_nvidia_lib=$(python3 -c "import site; print(site.getsitepackages()[0])")/nvidia/cu13/lib
[ -d "$_nvidia_lib" ] && export LD_LIBRARY_PATH="$_nvidia_lib:${LD_LIBRARY_PATH:-}"
```

---

### Error #15 — SFTTrainer `tokenizer` Renamed to `processing_class` (trl 0.12+)

**Error:**
```
TypeError: SFTTrainer.__init__() got an unexpected keyword argument 'tokenizer'
```

**Fix:** Replace `tokenizer=tokenizer` with `processing_class=tokenizer`.

---

### Error #16 — SFTTrainer `max_seq_length` Moved to `SFTConfig` (trl 0.12+)

**Error:**
```
TypeError: SFTTrainer.__init__() got an unexpected keyword argument 'max_seq_length'
```

**Fix:**
```python
from trl import SFTConfig, SFTTrainer
training_args = SFTConfig(max_seq_length=4096, output_dir=..., ...)
trainer = SFTTrainer(...)
```

---

### Error #17 — transformers Version Creep (`--upgrade` Overrides the Pin)

**Symptom:** The schema crash from Error #12 or #13 returns after running `pip install --upgrade`.

**Cause:** Upgrading any package without the transformers pin can pull in transformers 5.x
as a transitive dependency.

**Fix:** Always reinstall the full pinned stack together:
```bash
pip install "transformers>=4.46.0,<4.50.0" "trl>=0.12.0,<1.0.0"
```

Never run bare `pip install --upgrade transformers`.

---

### Error #18 — Web Terminal Drops Kill the Training Process

**Symptom:** RunPod web terminal disconnects. Training process dies.

**Cause:** Training runs as a foreground process. When the browser disconnects, SIGHUP
kills it.

**Fix:** Use `nohup`:
```bash
nohup python scripts/training/train_qlora.py --config configs/training_config.yaml \
  >> /workspace/logs/train.log 2>&1 &
tail -f /workspace/logs/train.log
```

Or use tmux:
```bash
apt-get install tmux && tmux new -s training
# Ctrl+B + D to detach, tmux attach -t training to reconnect
```

---

### Error #19 — `huggingface-cli` Deprecated — Upload Silently Fails

**Symptom:** Upload step exits with a deprecation warning. Adapter never appears on HuggingFace.

**Fix:**
```bash
# Wrong
huggingface-cli upload "$HF_REPO" "$ADAPTER_DIR" --token "$HF_TOKEN"

# Correct
hf upload "$HF_REPO" "$ADAPTER_DIR" --token "$HF_TOKEN"
```

---

### Error #20 — Training Monitor Blind to Pre-GPU Crashes (Happened 3 Times)

**Symptom:** Monitor reports `GPU 0% | active=False` indefinitely. Training never starts.
Pod burns credits doing nothing.

**Cause:**
1. **Wrong volume size.** Llama-3.3-70B needs ~140 GB for weights alone. A 100 GB volume
   fills at shard 22/30 with `OSError: [Errno 28] No space left on device` — before the GPU
   is ever touched.
2. **Wrong cache mount.** If `HF_HOME` is not set, weights download to `~/.cache/huggingface`
   on the 50 GB container disk, not the network volume.
3. **Monitor was blind.** It only detected completion (GPU active → idle). It had no alert for
   GPU never activating at all.

**Fixes applied to `training_monitor.py`:**
- Pre-flight check on first pod sighting: volume size, HF cache mount, free space
- Stuck-pod alert: Telegram notification after 45 min with GPU never active
- Mid-training disk polling: alert at 85% full

**Rule: `GPU 0% | active=False` after 20+ minutes = crashed. SSH in and check
`/workspace/logs/train.log`. Never assume it's still initializing.**

---

### Error #21 — `volumeMountPath` Must Be Set in API Deployments (2026-05-14)

**Symptom:** Pod allocates successfully (API returns a pod ID), appears in dashboard as
"starting," but `runtime` stays `null` forever. SSH never becomes available.

**Root cause:** The RunPod GraphQL `podFindAndDeployOnDemand` mutation requires
`volumeMountPath` whenever `volumeInGb > 0`. When omitted, Docker rejects the container:

```
invalid mount config for type "volume": field Target must not be empty
```

The pod retries indefinitely. The API returns no error — you have to check the pod's Docker
logs in the RunPod web console to see it.

**Fix:** Always include `volumeMountPath`:
```python
"input": {
    ...
    "volumeInGb": 300,
    "volumeMountPath": "/workspace",   # required when volumeInGb > 0
    ...
}
```

---

### Error #22 — NFS I/O Error During Model Download (2026-05-14)

**Symptom:** Training crashed at shard 3/30 with `OSError: [Errno 5] Input/output error`.
GPU never activated. Disk had plenty of free space.

**Root cause:** Transient RunPod NFS filesystem error — not disk-full, not OOM.

**Fix:** Terminate and re-launch. No code changes needed.

---

### Error #23 — SFTTrainer `dataset_text_field` Moved to `SFTConfig` (trl 0.12+)

**Error:**
```
TypeError: SFTTrainer.__init__() got an unexpected keyword argument 'dataset_text_field'
```

**Cause:** trl 0.12 moved `dataset_text_field` from `SFTTrainer` into `SFTConfig`, same as
`max_seq_length` (Error #16) and `tokenizer` → `processing_class` (Error #15).

**Fix:** Move it to `SFTConfig`:
```python
training_args = SFTConfig(
    dataset_text_field="text",   # ← here, not in SFTTrainer
    max_seq_length=...,
    ...
)
trainer = SFTTrainer(
    model=model,
    processing_class=tokenizer,
    args=training_args,
    ...
)
```

---

## Contributing

If you hit a new error and fix it, add it here. The people walking behind you will thank you.
