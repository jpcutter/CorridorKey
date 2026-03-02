# GVM Core Module

This folder contains the core logic and pre-trained models for Generative Video Matting (GVM).
It is designed to be a self-contained, portable module that can be dropped into any Python project.

## Directory Structure

```
gvm_core/
├── __init__.py           # Exports the main GVMProcessor class
├── wrapper.py            # High-level API for inference
├── gvm/                  # The core library package
│   ├── models/           # Spatio-temporal UNet definitions
│   ├── pipelines/        # Diffusers-based pipeline logic
│   └── utils/            # Video IO and processing utilities
└── weights/              # Bundled Model Weights (Autoencoder, UNet)
```

## Installation

When used as part of the CorridorKey project, all dependencies are managed by [uv](https://docs.astral.sh/uv/) via the root `pyproject.toml`. Run `uv sync` from the project root.

For standalone use, ensure you have Python 3.10+ and the required packages (PyTorch, Diffusers, Transformers, OpenCV, etc.).

## Usage

You can use the `GVMProcessor` to run inference on videos or image sequences.
The processor automatically finds the bundled model weights in the `weights/` directory.

### Basic Example

```python
from gvm_core import GVMProcessor

# Initialize the processor
# It will load models from ./weights automatically
processor = GVMProcessor(device="cuda")

# Process a video file
processor.process_sequence(
    input_path="path/to/input_video.mp4",
    output_dir="path/to/output_folder",
    num_frames_per_batch=8,   # Adjust based on VRAM (try 4 if OOM)
    denoise_steps=1           # 1-step inference is standard for this model
)
```

### Advanced Usage

You can customize the inference parameters:

```python
processor.process_sequence(
    input_path="path/to/sequence_folder", # Can also be a folder of images
    output_dir="output",
    num_frames_per_batch=8,
    decode_chunk_size=4,      # Reduces VRAM usage during decoding
    num_overlap_frames=1,     # Overlap between batches for temporal consistency
    mode='matte'              # 'matte' is the default mode
)
```

## Troubleshooting

- **Out of Memory (OOM)**: Reduce `num_frames_per_batch` (e.g., to 4 or 2) and `decode_chunk_size`.
- **Missing Weights**: Ensure the `weights/` folder exists inside `gvm_core/`. If you moved the code without the weights, you must download them or copy them separately.
