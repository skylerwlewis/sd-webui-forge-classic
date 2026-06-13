# macOS / Apple Silicon Setup Notes

**Date:** 2026-06-07  
**Repo:** https://github.com/Haoming02/sd-webui-forge-classic  
**Branch:** `neo`  
**Commit at time of setup:** `6b47c95339227f9fd822248ed514ef9fa7b702c1`  
**Machine:** M4 Max MacBook Pro (Apple Silicon, MPS backend)

---

## Context

**Context:** This repo is "Forge Neo" — an actively maintained fork of [lllyasviel/stable-diffusion-webui-forge](https://github.com/lllyasviel/stable-diffusion-webui-forge), 839 commits ahead of upstream as of the date above. This is the active installation for this machine; models are stored in the `models/` folder within this repo.

---

## Changes Made

### 1. `webui-user.sh` — Uncommented `TORCH_COMMAND` for macOS

The default `TORCH_COMMAND` was commented out. On macOS, PyTorch from PyPI installs an MPS-compatible build. The CUDA index URL used by the default installer produces a CUDA-only build that doesn't work on Apple Silicon.

**Change:** Uncommented and kept:
```bash
export TORCH_COMMAND="pip install torch==2.12.0 torchvision==0.27.0"
```
(No `--index-url https://download.pytorch.org/whl/cu...` — standard PyPI build supports MPS.)

The wiki even calls this out: *"For macOS, you may need to uncomment the `TORCH_COMMAND` line to install regular PyTorch instead of the CUDA variant"*

---

### 2. `modules/launch_utils.py` — Fixed crash on plain PyTorch version strings

**File:** `modules/launch_utils.py`, function `_torch_version()` (around line 95)

**Problem:** The regex `r"(\d+\.\d+\.\d+)(?:[^+]+)?\+(.+)"` requires a `+backend` suffix (e.g. `2.12.0+cu130`). MPS/CPU builds from PyPI have plain version strings like `2.12.0` — no suffix. The fallback in the `if m is None` block *also* ran the same regex, so it always crashed with `AttributeError: 'NoneType' object has no attribute 'group'`.

**Fix:** Added a third fallback for plain version strings, returning `"cpu"` as the build tag:
```python
ver = importlib.metadata.version("torch")
m = re.search(r"(\d+\.\d+\.\d+)(?:[^+]+)?\+(.+)", ver)

if m is None:
    env_ver = os.environ.get("PYTORCH_VERSION", None)
    if env_ver:
        m = re.search(r"(\d+\.\d+\.\d+)(?:[^+]+)?\+(.+)", env_ver)

if m is None:
    # Plain version with no build suffix (e.g. MPS/CPU builds)
    m2 = re.search(r"(\d+\.\d+\.\d+)", ver)
    if m2:
        return m2.group(1), "cpu"
    return "0.0.0", "cpu"

return m.group(1), m.group(2)
```

---

## Environment Setup (what was done for this machine)

```bash
brew install uv
cd sd-webui-forge-classic
uv venv venv --python 3.13 --seed
chmod +x webui.sh webui-user.sh
./webui-user.sh   # first run installs all dependencies
```

**Notes on attention backends:** The wiki explicitly states *"None of the attention packages works on macOS"* (SageAttention, FlashAttention, xformers). The code falls back to PyTorch's native `scaled_dot_product_attention`, which supports MPS. Do not pass `--sage`, `--flash`, or `--xformers` on macOS.

---

## Known Remaining Warnings (harmless)

- **`Caught Exception "Torch not compiled with CUDA enabled" / Memory Monitor Disabled`** — The memory monitor attempts a CUDA call and catches the exception. It is disabled gracefully. This is a code smell (should check backend *before* trying CUDA calls) but does not affect functionality. Good upstream PR candidate.
- **`insightface` warning** — Only needed for face-related preprocessors. Install manually with `pip install insightface` if needed.

---

## Suggested Upstream PRs

1. **`modules/launch_utils.py` — `_torch_version()` plain version crash** (done locally above). Straightforward fix, low risk, clearly benefits MPS and CPU users.

2. **Memory monitor CUDA assumption** — Wherever the memory monitor is initialized, it should check `torch.cuda.is_available()` *before* making CUDA calls, rather than wrapping them in a try/except. Cleaner and avoids misleading error output on MPS.

3. **Wiki / README macOS callout** — The wiki says "None of the attention packages works on macOS" but doesn't explain *why* or that MPS still works via `scaled_dot_product_attention`. Worth a small docs addition.

---

## Notes vs. Upstream (lllyasviel/stable-diffusion-webui-forge)

The original upstream project required these manual workarounds to run on MPS that Forge Neo has since addressed:
- `setuptools<70` pin (for `pkg_resources` in CLIP build)
- Custom `torch` install for MPS
- Manual `devices.py` patch for MPS detection
- `DPM++ 2M Karras` + 20-25 steps to avoid black image output with DreamShaper XL Lightning

These issues are either resolved or not applicable in Forge Neo.
