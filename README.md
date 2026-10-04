# Short report: ComfyUI VRAM regression investigation

**System:** Windows, AMD RX 9060 XT 16 GB, ROCm, ComfyUI 0.37.0, PyTorch 2.13.0+rocm10.0.0, DynamicVRAM enabled, async weight offloading with 2 streams, pinned memory enabled.

## Symptom before

Repeated image generations degraded badly after the first run.

Typical pattern:

- first Krea2 run: ~1.7–2.0 s/it
- later runs: up to ~19–34 s/it
- shared/system memory increased
- GPU power/noise dropped despite high reported utilization
- `--disable-smart-memory` made repeated runs stable again

This pointed to a VRAM residency/offload problem rather than raw GPU compute failure.

## How we investigated

We checked:

- ComfyUI core files and Git state — no recent core modification
- Torch/ROCm versions and timestamps — unchanged
- environment variables and launch scripts — nothing suspicious
- custom-node monkeypatching of `model_management` / `ModelPatcher` — none found
- FlashVSR, ACE-Step, UltimateSDUpscale and dependency changes
- package imports and requirements
- static source analysis of custom nodes

The strongest source-level finding was **ComfyUI-N-Nodes**: its RIFE frame interpolation code creates a model at module import time and moves it to the GPU. Because N-Nodes imports its Python modules automatically, this GPU model can exist even when RIFE is not used in the current workflow.

## A/B test

We moved:

`custom_nodes\ComfyUI-N-Nodes`

out of `custom_nodes`, restarted ComfyUI, and repeated the same workloads.

With N-Nodes disabled:

- SDXL batch 12: **3.63 → 3.80 → 3.78 s/it** — stable across repeated runs
- Krea2 batch 1: **1.74 → 1.75 → 1.71 s/it** — stable
- Krea2 batch 2 exceeded the practical VRAM limit and slowed to **33.77 s/it**, as expected
- After returning to batch 1, performance immediately recovered to **1.73 → 1.74 s/it**
- Later SDXL and Krea2 runs remained stable as well

The startup log also confirmed that N-Nodes was no longer imported, while the other custom nodes remained active.

## Measurement

We correlated ComfyUI timing with HWMonitor telemetry.

Healthy runs showed normal high GPU power and full cooling activity. During the deliberate Krea2 batch-2 overcommit, VRAM/shared-memory pressure rose and GPU power dropped sharply, consistent with memory residency/offload thrashing rather than thermal throttling.

After returning to batch 1, both performance and GPU behavior recovered.

Temperatures remained normal, so thermal throttling was not indicated.

## Current conclusion

The strongest current suspect is **ComfyUI-N-Nodes / its import-time RIFE GPU model**.

The likely mechanism is not necessarily an unbounded leak. Instead, N-Nodes may reserve GPU memory outside ComfyUI's normal model-management system, reducing available VRAM enough that DynamicVRAM/smart memory crosses an offload threshold on heavier workflows.

The result is strong but not yet absolute proof.

The cleanest final confirmation would be to restore N-Nodes and repeat the same short A/B test.
