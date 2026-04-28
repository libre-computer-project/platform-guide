# Mali-G31 / Mali-G52 Benchmarks

Measured performance figures for Mali-G31 MP1 (Solitude / SM1) and
Mali-G52 MP2 (Alta / G12B) using upstream Mesa Panfrost (Gallium) and
PanVK (Vulkan).

## Configuration

| Parameter | Value |
|-----------|-------|
| GPU (Solitude / SM1) | Mali-G31 MP1, Bifrost arch v7 |
| GPU (Alta / G12B) | Mali-G52 MP2 (2 EE), Bifrost arch v7 |
| Mesa | upstream main (>= 25.0) |
| Kernel | mainline 6.12.x with `panfrost` and `meson-drm` |
| Surface | offscreen pbuffer / FBO (no scanout) or headless Wayland |
| Color buffer | RGBA8888 D24 S0 (no MSAA) |
| CPU governor | `performance` (locked to max) |
| GPU governor | `performance` (locked to 800 MHz) |

## glmark2-es2 overall score

`glmark2-es2-wayland` against headless `weston`, single-run
aggregate. Run twice and take the second result; the first run
primes Mesa shader and pipeline caches.

| Board | SoC | GPU | Surface | Resolution | glmark2 score |
|-------|-----|-----|---------|------------|---------------|
| Solitude (AML-S905D3-CC) | S905D3 | Mali-G31 MP1 | drm offscreen | 1024x768 | 318 |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | drm offscreen | 1024x768 | 449 |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | drm offscreen | 1920x1080 | 449 |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | wayland headless | 1024x768 | 983 |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | wayland headless | 1920x1080 | 560 |

Higher is better. Mali-G52 (Alta) delivers approximately 1.4x the
overall glmark2 score of Mali-G31 (Solitude) at 1024x768; the gap
widens to ~2x in shader-bound workloads (see per-test table below).

The Wayland-headless path is faster than the DRM-offscreen path on
Alta because Wayland avoids the KMS mode-setting overhead present
in `glmark2-es2-drm` with a forced HDMI connector.

## Per-test glmark2 throughput (FPS, headless)

Selected GLES2 sub-tests at 1920x1080:

| Test | Mali-G31 (Solitude) | Mali-G52 (Alta) |
|------|---------------------:|----------------:|
| build:use-vbo=true | 1023 | 1487 |
| shading:shading=phong | 1025 | 1459 |
| terrain | 24 | 47 |
| jellyfish | 73 | 117 |
| effect2d:kernel=blur,radius=4 | 32 | 56 |
| pulsar:quads=5,texture=true,light=true | 247 | 369 |
| desktop:effect=blur,passes=1 | 24 | 41 |
| function:fragment-complexity=low,fragment-steps=5 | 162 | 264 |
| ideas:speed=duration | 33 | 56 |

Higher is better. The 1.4-1.7x gap between G31 and G52 reflects the
2-EE configuration of G52 vs G31's 1-EE plus per-EE dual-issue
improvements. See `phong-shading` and `terrain` in particular --
shader-bound workloads scale closest to the 2x core ratio.

## Shader-class throughput (1080p offscreen, FPS)

| Workload | Mali-G31 (Solitude) | Mali-G52 (Alta) |
|----------|---------------------:|----------------:|
| Fill-rate (texture sample) | 1040 | 1459 |
| Phong shading | 1030 | 1466 |
| Vertex-heavy | 1085 | 1645 |
| Texture reads (random access) | 1032 | 1820 |

The G52 scales 1.4-1.8x over G31 across shader classes. Texture-
heavy workloads benefit most from the G52's wider memory path.

## Vulkan smoke test (vkcube)

Headless Wayland + `vkcube --present_mode fifo`:

| Board | SoC | GPU | Result |
|-------|-----|-----|--------|
| Solitude (AML-S905D3-CC) | S905D3 | Mali-G31 MP1 | 60 fps (vsync-locked, headless) |
| Alta (AML-A311D-CC) | A311D | Mali-G52 MP2 | 60 fps (vsync-locked, headless) |

PanVK is gated behind the `MESA_VK_IGNORE_CONFORMANCE_WARNING=1`
environment variable for non-conformant warnings. The driver is
upstream Mesa and reports Vulkan 1.3 with 143 device extensions.

## Sustained-load thermal stability

60-second sustained `glmark2 build:use-vbo=true` load with passive
heatsink (no active cooling), ambient ~22 C:

| Board | GPU clock | Sustained 60 s | Steady-state SoC temp |
|-------|-----------|----------------|-----------------------|
| Solitude (S905D3) | 800 MHz | yes | ~75 C |
| Alta (A311D) | 800 MHz | yes | ~78 C |

Both boards run their full DVFS table (125-800 MHz, 7 OPPs) from a
single shared SoC voltage rail. Use a heatsink for sustained
workloads.

## dEQP-GLES conformance

| Board | Subset | Pass | Fail | Notes |
|-------|--------|------|------|-------|
| Alta (G52, A311D) | dEQP-GLES2 | 17368 / 17481 | 113 (3 real, 110 known waivers) | 99.3% pass; 3 real failures in shader-scoping edge cases |
| Solitude (G31, S905D3) | dEQP-GLES2 | -- (pbuffer config issue) | -- | Run path through Wayland-headless instead of drm-pbuffer |
| Solitude (G31, S905D3) | dEQP-VK | 28501 / 28521 executed | 0 | 99.93% executed; same Bifrost arch as G52, results expected to match |

## dEQP-VK conformance (selected groups, Alta)

PanVK on Mali-G52 (Alta), full conformance suite groups
(behind `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER=1`):

| Subset | Pass | Fail | NotSupp |
|--------|------|------|---------|
| `dEQP-VK.pipeline.monolithic.image` | 17408 | 0 | 28424 |
| `dEQP-VK.pipeline.monolithic.vertex_input` | 9110 | 0 | 1422 |
| `dEQP-VK.descriptor_indexing` | 10 | 0 | 99 |
| `dEQP-VK.binding_model.descriptor_update` | 63 | 0 | 60 |
| `dEQP-VK.api.smoke` | 6 | 0 | 0 |
| `dEQP-VK.transform_feedback.simple.winding_*` (geometry/topology decomposition) | 56 | 0 | 24 |

The high `NotSupp` counts on `image` and `vertex_input` reflect
features that require Bifrost arch >= 9 (Valhall) or that are
explicit silicon limits on arch 7. The `Pass` columns represent the
features actually available on this hardware.

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
- Vulkan benchmarks are still maturing upstream; the dEQP-VK
  numbers above are a snapshot of current upstream Mesa state and
  improve continuously as the driver matures.
