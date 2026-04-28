# Mali-G31 / Mali-G52 Benchmarks

Measured performance figures for Mali-G31 MP1 (Solitude / SM1) and
Mali-G52 MP2 (Alta / G12B) using upstream Mesa Panfrost (Gallium) and
PanVK (Vulkan).

Snapshot date: 2026-04-27. All numbers below were measured on a
single fresh build of upstream Mesa main.

## Configuration

| Parameter | Value |
|-----------|-------|
| GPU (Solitude / SM1) | Mali-G31 MP1, Bifrost arch v7 |
| GPU (Alta / G12B) | Mali-G52 MP2 (2 EE), Bifrost arch v7 |
| Mesa | upstream main (>= 25.0); 26.x at snapshot date |
| Kernel | mainline 6.12.x with `panfrost` and `meson-drm` |
| Surface | offscreen pbuffer / FBO or headless Wayland (weston) |
| Color buffer | RGBA8888 D24 S0 (no MSAA) |
| CPU governor | `performance` (locked to max) |
| GPU governor | `performance` (locked to 800 MHz) |

## glmark2-es2 overall score

`glmark2-es2-wayland` against headless `weston`, single-run
aggregate. Run twice and take the second result; the first run
primes Mesa shader and pipeline caches.

| Board | SoC | GPU | Resolution | glmark2 score |
|-------|-----|-----|------------|---------------|
| Solitude (AML-S905D3-CC) | S905D3 | Mali-G31 MP1 | 1024x768 | 662 |
| Solitude (AML-S905D3-CC) | S905D3 | Mali-G31 MP1 | 1920x1080 | 347 |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | 1024x768 | 1419 |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | 1920x1080 | 823 |

Higher is better. Mali-G52 (Alta) delivers approximately 2.1-2.4x
the overall glmark2 score of Mali-G31 (Solitude) at the same
resolution -- reflecting both the 2-EE configuration of G52 and
better per-EE dual-issue.

## Per-test glmark2 throughput (FPS, 1080p, headless Wayland)

Selected GLES2 sub-tests at 1920x1080:

| Test | Mali-G52 (Alta) FPS |
|------|---------------------:|
| build:use-vbo=true (vertex-heavy) | 1326 |
| shading:shading=phong (phong shading) | 810 |
| jellyfish (texture/geometry) | 478 |
| terrain (demanding) | 15 |

The terrain scene is the hardest workload in glmark2; it stresses
fragment shader length and fill-rate together. The other scenes
demonstrate per-class throughput. SM1 G31 numbers track at
roughly 0.45-0.50x the G52 figures (consistent with single-EE
hardware).

## Vulkan smoke test (vkcube)

Headless Wayland + `vkcube --present_mode fifo`:

| Board | SoC | GPU | Result |
|-------|-----|-----|--------|
| Solitude (AML-S905D3-CC) | S905D3 | Mali-G31 MP1 | 60 fps vsync-locked, no errors |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | 60 fps vsync-locked, 120 frames rendered, no errors |

PanVK is gated behind the `MESA_VK_IGNORE_CONFORMANCE_WARNING=1`
environment variable for non-conformant warnings during ongoing
upstream development. The driver is upstream Mesa and reports
Vulkan 1.3 with 143 device extensions on both Mali-G31 and Mali-G52.

## Sustained-load thermal stability

60-second sustained `glmark2 build:use-vbo=true` load with passive
heatsink (no active cooling), ambient ~22 C:

| Board | GPU clock | Sustained 60 s | Temp before | Temp after | Throttle? |
|-------|-----------|----------------|-------------|------------|-----------|
| Solitude (S905D3) | 800 MHz | yes | ~63 C | ~67.6 C | no |
| Alta (A311D) | 800 MHz | yes | ~61.3 C | ~67.3 C | no |

Both boards run their full DVFS table (125-800 MHz, 7 OPPs) from a
single shared SoC voltage rail and held 800 MHz throughout the
60-second sustained load with no thermal throttling.

## dEQP-GLES2 conformance

| Board | Subset | Pass | Fail | NotSupp | Total | Pass% |
|-------|--------|-----:|-----:|--------:|------:|------:|
| Alta (G52, A311D) | dEQP-GLES2 functional | 16957 | 48 | 156 | 17165 | 98.8% |

Pass-rate among graded tests (PASS / (PASS+FAIL)): **99.7%**.
The 48 failures are split between shader-scoping edge cases and
texture-format minutiae documented upstream as known
non-conformances on Bifrost; no driver-stability failures.

The Solitude (G31) dEQP-GLES2 path requires a Wayland-headless EGL
context (the surfaceless pbuffer path returns ENOMEM from BO
allocation without a DRM master). Use the headless-weston
invocation in [api.md](api.md) to run the suite on G31.

## dEQP-VK conformance (selected groups)

PanVK on Mali-G31 (Solitude) and Mali-G52 (Alta), 6 representative
groups from upstream VK-GL-CTS. **Bit-for-bit identical results
across both Mali variants.**

| Group | Pass | Fail | NotSupp | Total |
|-------|-----:|-----:|--------:|------:|
| `dEQP-VK.api.smoke` | 6 | 0 | 0 | 6 |
| `dEQP-VK.descriptor_indexing` | 10 | 0 | 99 | 109 |
| `dEQP-VK.binding_model.descriptor_update` | 63 | 0 | 60 | 123 |
| `dEQP-VK.transform_feedback.simple.winding_*` | 56 | 0 | 24 | 80 |
| `dEQP-VK.pipeline.monolithic.image` | 17408 | 0 | 28424 | 45832 |
| `dEQP-VK.pipeline.monolithic.vertex_input` | 9111 | 0 | 1421 | 10532 |
| **All 6 groups (totals)** | **26654** | **0** | **30028** | **56682** |

Zero failures across all 6 groups on both Mali-G31 and Mali-G52.

The high `NotSupp` counts on `image` and `vertex_input` reflect
features that require Bifrost arch >= 9 (Valhall) or that are
explicit silicon limits on arch 7. The `Pass` columns represent the
features actually available on this hardware. NotSupp is a clean
"this combination requires a feature we don't expose" return, not
a test failure.

## Notes on benchmarking

- Always set CPU and GPU governors to `performance` before
  benchmarking. Leaving the default `schedutil` /
  `simple_ondemand` introduces ~35% variance at low draw counts.
- glmark2 `-s 1024x768` is a fixed budget; numbers are not
  directly comparable to other resolutions.
- `glmark2-es2-drm` does not work on headless boards because the
  meson-vpu KMS device and the panfrost render device are separate
  DRM cards. Use the Wayland-headless method shown in
  [api.md](api.md).
- Run each benchmark twice and take the second result. The first
  run primes Mesa shader and pipeline caches.
- Vulkan support is mature for the conformance groups listed above
  but PanVK driver development is ongoing in upstream Mesa; expect
  more groups to clear over time as the driver matures.
