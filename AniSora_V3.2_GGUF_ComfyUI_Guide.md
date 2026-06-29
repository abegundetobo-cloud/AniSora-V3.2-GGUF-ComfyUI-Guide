# AniSora V3.2 GGUF on ComfyUI — Complete Setup Guide

> The first publicly documented working guide for AniSora V3.2 Q8 GGUF on ComfyUI, including full RTX 5090 / Blackwell (sm_120) support.

This guide documents everything needed to get clean, stable anime video generation from AniSora V3.2 GGUF models in ComfyUI. It took two days of debugging to figure all of this out — we're sharing it so nobody else has to go through the same pain.

---

## 1. Hardware & Software Requirements

### Tested Setup
- GPU: RTX 5090 32GB (also works on 4090, 3090 with Q4/Q5 GGUF variants)
- Platform: RunPod Secure Cloud
- Template: RunPod ComfyUI (runpod/stable-diffusion:comfy-ui-6.0.0)
- Container disk: 30GB, Persistent storage: 50GB at `/workspace`

### ⚠️ CRITICAL: RTX 5090 / Blackwell GPU Users

**The RTX 5090 uses the sm_120 (Blackwell) architecture. Standard PyTorch builds (cu124, cu128) DO NOT support this GPU and will produce pure static/noise output with no error message. You must install PyTorch cu130.**

This is the single most important thing for 5090 users. Everything else is the same as other GPUs.

---

## 2. Required Models

Download from HuggingFace and place in `/workspace/ComfyUI/models/` (persistent storage):

| Model | Source | Destination |
|-------|--------|-------------|
| Index-Anisora-V3.2-High-Q8_0.gguf (~15.9GB) | QuantStack/Index-Anisora-V3.2-GGUF | `/diffusion_models/High/` |
| Index-Anisora-V3.2-Low-Q8_0.gguf (~15.9GB) | QuantStack/Index-Anisora-V3.2-GGUF | `/diffusion_models/Low/` |
| wan_2.1_vae.safetensors | Comfy-Org/Wan_2.1_ComfyUI_repackaged | `/vae/split_files/vae/` |
| models_t5_umt5-xxl-enc-bf16.pth (~11.4GB) | Wan-AI/Wan2.1-T2V-14B | `/text_encoders/split_files/text_encoders/` |
| clip_vision_h.safetensors | Comfy-Org/Wan_2.1_ComfyUI_repackaged | `/clip_vision/split_files/clip_vision/` |

> **Note on Text Encoder:** The fp8 scaled version (`umt5_xxl_fp8_e4m3fn_scaled.safetensors`) does NOT work with `LoadWanVideoT5TextEncoder`. Use the bf16 `.pth` version above with the native `CLIPLoader` node instead.

---

## 3. Required Custom Nodes

- **ComfyUI-GGUF** (city96) — for `UnetLoaderGGUF` node
- **ComfyUI-VideoHelperSuite** (Kosinkadink) — for `VHS_VideoCombine`

> **WanVideoWrapper (kijai) is NOT required for this workflow.** The native ComfyUI nodes handle everything and produce better results.

---

## 4. One-Shot Setup Script (RunPod)

Save this as `/workspace/setup.sh` and run it on every new pod with `bash /workspace/setup.sh`:

```bash
#!/bin/bash

# CRITICAL: Install cu130 PyTorch for RTX 5090 (Blackwell/sm_120)
pip install torch==2.12.1 torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130 --break-system-packages

# Install all dependencies
pip install av sqlalchemy alembic pydantic comfy-aimdo comfy-kitchen hf_transfer simpleeval accelerate imageio-ffmpeg gguf ftfy einops diffusers peft sentencepiece protobuf pyloudnorm opencv-python scipy --break-system-packages

# Update ComfyUI to latest
cd /ComfyUI && git fetch origin && git reset --hard origin/master

# Symlink custom nodes from persistent storage
ln -sf /workspace/custom_nodes/ComfyUI-VideoHelperSuite /ComfyUI/custom_nodes/ 2>/dev/null
ln -sf /workspace/custom_nodes/ComfyUI-GGUF /ComfyUI/custom_nodes/ 2>/dev/null

# Symlink models
rm -rf /ComfyUI/models/clip_vision && ln -sf /workspace/ComfyUI/models/clip_vision /ComfyUI/models/clip_vision
rm -rf /ComfyUI/models/diffusion_models && ln -sf /workspace/ComfyUI/models/diffusion_models /ComfyUI/models/diffusion_models
rm -rf /ComfyUI/models/vae && ln -sf /workspace/ComfyUI/models/vae /ComfyUI/models/vae
rm -rf /ComfyUI/models/text_encoders && ln -sf /workspace/ComfyUI/models/text_encoders /ComfyUI/models/text_encoders

# Symlink GGUF models to unet folder for UnetLoaderGGUF
mkdir -p /ComfyUI/models/unet
ln -sf /workspace/ComfyUI/models/diffusion_models/High/Index-Anisora-V3.2-High-Q8_0.gguf /ComfyUI/models/unet/ 2>/dev/null
ln -sf /workspace/ComfyUI/models/diffusion_models/Low/Index-Anisora-V3.2-Low-Q8_0.gguf /ComfyUI/models/unet/ 2>/dev/null

# Start ComfyUI
pkill -f "main.py" ; sleep 2 ; cd /ComfyUI && python main.py --listen 0.0.0.0 --port 3000 > /tmp/comfy.log 2>&1 &
echo "ComfyUI starting on port 3000..."
```

---

## 5. Working Workflow: Native Two-Pass I2V

### Node Structure

The working workflow uses **native ComfyUI nodes + UnetLoaderGGUF**. DO NOT use WanVideoWrapper nodes — they cause temporal coherence issues with these GGUF models.

```
UnetLoaderGGUF (High model)
UnetLoaderGGUF (Low model)
CLIPLoader (text encoder, type: wan)
CLIPTextEncode (positive prompt)
CLIPTextEncode (negative prompt)
VAELoader
CLIPVisionLoader
LoadImage (your input image)
CLIPVisionEncode
WanImageToVideo ← native node
KSamplerAdvanced (High: add_noise=enable, steps=30, start=0, end=22)
KSamplerAdvanced (Low: add_noise=disable, steps=30, start=22, end=30)
VAEDecode
VHS_VideoCombine (h264, yuv420p, 16fps)
```

The workflow JSON is included in this repo — just drag it into ComfyUI.

### Recommended Settings

| Setting | Value |
|---------|-------|
| Sampler | euler_ancestral |
| Scheduler | beta |
| CFG | 5.5 |
| Steps | 30 total (High: 0→22, Low: 22→30) |
| Resolution | 1280x720 |
| Frames | 49–60 |
| FPS | 16 |

---

## 6. Pitfalls — What NOT to Do

### PyTorch
- **DO NOT use PyTorch cu124 or cu128 on RTX 5090** — produces pure static noise with no error message. Must use cu130.

### Workflow Nodes
- DO NOT use `WanVideoSampler` (WanVideoWrapper) — causes temporal coherence breakdown
- DO NOT use `WanVideoModelLoader` with these GGUF files — use `UnetLoaderGGUF` instead
- DO NOT use `LoadWanVideoT5TextEncoder` with fp8 scaled text encoder — use `CLIPLoader` with type `wan`
- DO NOT connect `WanImageToVideo`'s clip_vision input directly from `CLIPVisionLoader` — must go through `CLIPVisionEncode` first

### Sampler Settings
- DO NOT use CFG above 7.0 — causes color explosion and artifacts
- DO NOT use dpmpp_2m/karras with Wan video models — causes artifacts
- DO NOT connect `denoised_samples` output to next sampler's `samples` input — must use `samples → samples`

### Two-Pass Setup
- Both `KSamplerAdvanced` nodes must have the **same total steps value**
- Second sampler (Low) must have `add_noise=disable`
- Both samplers should use the same seed (set to fixed, not randomize)

---

## 7. Known Limitations

- **Rubber band effect:** Videos animate then return toward the original pose. This is inherent to I2V diffusion. Fix: use shorter clips (33–49 frames) and trim the end, or use First/Last Frame workflow (FLF2V).
- **Generation time:** ~18–25 minutes at 720p, 60 frames, 30 steps on RTX 5090. Subsequent runs are faster (models stay in VRAM).
- **Model loading:** First generation after restart includes 5–7 min model loading time.

---

## 8. Prompt Tips

- Focus your prompt on **MOTION**, not on re-describing the image. The model already sees the image.
- ✅ Good: `"hair blowing gently in the wind, slight head movement, blinking"`
- ❌ Bad: `"anime girl with red hair wearing a blue jacket"` (model already knows this)
- Add to negative prompt: `"blurry motion, flickering, inconsistent lighting, static, watermark, text, artifacts"`
- Aim for 80–120 word prompts for best adherence

---

## Files in This Repo

- `AniSora_V3.2_GGUF_ComfyUI_Guide.md` — this guide
- `AniSora_MySettings.json` — working ComfyUI workflow (drag into ComfyUI)
- `setup.sh` — one-shot setup script for RunPod

---

*This guide was produced through two days of live debugging on RunPod with an RTX 5090. Shared freely — pay it forward.*
