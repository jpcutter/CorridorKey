# CLAUDE.md - CorridorKey

Neural-network green screen removal for VFX. Predicts true un-multiplied foreground color + linear alpha for every pixel — physically accurate color separation, not a binary mask.

## Key Files

| What | Where |
|---|---|
| CLI | `corridorkey_cli.py` (also: `uv run corridorkey`) |
| Pipeline library | `clip_manager.py` |
| Core engine | `CorridorKeyModule/inference_engine.py` |
| Color math | `CorridorKeyModule/core/color_utils.py` |
| Tests | `uv run pytest` (no GPU needed) |

## The One Rule

Use piecewise sRGB transfer functions from `color_utils.py` — never `x ** 2.2`. Track sRGB vs Linear through every operation. See [Color Pipeline](docs/COLOR_PIPELINE.md) for the full invariant checklist.

## Documentation

Read these on-demand — don't pre-load everything:

- **[LLM Handover](docs/LLM_HANDOVER.md)** — quick reference table, directory structure, API signatures, debugging checklist, AI directives (start here)
- **[Architecture](docs/ARCHITECTURE.md)** — GreenFormer model, inference pipeline, data flow, design principles
- **[API Reference](docs/API_REFERENCE.md)** — full Python API for all three modules
- **[Color Pipeline](docs/COLOR_PIPELINE.md)** — critical invariants, sRGB/Linear math, output specs
- **[Troubleshooting](docs/TROUBLESHOOTING.md)** — common issues and fixes
- **[Contributing](docs/CONTRIBUTING.md)** — dev setup, conventions, code review checklist
- **[Artist Quickstart](docs/QUICKSTART_ARTISTS.md)** — for non-developer users
- **[Getting Started](docs/GETTING_STARTED.md)** — developer setup, CLI commands, compositor integration
