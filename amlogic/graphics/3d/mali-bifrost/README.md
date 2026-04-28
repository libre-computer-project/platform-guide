# Mali-G31 / Mali-G52 (Bifrost) 3D GPU

Open-source OpenGL ES 3.2, OpenGL 3.3, and Vulkan 1.3 acceleration on
Amlogic G12B and SM1 SoCs via the upstream Panfrost driver in Mesa.

## What it is

The Mali-G31 and Mali-G52 are tile-based deferred-rendering GPUs from
the ARM Bifrost architecture family. Both are integrated in the
Amlogic G12/SM1 12 nm process node and supported in mainline Linux
and upstream Mesa via the Panfrost Gallium driver and the PanVK
Vulkan driver -- no proprietary blob, no out-of-tree kernel module.

| Property | Mali-G31 (SM1) | Mali-G52 (G12B) |
|----------|----------------|-----------------|
| Architecture | ARM Bifrost (v7) | ARM Bifrost (v7) |
| Configuration | 1 execution core (MC1) | 2 execution cores (MC2) |
| Marketing name | Mali-G31 MP1 (2EE per core) | Mali-G52 MP2 (2EE per core) |
| Process | 12 nm | 12 nm |
| Default clock | 800 MHz (7-OPP DVFS) | 800 MHz (7-OPP DVFS) |
| OpenGL ES | 3.2 (GLSL ES 3.20) | 3.2 (GLSL ES 3.20) |
| OpenGL | 3.3 core + compatibility | 3.3 core + compatibility |
| Vulkan | 1.3 | 1.3 |
| User-space stack | Mesa Panfrost (GL/GLES) + PanVK (Vulkan), EGL, GBM | Mesa Panfrost (GL/GLES) + PanVK (Vulkan), EGL, GBM |
| Kernel driver | `panfrost` (mainline, `drivers/gpu/drm/panfrost/`) | `panfrost` (mainline) |

## Supported boards

| Board | Product name | SoC | GPU | RAM |
|-------|--------------|-----|-----|-----|
| AML-A311D-CC | Alta | A311D | Mali-G52 MP2 | 4 GB |
| AML-A311D-CM | Alta CM | A311D | Mali-G52 MP2 | 4 GB |
| AML-S905D3-CC | Solitude | S905D3 | Mali-G31 MP1 | 2 / 4 GB |
| AML-S905D3-CM | Solitude CM | S905D3 | Mali-G31 MP1 | 2 / 4 GB |

Mali-G52 also exists on the S922X variant of G12B; Mali-G31 also
exists on S905X3 / S805X2 variants of SM1. The list above covers
shipping Libre Computer boards.

## What it can do

### Standard graphics APIs

- **OpenGL ES 2.0 / 3.0 / 3.1 / 3.2** with GLSL ES 1.00 / 3.00 /
  3.10 / 3.20.
- **OpenGL 2.1 / 3.0 / 3.1 / 3.2 / 3.3** core and compatibility
  profiles via Mesa `st/mesa`.
- **Vulkan 1.3** via PanVK (143 device extensions).

### Compute and feature highlights

- **Compute shaders** (GLES 3.1, OpenGL 4.3, Vulkan compute).
- **Geometry shaders** -- emulated via NIR compute lowering pass
  (Bifrost has no native GS hardware; the geometry-shader stage runs
  as a series of compute dispatches that produce the same observable
  results).
- **Transform feedback** with primitive-decomposed capture for line,
  triangle, and adjacency topologies (matches OpenGL / Vulkan spec
  semantics).
- **Multi-sample anti-aliasing** up to 4x.
- **Tessellation** is not exposed (Bifrost silicon does not include
  the tessellation unit).
- **Hardware texture decode** for ASTC (LDR + HDR), ETC2, BC1-7
  (where the kernel exposes the format), R10G10B10A2.
- **AFBC** (ARM Frame Buffer Compression) for textures and render
  targets.
- **FP16 in compiler** (full 16-bit shader path).
- **Int64** (full 64-bit integer support).
- **Pixel local storage** -- 16 bytes per fragment (Bifrost fast
  path).
- **Sample shading** (per-sample fragment shading for MSAA).
- **Anisotropic filtering** (silicon revision-dependent).
- **Hardware shader clock** (`GL_ARB_shader_clock`) when the kernel
  exposes timestamps.
- **Hardware draw-indirect** for `glDrawElementsIndirect` and
  `glDrawArraysIndirect`.

### Off-screen / headless rendering

- EGL surfaceless platform.
- GBM via `/dev/dri/renderD128`.
- Wayland WSI through any Wayland compositor (weston, sway,
  Mutter, etc).
- X11 WSI for legacy X applications.

## Use cases

The Mali-G31 / G52 family is well-matched to modern 3D and compute
workloads:

- **Wayland / Vulkan-capable desktops** -- KDE Plasma, GNOME, sway
  with Vulkan-accelerated WSI; modern toolkits (GTK4, Qt 6) target
  GLES 3.x or Vulkan.
- **3D game engines** -- Godot 4 (GLES 3 and Vulkan render backends),
  Unreal Engine 4 GLES 3.1 path, Unity GLES 3.2 / Vulkan path on
  Linux ARM64.
- **Compute workloads** -- GLES 3.1 compute shaders for image
  processing, neural inference (TFLite GPU delegate where available),
  particle simulation.
- **Browser GPU acceleration** -- Chromium / Firefox WebGL2 via
  surfaceless EGL or Wayland.
- **Embedded HMI / dashboards** -- Qt Quick, LVGL with GPU backend,
  Slint, Flutter Linux embedder.
- **Headless render pipelines** -- automated visualization, server-
  side rendering, CI image generation via EGL surfaceless.
- **Video texture composition** -- 4K video decoded by the SoC's
  video decoder (see `amlogic/video/decoder/`) imported as a
  GL/Vulkan texture, composited with overlays at 60 fps.

The G52 (Alta) is approximately 2x the per-core throughput of the
G31 (Solitude) due to 2 execution cores vs 1, plus better per-EE
dual-issue. See [benchmarks.md](benchmarks.md) for measured numbers.

## See also

- [api.md](api.md) -- device nodes, EGL platforms, environment, code patterns
- [benchmarks.md](benchmarks.md) -- glmark2, shader throughput, dEQP per board and clock
