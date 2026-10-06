# Motion Scope

Real-time motion magnification in the browser. Reads a webcam or a
DSLR-as-webcam feed and exaggerates subtle motion live, so small movements — a
swaying branch, a pulse, machine vibration, insect wingbeats — become visible.

A single HTML file with no build step. The Neural method also loads ONNX
models from the `models/` folder next to it. Download the full zip from
[Releases](https://github.com/Theory-box/motion-scope/releases/latest).

## Requirements

- A modern browser with camera access (`getUserMedia`).
- Served over `https://` or opened as an artifact. `file://` blocks the camera
  in Chrome — use a local server (`python -m http.server`) if opening directly.
- A tripod. The method assumes a fixed camera; any shake is amplified too.

## Run

1. Open `motion_scope.html` in a browser (or as an artifact).
2. Pick a source, press **Start**, grant camera permission (retry once if the
   first attempt is blocked while the prompt is still up).

## Methods

Pick a method from the icon rail on the left of the video (two-letter labels).
The control panel shows only the settings that apply to the current method.

- **Linear** (`LI`, default) — amplifies per-pixel brightness change. Simple and
  responsive, but amplifies brightness noise along with the motion.
- **Phase** (`PH`, beta) — a single-scale Riesz method that amplifies local
  *displacement* rather than brightness, so brightness noise is largely left
  alone. Includes amplitude-weighted smoothing, which only shifts pixels sitting
  on real structure and ignores flat/noisy regions. Heavier than Linear; drop
  **Detail** if the frame rate falls. Favours fine-detail motion.
- **Riesz** (`RI`, beta) — the full multi-scale Riesz-pyramid method (Wadhwa et
  al. 2014): Laplacian pyramid, quaternionic phase per level, temporal
  band-pass, amplitude-weighted smoothing, then phase shift and collapse.
  Handles motion at every scale, not just fine detail. Runs on the GPU with a
  CPU fallback.
- **Neural** (`AI`, beta) — learned motion magnification running in the browser
  through ONNX Runtime Web (WebGPU, WASM fallback). Choose between **MagNet**
  (instant, any resolution), **STB-VMM 128 / 256** (Swin Transformer — sharper,
  less noise) and **theta-Net** (small model, native 1280). **Sharp** mode puts
  the magnified motion back onto the full-resolution frame so low-res models
  still look crisp. Needs the `models/` folder next to the HTML. Slow live, so
  best combined with **Process video** (offline render to WebM).
- **Warp** (`WA`, beta) — Lagrangian magnification: measures a motion field
  against a slowly updated rest frame and physically pushes pixels along the
  amplified motion. Good for large, visible movement; an **Overlay** slider
  layers Linear-style brightness amplification on top.
- **Isolate** (`IS`) — learns the still background and shows only what differs
  from it, on black. Good for spotting where something is moving rather than how;
  slow-drifting things (smoke) gradually fade into the background.
- **Accumulate** (`AC`) — not magnification: stacks frames from a static camera
  to denoise and brighten dark or noisy scenes. The **Stack** tab averages
  frames (with auto-brighten, freeze and Save PNG); the **Super-res** tab uses
  tiny sub-pixel shifts between frames to build a 2×/3×/4× higher-resolution
  image (drizzle). Hot pixels and fixed-pattern noise don't average out.

Each magnification method has two views: **Amplified** (`A`, motion over the
scene) and **Motion only** (`M`, the bare motion signal).

## Controls

| Control | Meaning |
| --- | --- |
| **Method** | Linear, Phase, Riesz, Neural, Warp, Isolate or Accumulate — see [Methods](#methods) |
| **Amplified / Motion only** | Overlay boosted motion on the image, or show just the motion field |
| **Amplification** | Gain applied to the motion signal |
| **Low / High cutoff** | Temporal band (Hz) that gets amplified — sway ≈ low, wingbeats ≈ high |
| **Temporal denoise** | Motion-adaptive recursive averaging on the feed *before* magnifying: de-noises static regions, passes moving ones through |
| **Spatial scale** | Amplifies a coarser pyramid level (block size 2^level). Higher = less fine noise, only larger motion |
| **Smoothing** | Fine spatial blur of the motion signal (also the amplitude-blur radius in Phase mode) |
| **Color** | Linear only: 0% drives motion from brightness (no colour speckle), 100% amplifies each RGB channel |
| **Detail** | Processing resolution. Lower = smoother, less noise, faster |
| **Reset baseline** | Re-seed the temporal filters after the scene settles |

## Noise handling

The three noise controls attack different sources:

- **Temporal denoise** is the biggest win for a tripod scene. It keeps a running
  per-pixel estimate of the "true" value and averages toward it where the pixel
  is still, but hands the raw value straight through where motion exceeds a
  threshold — so the static background de-noises without smearing the movers.
  The denoised frame feeds both the base image and the magnifier.
- **Spatial scale** amplifies coarse spatial detail, where signal-to-noise is
  higher; fine-grained sensor noise lives at the finest scale and is dropped.
- **Color** removes the coloured confetti in Linear mode by driving motion from
  luminance so channels move together.
- **Phase mode** is itself a noise strategy: amplifying displacement instead of
  brightness means brightness noise isn't amplified linearly.

Most remaining noise originates in the camera's live feed (lower-res, compressed,
reduced in-camera NR, high ISO in dim scenes). Lower ISO / more light beats any
in-app fix.

## Status

Linear, Isolate and Accumulate: working. Phase, Riesz, Neural and Warp: beta.
See [CHANGELOG.md](CHANGELOG.md) for the full history.
