# API: Mali-G31 / Mali-G52 via Panfrost / PanVK

Mali Bifrost acceleration is exposed through standard Linux DRM,
Mesa, and Vulkan loader interfaces. There are no Amlogic-specific
APIs -- application code is identical to any other Mali Bifrost or
Mesa-Panfrost target.

## Device nodes

| Node | Driver | Purpose |
|------|--------|---------|
| `/dev/dri/card0` | `meson-vpu` | KMS scanout (display, plane, CRTC, connector) |
| `/dev/dri/card1` | `panfrost` | GPU render node (legacy fd) |
| `/dev/dri/renderD128` | `panfrost` | GPU render node (recommended for headless / unprivileged use) |

`/dev/dri/renderD128` is the standard render-only node. Group `render`
membership grants access without root.

`meson-vpu` exposes the display engine and `panfrost` exposes the GPU
as two separate DRM devices on this SoC. Mesa's `kmsro` infrastructure
transparently bridges them when a GBM-backed scanout buffer is needed:
`gbm_create_device(/dev/dri/card0)` allocates the front buffer on
`meson-vpu`, but render commands route to `panfrost` automatically. No
manual device pairing is required from application code.

## OpenGL ES / OpenGL detection

```c
#include <EGL/egl.h>
#include <GLES3/gl32.h>

const char *vendor   = (const char *)glGetString(GL_VENDOR);
const char *renderer = (const char *)glGetString(GL_RENDERER);
const char *version  = (const char *)glGetString(GL_VERSION);

/*
 * On a Panfrost-driven Mali-G31 / G52:
 *   GL_VENDOR    = "Mesa"
 *   GL_RENDERER  = "Mali-G31 (Panfrost)"   (Solitude / SM1)
 *                  "Mali-G52 r1 (Panfrost)" (Alta / G12B)
 *   GL_VERSION   = "OpenGL ES 3.2 Mesa 25.0.7" (or current Mesa version)
 */
```

Detect "this is a Bifrost Mali" robustly via a `GL_RENDERER`
substring match on `Panfrost`. Discriminate G31 vs G52 with a
substring match on `G31` or `G52`.

## Vulkan detection

```c
#include <vulkan/vulkan.h>

VkPhysicalDeviceProperties props;
vkGetPhysicalDeviceProperties(physical_device, &props);

/*
 * On a PanVK-driven Mali-G31 / G52:
 *   props.deviceName     = "Mali-G31 (Panfrost)" or "Mali-G52 (Panfrost)"
 *   props.apiVersion     = VK_API_VERSION_1_3
 *   props.driverVersion  = encoded Mesa version
 */
```

Standard `vulkaninfo` lists the Vulkan device, supported extensions,
and limits. The PanVK ICD is registered in
`/usr/share/vulkan/icd.d/panfrost_icd.aarch64.json`.

## Supported EGL platforms

Panfrost works with all standard Mesa EGL platforms:

| Platform | Display server required? | Typical use |
|----------|--------------------------|-------------|
| `EGL_PLATFORM_SURFACELESS_KHR` | No | Headless compute, off-screen rendering, server-side image generation |
| `EGL_PLATFORM_GBM_KHR` | No | Direct DRM scanout, fbcon-style fullscreen apps |
| `EGL_PLATFORM_WAYLAND_KHR` | Wayland compositor (weston, sway, KDE, ...) | Standard Wayland clients |
| `EGL_PLATFORM_X11_KHR` | X server | Legacy X11 clients |

The Wayland and GBM paths use the `kmsro` bridge described above
without app intervention.

### Headless EGL example (GLES 3.2)

```c
#include <EGL/egl.h>
#include <EGL/eglext.h>
#include <GLES3/gl32.h>

EGLDisplay dpy = eglGetPlatformDisplay(EGL_PLATFORM_SURFACELESS_MESA, EGL_DEFAULT_DISPLAY, NULL);
eglInitialize(dpy, NULL, NULL);
eglBindAPI(EGL_OPENGL_ES_API);

EGLint cfg_attrs[] = {
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,
    EGL_RED_SIZE,   8,
    EGL_GREEN_SIZE, 8,
    EGL_BLUE_SIZE,  8,
    EGL_ALPHA_SIZE, 8,
    EGL_DEPTH_SIZE, 24,
    EGL_NONE,
};
EGLConfig cfg;
EGLint    n;
eglChooseConfig(dpy, cfg_attrs, &cfg, 1, &n);

EGLint ctx_attrs[] = {
    EGL_CONTEXT_MAJOR_VERSION, 3,
    EGL_CONTEXT_MINOR_VERSION, 2,
    EGL_NONE,
};
EGLContext ctx = eglCreateContext(dpy, cfg, EGL_NO_CONTEXT, ctx_attrs);
eglMakeCurrent(dpy, EGL_NO_SURFACE, EGL_NO_SURFACE, ctx);

/* ... GLES 3.2 commands, FBO render, glReadPixels, etc. ... */
```

### Headless desktop OpenGL example (GL 3.3)

```c
EGLint ctx_attrs[] = {
    EGL_CONTEXT_MAJOR_VERSION, 3,
    EGL_CONTEXT_MINOR_VERSION, 3,
    EGL_CONTEXT_OPENGL_PROFILE_MASK, EGL_CONTEXT_OPENGL_CORE_PROFILE_BIT,
    EGL_NONE,
};
eglBindAPI(EGL_OPENGL_API);
EGLContext ctx = eglCreateContext(dpy, cfg, EGL_NO_CONTEXT, ctx_attrs);
```

## Library loading

Mesa Panfrost ships in Debian/Ubuntu as part of the
`libgl1-mesa-dri`, `libegl1`, `libgles2`, `libgles2-mesa` package
set. PanVK Vulkan ships in `mesa-vulkan-drivers`. Detect that the
GLES / EGL / Vulkan ICD is the Mesa one by checking the version
string.

To force a specific Mesa install (e.g., a self-built Mesa under
`/opt/mesa`) prefix `LD_LIBRARY_PATH` for GL/GLES, and set
`VK_DRIVER_FILES` for Vulkan:

```bash
LD_LIBRARY_PATH=/opt/mesa/lib/aarch64-linux-gnu my-gles-app

VK_DRIVER_FILES=/opt/mesa/share/vulkan/icd.d/panfrost_icd.aarch64.json \
  LD_LIBRARY_PATH=/opt/mesa/lib/aarch64-linux-gnu my-vulkan-app
```

Panfrost's Gallium DRI module is `panfrost_dri.so`. PanVK's Vulkan
ICD is `libvulkan_panfrost.so`. Mesa loads both automatically when
it sees the panfrost device on `/dev/dri/`.

## Headless render setup (no display attached)

A board with no monitor connected has no active KMS connector by
default; KMS-tied compositors (the DRM-backed glmark2 variant,
kmscube, etc.) will not start. There are two options:

1. **EGL surfaceless** -- as in the example above. Suitable for
   benchmarks, automated tests, server-side image generation.

2. **Headless Wayland compositor** -- run a compositor like
   `weston --backend=headless --renderer=gl`, then point a Wayland
   client at it via `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR`. Useful
   for running existing Wayland-only apps in CI.

```bash
mkdir -p /tmp/wl && chmod 0700 /tmp/wl
XDG_RUNTIME_DIR=/tmp/wl \
  weston --backend=headless --renderer=gl --width=1024 --height=768 --socket=bench &
sleep 2

XDG_RUNTIME_DIR=/tmp/wl WAYLAND_DISPLAY=bench glmark2-es2-wayland -s 1024x768
```

For Vulkan apps, the same Wayland surface works through Vulkan WSI:

```bash
XDG_RUNTIME_DIR=/tmp/wl WAYLAND_DISPLAY=bench vkcube --present_mode fifo
```

## Permissions

Render-node access requires membership in the `render` group:

```bash
sudo usermod -aG render $USER
# log out and back in for the supplementary group to take effect
```

KMS scanout (`/dev/dri/card0`) traditionally requires `video` or
seat-managed access. For headless / EGL-surfaceless workflows, only
`render` is required.

## Frequency control (devfreq)

The GPU is a devfreq device under `/sys/class/devfreq/`:

```bash
DEVFREQ=$(ls -d /sys/class/devfreq/*.gpu | head -1)

cat $DEVFREQ/available_frequencies   # 7 OPPs from 125 to 800 MHz
cat $DEVFREQ/cur_freq                # current
cat $DEVFREQ/governor                # default: simple_ondemand

# Lock to maximum (benchmarking):
echo performance > $DEVFREQ/governor

# Restore default:
echo simple_ondemand > $DEVFREQ/governor
```

The default `simple_ondemand` governor scales between OPPs based on
GPU busy fraction. For predictable benchmark numbers, set the
`performance` governor on both the GPU devfreq and the CPU cpufreq
nodes.

The OPP table for both Mali-G31 (Solitude) and Mali-G52 (Alta) ships
with 7 frequencies between 125 MHz and 800 MHz. Both run from a
shared SoC voltage rail (no per-GPU regulator).

## Performance counters

Panfrost exposes hardware performance counters via the kernel
unstable IOCTL interface. Enable at module load:

```bash
sudo modprobe panfrost unstable_ioctls=1
```

Tools that consume Panfrost perfcnt:

- `pan_perfcnt` (Mesa source tree, build-time tool).
- `perfetto` GPU producer (when Perfetto is built with Panfrost
  support).

The counters cover ALU cycles, fragment cycles, tile-list parse
cycles, memory traffic, etc. See the Panfrost upstream documentation
for the full counter set.

## Upstream documentation

- Mesa Panfrost driver -- `src/gallium/drivers/panfrost/` (GL/GLES
  Gallium driver) and `src/panfrost/vulkan/` (PanVK Vulkan driver)
  in the Mesa source tree.
- Linux kernel driver -- `Documentation/gpu/drivers.rst`
  (`panfrost` section) and `drivers/gpu/drm/panfrost/` in the
  kernel source tree.
- DRM render node concept -- `Documentation/gpu/drm-uapi.rst`.
- Vulkan loader -- `Documentation/vulkan-spec` /
  https://vulkan.lunarg.com/doc/sdk/latest/linux/loader_and_layer_interface.html.
