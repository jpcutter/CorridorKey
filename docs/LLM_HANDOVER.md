# CorridorKey: LLM Handover Guide

Technical reference for AI coding assistants working on the CorridorKey codebase. This is the primary reference document — read this when you need project context.

For full API signatures, see [API_REFERENCE.md](API_REFERENCE.md). For architecture diagrams and model internals, see [ARCHITECTURE.md](ARCHITECTURE.md). For color math details, see [COLOR_PIPELINE.md](COLOR_PIPELINE.md).

---

## 1. Quick Reference

| Item | Detail |
|---|---|
| Language | Python 3.11+ |
| Framework | PyTorch 2.9.1, Timm 1.0.24, Diffusers, OpenCV |
| CLI entry point | `corridorkey_cli.py` (`corridorkey` console script) |
| Pipeline library | `clip_manager.py` (importable, no CLI) |
| Core engine | `CorridorKeyModule/inference_engine.py` |
| Model arch | `CorridorKeyModule/core/model_transformer.py` |
| Color math | `CorridorKeyModule/core/color_utils.py` |
| Alpha generators | `gvm_core/` (GVM) and `VideoMaMaInferenceModule/` (VideoMaMa) |
| Checkpoint | `CorridorKeyModule/checkpoints/CorridorKey.pth` (~300MB) |
| Native resolution | 2048x2048 (bilinear resize in, Lanczos4 resize out) |
| Min VRAM | ~22.7 GB (CorridorKey alone) |
| Package manager | [uv](https://docs.astral.sh/uv/) (`pyproject.toml` + `uv.lock`) |
| Tests | `tests/` (pytest suite: color math, clip manager, inference engine) + `test_vram.py` (GPU VRAM benchmark) |
| Linter | [ruff](https://docs.astral.sh/ruff/) (configured in `pyproject.toml`, excludes third-party modules) |
| CI/CD | None |

### Target Audience

Two distinct user groups — route them appropriately:
- **VFX Artists / Editors** → `docs/QUICKSTART_ARTISTS.md`
- **Developers / ML Engineers** → `docs/GETTING_STARTED.md` and `docs/API_REFERENCE.md`

---

## 2. Directory Structure

```
CorridorKey/
├── corridorkey_cli.py                 # CLI entry point — wizard, arg parsing, console script
├── clip_manager.py                    # Pipeline library — ClipEntry, inference, alpha gen
├── test_vram.py                       # VRAM benchmark utility (manual, GPU-only)
├── pyproject.toml                     # Dependencies and project metadata (uv)
│
├── CorridorKey_DRAG_CLIPS_HERE_local.sh   # Linux/macOS launcher
├── CorridorKey_DRAG_CLIPS_HERE_local.bat  # Windows launcher
├── Install_CorridorKey_Windows.bat        # Windows auto-installer
├── Install_GVM_Windows.bat                # GVM module installer
├── Install_VideoMaMa_Windows.bat          # VideoMaMa module installer
├── RunGVMOnly.sh                          # Alpha generation only
├── RunInferenceOnly.sh                    # Inference only
│
├── tests/                             # Pytest test suite (no GPU required)
│   ├── conftest.py                    # Shared fixtures (sample frames, mock model, tmp dirs)
│   ├── test_imports.py                # Smoke tests — all packages import cleanly
│   ├── test_color_utils.py            # sRGB/Linear, premul, despill, clean_matte, compositing
│   ├── test_gamma_consistency.py      # Documents the gamma 2.2 vs piecewise sRGB inconsistency
│   ├── test_clip_manager.py           # File detection, path mapping, ClipEntry discovery
│   └── test_inference_engine.py       # process_frame pipeline (mocked model, no GPU)
│
├── CorridorKeyModule/                 # Core engine
│   ├── __init__.py                    # Exports CorridorKeyEngine
│   ├── inference_engine.py            # CorridorKeyEngine class
│   ├── README.md
│   └── core/
│       ├── model_transformer.py       # GreenFormer architecture
│       └── color_utils.py             # Compositing math
│
├── gvm_core/                          # GVM alpha hint generator
│   ├── __init__.py                    # Exports GVMProcessor
│   ├── wrapper.py                     # GVMProcessor class
│   ├── LICENSE.md                     # CC BY-NC-SA 4.0
│   ├── README.md
│   └── gvm/                           # Internal diffusion package
│       ├── models/unet_spatio_temporal_condition.py
│       ├── pipelines/pipeline_gvm.py
│       └── utils/inference_utils.py
│
├── VideoMaMaInferenceModule/          # VideoMaMa alpha hint generator
│   ├── __init__.py                    # Exports load_videomama_model, etc.
│   ├── inference.py                   # Inference API
│   ├── pipeline.py                    # SVD-based pipeline
│   ├── LICENSE.md                     # CC BY-NC 4.0 + Stability AI
│   └── README.md
│
├── ClipsForInference/                 # User input staging area
├── Output/                            # Output destination
├── IgnoredClips/                      # Excluded clips
└── docs/                              # Documentation
```

---

## 3. Key API Entry Points

For full signatures and return types, see [API_REFERENCE.md](API_REFERENCE.md).

### `CorridorKeyEngine` (`CorridorKeyModule/inference_engine.py`)
- `__init__(checkpoint_path, device='cuda', img_size=2048, use_refiner=True)`
- `process_frame(image, mask_linear, refiner_scale=1.0, input_is_linear=False, fg_is_straight=True, despill_strength=1.0, auto_despeckle=True, despeckle_size=400)` → dict with keys: `alpha`, `fg`, `comp`, `processed`

### `color_utils` (`CorridorKeyModule/core/color_utils.py`)
- `srgb_to_linear(x)` / `linear_to_srgb(x)` — piecewise sRGB transfer, supports NumPy and PyTorch
- `premultiply(fg, alpha)` / `unpremultiply(fg, alpha)`
- `composite_straight(fg, bg, alpha)` / `composite_premul(fg, bg, alpha)`
- `despill(image, green_limit_mode='average', strength=1.0)` — luminance-preserving
- `clean_matte(alpha_np, area_threshold, dilation, blur_size)` — morphological cleanup
- `dilate_mask(mask, radius)` — supports NumPy (cv2) and PyTorch (MaxPool)

### `corridorkey_cli` (`corridorkey_cli.py`)
- `main()` — argparse CLI entry point, registered as `corridorkey` console script
- `interactive_wizard(win_path)` — the interactive wizard loop

### `GVMProcessor` (`gvm_core/wrapper.py`)
- `__init__(model_base=None, unet_base=None, lora_base=None, device="cuda", seed=None)`
- `process_sequence(input_path, output_dir, num_frames_per_batch=8, denoise_steps=1, ...)`

### `VideoMaMaInferenceModule` (`VideoMaMaInferenceModule/inference.py`)
- `load_videomama_model(base_model_path=None, unet_checkpoint_path=None, device="cuda")`
- `run_inference(pipeline, input_frames, mask_frames, chunk_size=24)` — generator yielding frame chunks

---

## 4. Forward Pass Detail

**File:** `CorridorKeyModule/core/model_transformer.py`

```python
# Input: x [B, 4, 2048, 2048]
features = encoder(x)                    # 4 multiscale feature maps
alpha_logits = alpha_decoder(features)   # [B, 1, H/4, W/4]
fg_logits = fg_decoder(features)         # [B, 3, H/4, W/4]

# Upsample to full resolution
alpha_logits_up = interpolate(alpha_logits, input_size)
fg_logits_up = interpolate(fg_logits, input_size)

# Coarse probabilities for refiner input
alpha_coarse = sigmoid(alpha_logits_up)
fg_coarse = sigmoid(fg_logits_up)

# Refiner predicts delta logits from [RGB, alpha_coarse, fg_coarse]
delta = refiner(rgb, cat(alpha_coarse, fg_coarse))  # [B, 4, H, W] × 10.0

# Residual addition in logit space + final activation
alpha_final = sigmoid(alpha_logits_up + delta[:, 0:1])
fg_final = sigmoid(fg_logits_up + delta[:, 1:4])
```

Key detail: the refiner's output is scaled by `10×` and its weights are whisper-initialized (`Normal(0, 1e-3)` + zero bias), so it starts as near-identity and learns corrections.

---

## 5. Alpha Hint Generators

### GVM (`gvm_core/`)

- **Method:** Stable Video Diffusion with LoRA, FlowMatch scheduler (1-step denoising)
- **Input:** RGB video/sequence only — fully automatic
- **Processing:** Resize to 1024 short-edge (max 1920 long-edge), pad to multiples of 32, batch inference
- **Output post-processing:** Threshold ≥240/255 → 1.0, ≤25/255 → 0.0, resize to input resolution
- **VRAM:** ~80GB
- **Key class:** `GVMProcessor` in `gvm_core/wrapper.py`

### VideoMaMa (`VideoMaMaInferenceModule/`)

- **Method:** Masked Stable Video Diffusion with CLIP conditioning
- **Input:** RGB frames + binary `VideoMamaMaskHint` (thresholded at value 10)
- **Processing:** Resize to 1024×576 (SVD standard), chunk-based inference (default 24 frames)
- **VRAM:** ~24GB+ (FP16)
- **Key function:** `run_inference()` in `VideoMaMaInferenceModule/inference.py` — generator yielding frame chunks

---

## 6. CLI / Library Split

The CLI and pipeline logic are separated into two files:

- **`corridorkey_cli.py`** — CLI entry point. Contains `main()` (argparse), `interactive_wizard()`, and environment configuration. Registered as the `corridorkey` console script in `pyproject.toml`. All launcher scripts (`.sh`, `.bat`) invoke this file.
- **`clip_manager.py`** — Pipeline library. Contains `ClipEntry`, `ClipAsset`, inference orchestration, alpha generation, and file I/O. Importable without side effects (no `__main__` block).

---

## 7. Clip Manager Data Model and Wizard Flow

**File:** `clip_manager.py`

### Key Classes

- **`ClipAsset`** — Wraps a media source (directory of images or video file). Properties: `path`, `type` (`'sequence'`/`'video'`), `frame_count`
- **`ClipEntry`** — Represents a complete shot. Properties: `name`, `root_path`, `input_asset`, `alpha_asset`. Methods: `find_assets()` (discovers Input/AlphaHint with fallback heuristics), `validate_pair()` (ensures frame counts match)

### Wizard Flow (`--action wizard`)

1. **Map Path** — Accepts Windows `V:\` paths and converts to Linux `/mnt/ssd-storage/`
2. **Analyze** — Detects if target is a single shot or batch of shots
3. **Organize** — Creates `Input/`, `AlphaHint/`, `VideoMamaMaskHint/` folder structure
4. **Status Loop** — Categorizes clips as READY / MASKED / RAW
5. **Actions** — `[v]` VideoMaMa, `[g]` GVM, `[i]` Inference, `[r]` Re-Scan, `[q]` Quit

### Expected Shot Folder Structure

```
MyShot/
├── Input/              # RGB frames (PNG, EXR, etc.) or Input.mp4
├── AlphaHint/          # Coarse alpha masks (generated or manual)
├── VideoMamaMaskHint/  # Rough binary mask for VideoMaMa (optional)
└── Output/             # Created by inference
    ├── FG/             # Straight foreground color (EXR, sRGB gamut)
    ├── Matte/          # Linear alpha channel (EXR)
    ├── Processed/      # Premultiplied RGBA (EXR, Linear)
    └── Comp/           # Preview composite over checkerboard (PNG)
```

### User-Configurable Settings (prompted at inference time)

| Setting | Default | Range | Notes |
|---|---|---|---|
| Gamma Space | sRGB | Linear or sRGB | Tells engine how to handle input gamma |
| Despill Strength | 10 (max) | 0-10 | Mapped to 0.0-1.0 internally |
| Auto-Despeckle | ON | ON/OFF | Connected-components cleanup |
| Despeckle Size | 400px | 0+ | Min pixel area to keep |
| Refiner Strength | 1.0 | float | Experimental, scales refiner output |

---

## 8. Debugging Checklist

When a user reports a visual artifact, check in this order:

1. **Dark fringes / crushed shadows?** → sRGB↔Linear conversion order error in `inference_engine.py` or downstream comp
2. **Blocky artifacts?** → Refiner not active (`use_refiner=False`) or refiner scale too low
3. **Green edges?** → Despill strength too low or applied in wrong color space
4. **Tracking markers in output?** → Auto-despeckle disabled or `despeckle_size` too small
5. **Mismatched frame counts?** → Input and AlphaHint sequences have different lengths
6. **OOM error?** → VRAM < 22.7GB, or running on GPU driving displays
7. **Black output?** → Checkpoint not found or wrong path; check `CorridorKeyModule/checkpoints/`
8. **Washed out / too bright?** → Double gamma correction (linear treated as sRGB or vice versa)

---

## 9. Directives for AI Assistants

1. **Color math is sacred.** Every compositing operation must respect the sRGB piecewise transfer functions and the straight/premultiplied distinction. When in doubt, trace the data through `color_utils.py`.
2. **Performance matters.** This processes 4K video frame-by-frame. Every `.numpy()` transfer, `cv2.resize`, or unnecessary copy matters in the hot loop.
3. **The model is fixed.** Training code is not in this repository. Do not modify `model_transformer.py` architecture unless explicitly building a new training pipeline.
4. **Run `uv run pytest`** after any changes to verify the test suite still passes (no GPU required). Run `uv run python test_vram.py` after engine changes to verify VRAM hasn't regressed (requires GPU).
5. **Respect licenses.** CorridorKey allows commercial *use* but not resale. GVM and VideoMaMa are strictly non-commercial.
6. **Preserve output compatibility.** Don't break the EXR output format that downstream Nuke/Fusion/Resolve workflows depend on.

---

## 10. Code Artifacts You May Encounter

- **Commented-out logit clamping** in `model_transformer.py` — An earlier "Humility Clamp" ([-3, 3]) was removed to preserve backbone detail. The commented code can be ignored.
- **`_orig_mod.` prefix stripping** in checkpoint loading — Handles models saved after `torch.compile()`. This is intentional and must stay.
- **Training code** is not in this repository. It may be released separately if community demand is sufficient.
