# Amlogic 2D Graphics Engine

Hardware-accelerated 2D blit engine on Amlogic SoCs. Performs
format conversion, scaling, blending, and rigid-body transforms
with zero CPU involvement, exposed as a V4L2 memory-to-memory
device.

## Supported boards

| Board | SoC | V4L2 device |
|-------|-----|-------------|
| AML-S805X-AC (La Frite)        | S805X  | `/dev/videoN` (meson-ge2d) |
| AML-S805X-AC-V2 (Das Frite)    | S805X  | `/dev/videoN` |
| AML-S905X-CC (Le Potato)       | S905X  | `/dev/videoN` |
| AML-S905X-CC-V2 (Sweet Potato) | S905X  | `/dev/videoN` |
| AML-S905X-CC-V3 (Das Potato)   | S905X  | `/dev/videoN` |
| AML-A311D-CC (Alta)            | A311D  | `/dev/videoN` |
| AML-A311D-CM (Alta CM)         | A311D  | `/dev/videoN` |
| AML-S905D3-CC (Solitude)       | S905D3 | `/dev/videoN` |
| AML-S905D3-CM (Solitude CM)    | S905D3 | `/dev/videoN` |

The device node number depends on what other V4L2 drivers are
present on the running kernel. Find it with:

```sh
v4l2-ctl --list-devices | grep -A1 meson-ge2d
```

## Capabilities

### Format conversion (39 pixel formats)

| Category | Formats |
|----------|---------|
| 32-bit RGB/BGR | XRGB32, ARGB32, RGBX32, RGBA32, BGRX32, BGRA32, XBGR32, ABGR32 |
| 24-bit RGB/BGR | RGB24, BGR24 |
| 16-bit RGB | RGB565, XRGB555, ARGB555, XRGB444, ARGB444, RGBX444, RGBA444 |
| Packed YUV | YUYV (YUV422), YUV24 (YUV444), YUV32, YUVA32, YUVX32, AYUV32, VUYA32, VUYX32 |
| Semi-planar YUV | NV12, NV21, NV12M, NV21M (4:2:0), NV16, NV61 (4:2:2) |
| 3-plane planar YUV | YU12/I420, YV12, YUV422P |
| MESON deep-color | AM4A (10-bit YUV444), AM2A (10-bit YUV422), AM2C (12-bit YUV422), AMRA (10-bit RGB) |
| 8-bit indexed palette | PAL8 (256-entry BGR lookup) |

All conversion pairs run in hardware with BT.601 or BT.709 color
matrix; source selects the colorspace tag via V4L2's
`V4L2_COLORSPACE_SMPTE170M` (BT.601) or `V4L2_COLORSPACE_REC709`
(BT.709).

### Scaling

33-phase polyphase scaler with three filter tables selectable via
the `V4L2_CID_USER_MESON_GE2D_SCALE_FILTER` control: bilinear
(default), bicubic, triangle. Arbitrary ratios from 1:8 downscale
to 8:1 upscale in a single pass; a hardware pre-decimator engages
transparently beyond 2x downscale.

### Alpha blending

Porter-Duff compositing via per-pixel alpha, global alpha
(`V4L2_CID_ALPHA_COMPONENT`), and premultiplied-alpha modes.
Dual-source composition through the `VIDIOC_MESON_GE2D_SET_SRC2`
private ioctl: a second source buffer (or fill color) blends into
the primary source with any Porter-Duff mode in one hardware pass.

### Transforms

Horizontal flip, vertical flip, 90/180/270 rotation, and
rectangular cropping via the V4L2 selection API -- all combinable
with scaling and format conversion in the same blit.

### 10- and 12-bit deep color

On A311D (G12B) and S905D3 (SM1): the MESON deep-color formats
(AM4A, AM2A, AM2C, AMRA) engage a 10/12-bit internal pipeline. CSC
and scale preserve the extra precision; memory stores the top 8
bits of each component. S805X and S905X do not support deep color.

Byte-layout details are in the kernel's
`Documentation/userspace-api/media/v4l/pixfmt-meson-deep-color.rst`.

### Indexed palette (CLUT8)

256-entry color lookup table on the SRC1 path, selected by using
`V4L2_PIX_FMT_PAL8` on the output queue. The palette is uploaded
via the `VIDIOC_MESON_GE2D_SET_CLUT8` private ioctl and persists
per file handle until replaced.

### Dynamic frequency scaling

Four clock states (250 / 400 / 500 / 667 MHz) exposed through the
thermal cooling framework. The driver auto-throttles under
thermal pressure; userspace can force the 667 MHz turbo state via
the `turbo` sysfs attribute.

## Common use cases

- **Video playback**: V4L2 decoder emits NV12, 2D engine converts
  to XRGB for display in zero CPU cycles.
- **Display compositing**: blend DRM video plane (SRC1, scaled
  NV12 -> XRGB) with an OSD plane (SRC2, ARGB) into a single
  framebuffer for headless RDP / VNC streaming.
- **Transcode preprocessing**: RGB editing workspace -> YUV
  encoder input, or vice versa, in a DMABUF zero-copy pipeline
  between video decoder and encoder.
- **Thumbnail / preview generation**: downscale 4K frames to
  320x240 thumbnails at ~35 frames/sec on A311D.
- **Subtitle burn-in**: SRC1 = decoded NV12, SRC2 = pre-rendered
  subtitle ARGB, DST = encoder input. Single hardware pass.
- **Damage-tracked partial re-composite**: small dirty rectangles
  in a larger framebuffer re-composite only the changed area.

## Zero-copy pipeline

Both V4L2 queues support `V4L2_MEMORY_DMABUF`. Import DMABUF fds
from a video decoder (capture side) or export from the 2D engine
to a KMS display or video encoder (both directions). Enables
end-to-end zero-copy pipelines where decoded video never lands
in userspace memory.

## Next steps

- [api.md](api.md) -- V4L2 interface, ioctls, controls, example code.
- [benchmarks.md](benchmarks.md) -- measured throughput per platform.
