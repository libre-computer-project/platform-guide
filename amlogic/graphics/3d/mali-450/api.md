# API: Mali-450 via Lima

Mali-450 acceleration is exposed through standard Linux DRM and Mesa
interfaces. There are no Amlogic-specific APIs -- application code is
identical to any other Mali Utgard or Mesa-Lima target.

## Device nodes

| Node | Driver | Purpose |
|------|--------|---------|
| `/dev/dri/card0` | `meson-vpu` | KMS scanout (display, plane, CRTC, connector) |
| `/dev/dri/card1` | `lima` | GPU render node (legacy fd) |
| `/dev/dri/renderD128` | `lima` | GPU render node (recommended for headless / unprivileged use) |

`/dev/dri/renderD128` is the standard render-only node. Group `render`
membership grants access without root.

`meson-vpu` exposes the display engine and `lima` exposes the GPU as
two separate DRM devices on this SoC. The Mesa `kmsro` infrastructure
transparently bridges them when a GBM-backed scanout buffer is needed:
`gbm_create_device(/dev/dri/card0)` allocates the front buffer on
`meson-vpu`, but render commands route to `lima` automatically. No
manual device pairing is required from application code.

## OpenGL ES detection

```c
#include <EGL/egl.h>
#include <GLES2/gl2.h>

const char *vendor   = (const char *)glGetString(GL_VENDOR);
const char *renderer = (const char *)glGetString(GL_RENDERER);
const char *version  = (const char *)glGetString(GL_VERSION);

/*
 * On a Lima-driven Mali-450:
 *   GL_VENDOR    = "Mesa"
 *   GL_RENDERER  = "Mali450"
 *   GL_VERSION   = "OpenGL ES 2.0 Mesa 25.0.7" (or current Mesa version)
 */
```

Detect "this is a Mali-450" robustly via a `GL_RENDERER` substring
match on `Mali450` (note: no space, no MP3 suffix in the string).

## Supported EGL platforms

Lima works with all standard Mesa EGL platforms:

| Platform | Display server required? | Typical use |
|----------|--------------------------|-------------|
| `EGL_PLATFORM_SURFACELESS_KHR` | No | Headless compute, off-screen rendering, server-side image generation |
| `EGL_PLATFORM_GBM_KHR` | No | Direct DRM scanout, fbcon-style fullscreen apps |
| `EGL_PLATFORM_WAYLAND_KHR` | Wayland compositor (weston, sway, ...) | Standard Wayland clients |
| `EGL_PLATFORM_X11_KHR` | X server | Legacy X11 clients |

The Wayland and GBM paths use the `kmsro` bridge described above
without app intervention.

### Headless EGL example

```c
#include <EGL/egl.h>
#include <EGL/eglext.h>

EGLDisplay dpy = eglGetPlatformDisplay(EGL_PLATFORM_SURFACELESS_MESA, EGL_DEFAULT_DISPLAY, NULL);
eglInitialize(dpy, NULL, NULL);
eglBindAPI(EGL_OPENGL_ES_API);

EGLint cfg_attrs[] = {
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES2_BIT,
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

EGLint ctx_attrs[] = { EGL_CONTEXT_CLIENT_VERSION, 2, EGL_NONE };
EGLContext ctx = eglCreateContext(dpy, cfg, EGL_NO_CONTEXT, ctx_attrs);
eglMakeCurrent(dpy, EGL_NO_SURFACE, EGL_NO_SURFACE, ctx);

/* ... draw to FBOs, glReadPixels, etc. ... */
```

## Library loading

Mesa Lima ships in Debian/Ubuntu as part of the `libgl1-mesa-dri`,
`libegl1`, `libgles2` package set. Detect that the GLES2 / EGL ICD is
the Mesa one by checking the version string.

To force a specific Mesa install (e.g., a self-built Mesa under
`/opt/mesa`) prefix `LD_LIBRARY_PATH`:

```bash
LD_LIBRARY_PATH=/opt/mesa/lib/aarch64-linux-gnu my-gles-app
```

Lima's Gallium DRI module is `lima_dri.so`; Mesa loads it via
`libgallium_dri.so` automatically when it sees the lima KMS-renderer
device on `/dev/dri/`.

## Headless render setup (no display attached)

A Mali-450 board with no monitor connected has no active KMS
connector by default; KMS-tied compositors (the DRM-backed glmark2
variant, kmscube, etc.) will not start. There are two options:

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

## Permissions

Render-node access requires membership in the `render` group:

```bash
sudo usermod -aG render $USER
# log out and back in for the supplementary group to take effect
```

KMS scanout (`/dev/dri/card0`) traditionally requires `video` or
seat-managed access. For headless / EGL-surfaceless workflows, only
`render` is required.

## Frequency control (cpufreq-style)

The GPU is a devfreq device under `/sys/class/devfreq/`:

```bash
DEVFREQ=$(ls -d /sys/class/devfreq/*.gpu | head -1)

cat $DEVFREQ/available_frequencies   # list of OPPs
cat $DEVFREQ/cur_freq                # current
cat $DEVFREQ/governor                # default: simple_ondemand

# Lock to maximum (benchmarking):
echo performance > $DEVFREQ/governor

# Restore default:
echo simple_ondemand > $DEVFREQ/governor
```

The default `simple_ondemand` governor scales between OPPs based on GPU
busy fraction. For predictable benchmark numbers, set the `performance`
governor on both the GPU devfreq and the CPU cpufreq nodes.

## Upstream documentation

- Mesa Lima driver -- `src/gallium/drivers/lima/` in the Mesa source tree
- Linux kernel driver -- `Documentation/gpu/drivers.rst` (`lima` section)
  and `drivers/gpu/drm/lima/` in the kernel source tree
- DRM render node concept -- `Documentation/gpu/drm-uapi.rst`
