# Mali-450 (Utgard) 3D GPU

Open-source OpenGL ES 2.0 acceleration on Amlogic GXL-family SoCs via the
upstream Lima driver in Mesa.

## What it is

The Mali-450 MP3 is a tile-based deferred-rendering 3D GPU integrated in
the Amlogic GXL family (28 nm process). It implements the ARM Utgard
architecture and is supported in mainline Linux and upstream Mesa via
the Lima Gallium driver -- no proprietary blob, no out-of-tree kernel
module.

| Property | Value |
|----------|-------|
| Architecture | ARM Utgard (Mali Utgard family) |
| Configuration | 1 Geometry Processor (GP) + 3 Pixel Processors (PP) |
| Marketing name | Mali-450 MP3 |
| Process | 28 nm |
| Default clock | 666 MHz (S905X), 500 MHz (S805X stock DVFS top) |
| Max validated clock | 1584 MHz (overclocked, see [benchmarks.md](benchmarks.md)) |
| API | OpenGL ES 2.0, OpenGL 2.1 (compatibility profile via Mesa st/mesa) |
| User-space stack | Mesa Lima Gallium driver, EGL, GBM |
| Kernel driver | `lima` (mainline, `drivers/gpu/drm/lima/`) |

## Supported boards

| Board | Product name | SoC | RAM |
|-------|--------------|-----|-----|
| AML-S905X-CC | Le Potato | S905X | 1 / 2 GB |
| AML-S905X-CC-V2 | Sweet Potato | S905X | 1 / 2 GB |
| AML-S905X-CC-V3 | Das Potato | S905X | 2 GB |
| AML-S805X-AC | La Frite | S805X | 512 MB / 1 GB |
| AML-S805X-AC-V2 | Das Frite | S805X | 512 MB / 1 GB |
| AML-S905D-PC | Tartiflette | S905D | 2 GB / 3 GB |

Mali-450 also exists on the S912 (octa-A53) but is not present on any
shipping Libre Computer board.

## What it can do

- OpenGL ES 2.0 (GLSL 1.00 ES)
- OpenGL 2.1 compatibility profile (fixed-function pipeline emulated by
  Mesa `st/mesa`: matrix stack, glBegin/glEnd, glLight, glFog, display
  lists, ARB assembly programs)
- Vertex/fragment shading with full IEEE-754 single precision in the
  Geometry Processor (GP)
- Multi-sample anti-aliasing (MSAA, up to 4x EXT_multisampled_render_to_texture)
- Hardware texture filtering (linear, mipmap, anisotropic)
- ETC1 compressed textures
- Shadow samplers (`GL_EXT_shadow_samplers`)
- Programmable framebuffer fetch (`GL_EXT_shader_framebuffer_fetch`) --
  enables tile-resident blending in user shaders
- Off-screen rendering via FBO, EGL surfaceless, GBM, Wayland WSI
- Headless rendering (no display required) via EGL surfaceless or
  `eglGetPlatformDisplay(EGL_PLATFORM_GBM_KHR)` on `/dev/dri/renderD128`

## What it cannot do (silicon limits)

- OpenGL ES 3.0 / 3.1 / 3.2 -- requires geometry shaders, transform
  feedback, compute shaders that the Utgard pipeline does not implement
- Vulkan -- there is no Vulkan driver targeting Utgard
- ASTC compressed textures (Bifrost-only)
- HW MSAA resolve (no resolve unit; multi-sample resolve is a limitation
  of the design rather than a driver gap)
- Stencil INCR / DECR / WRAP arithmetic ops on d24s8 -- partially
  functional in silicon; passes core stencil but fails specific dEQP
  ops sub-tests. Stencil compare and replace work correctly.

## Use cases

The Mali-450 is well-matched to embedded UI and lightweight 3D
workloads:

- **Kiosk / dashboard UIs** -- Wayland or X11 compositors, GTK with
  GLES2 rendering, browser canvas/WebGL via Wayland.
- **Digital signage** -- single full-screen video texture, animated
  overlays, GLSL shader effects.
- **Embedded HMI** -- Qt Quick (QtQuick 2.x targets GLES2), LVGL with
  the GLES2 backend, EmbeddedSwing.
- **Retro games / 2D engines** -- SDL2-GLES2, Love2D, Allegro 5,
  Cocos2d-x.
- **Lightweight emulators** -- via gl4es / gallium-llvmpipe-fallback for
  legacy GL workloads not covered by GLES2.

For 3D-heavy workloads (modern games, full GLES 3.x renderers, Vulkan)
use a Bifrost board (Mali-G31 on the SM1 family, Mali-G52 on the G12B
family).

## See also

- [api.md](api.md) -- device nodes, EGL platforms, environment, code patterns
- [benchmarks.md](benchmarks.md) -- glmark2, draw-call, shader fill-rate
  numbers per platform and clock
