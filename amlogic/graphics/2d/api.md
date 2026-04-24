# V4L2 API

The 2D graphics engine is a standard V4L2 memory-to-memory (M2M)
multiplanar device. Any userspace stack that targets V4L2 M2M
(GStreamer `v4l2convert`, ffmpeg, libv4l2, raw ioctl) can drive it.

## Device

- Driver name: `meson-ge2d`
- Device class: `V4L2_CAP_VIDEO_M2M_MPLANE | V4L2_CAP_STREAMING`
- Memory models: MMAP, DMABUF (import + export)
- Queue types: `V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE` (source),
  `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE` (destination)

Find the device node:

```sh
v4l2-ctl --list-devices | grep -A1 meson-ge2d
```

List the formats it advertises on the OUTPUT side:

```sh
v4l2-ctl -d /dev/videoN --list-formats-out
```

## Standard controls

| Control | Type | Description |
|---------|------|-------------|
| `V4L2_CID_HFLIP` | boolean | Horizontal flip |
| `V4L2_CID_VFLIP` | boolean | Vertical flip |
| `V4L2_CID_ROTATE` | menu | 0 / 90 / 180 / 270 degrees |
| `V4L2_CID_ALPHA_COMPONENT` | integer (0..255) | Global alpha for SRC1 when blending |

## Driver-specific controls

Control IDs under `V4L2_CID_USER_BASE + 0x1080`:

| Control | Offset | Description |
|---------|--------|-------------|
| `V4L2_CID_USER_MESON_GE2D_LOGIC_OP` | +0 | ALU logic op for non-blend blits: COPY, CLEAR, SET, XOR, AND, OR |
| `V4L2_CID_USER_MESON_GE2D_COLORKEY` | +1 | SRC1 color-key value (ARGB) |
| `V4L2_CID_USER_MESON_GE2D_COLORKEY_MASK` | +2 | SRC1 color-key mask (0 = disabled) |
| `V4L2_CID_USER_MESON_GE2D_FILLCOLOR` | +3 | SRC1 fill color (non-zero = fill mode) |
| `V4L2_CID_USER_MESON_GE2D_SCALE_FILTER` | +4 | 0 = bilinear, 1 = bicubic, 2 = triangle |
| `V4L2_CID_USER_MESON_GE2D_BITMASK` | +5 | Per-pixel destination write mask |
| `V4L2_CID_USER_MESON_GE2D_ANTIFLICKER` | +6 | Enable the anti-flicker filter |

## Custom ioctls

### `VIDIOC_MESON_GE2D_SET_SRC2` -- dual-source compositing

Configures a second source buffer for hardware compositing. The
ALU blends SRC1 (output queue) with SRC2 (DMABUF or fill color)
and writes the result to DST (capture queue).

```c
#define VIDIOC_MESON_GE2D_SET_SRC2 \
    _IOW('V', BASE_VIDIOC_PRIVATE + 0, struct meson_ge2d_src2_info)

#define MESON_GE2D_SRC2_FLAG_FILL_COLOR          (1U << 0)
#define MESON_GE2D_SRC2_FLAG_SRC2_X_REV          (1U << 1)
#define MESON_GE2D_SRC2_FLAG_SRC2_Y_REV          (1U << 2)
#define MESON_GE2D_SRC2_FLAG_SRC1_PREMULTIPLIED  (1U << 3)

struct meson_ge2d_src2_info {
    __s32 fd;             /* DMABUF fd, -1 disable, -2 keep current */
    __u32 pixelformat;    /* V4L2 fourcc, packed single-plane only */
    __u32 width;
    __u32 height;
    __u32 bytesperline;
    __s32 x_offset;
    __s32 y_offset;
    __u32 blend_op;       /* packed Porter-Duff factors */
    __u32 src2_gb_alpha;  /* SRC2 global alpha 0..255 (A311D/S905D3 only) */
    __u32 const_color;    /* ALU constant color, ARGB8888 */
    __u32 def_color;      /* SRC2 default fill color, ARGB8888 */
    __u32 color_key;      /* SRC2 color key value (0 = disabled) */
    __u32 color_key_mask;
    __u32 repeat_x;       /* 0=1x, 1=2x, 2=4x, 3=8x */
    __u32 repeat_y;
    __u32 flags;          /* MESON_GE2D_SRC2_FLAG_* */
    __u32 reserved[2];    /* must be zero */
};
```

SRC2 constraints:

- No independent scaler; SRC2 is a direct pixel read with
  optional power-of-2 repeat.
- SRC2 and DST share the format register, so their pixel format
  must match.
- SRC2 supports packed single-plane formats only (no NV12 or
  planar YUV as SRC2).

### `VIDIOC_MESON_GE2D_SET_CLUT8` -- palette upload

256-entry color lookup table for the 8-bit indexed `PAL8` source
format.

```c
#define VIDIOC_MESON_GE2D_SET_CLUT8 \
    _IOW('V', BASE_VIDIOC_PRIVATE + 2, struct meson_ge2d_clut8_info)

struct meson_ge2d_clut8_info {
    __u32 data[256];   /* palette entries */
    __u32 count;       /* 1..256 */
    __u32 flags;       /* reserved, must be 0 */
    __u32 reserved[3]; /* must be 0 */
};
```

Palette entry byte layout (hardware-defined):

| Bits | Channel |
|------|---------|
| [31:24] | Blue |
| [23:16] | Green |
| [15:8]  | Red |
| [7:0]   | ignored |

The LUT pipeline has no alpha channel -- output alpha is fixed
to 0xFF. The LSB of each entry is ignored by the hardware.

## Minimal example: RGB -> NV12 conversion

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>

int fd = open("/dev/video3", O_RDWR);

/* OUTPUT queue: XRGB32 source */
struct v4l2_format out_fmt = {
    .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .fmt.pix_mp = {
        .width = 1920, .height = 1080,
        .pixelformat = V4L2_PIX_FMT_XRGB32,
        .field = V4L2_FIELD_NONE,
        .num_planes = 1,
    },
};
ioctl(fd, VIDIOC_S_FMT, &out_fmt);

/* CAPTURE queue: NV12M destination */
struct v4l2_format cap_fmt = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
    .fmt.pix_mp = {
        .width = 1920, .height = 1080,
        .pixelformat = V4L2_PIX_FMT_NV12M,
        .field = V4L2_FIELD_NONE,
        .num_planes = 2,
    },
};
ioctl(fd, VIDIOC_S_FMT, &cap_fmt);

/* REQBUFS, mmap, QBUF, STREAMON on both queues, DQBUF a completed
 * pair in a loop -- standard V4L2 M2M flow. */
```

## GStreamer

```sh
# Hardware-accelerated format conversion in a GStreamer pipeline:
gst-launch-1.0 v4l2src ! v4l2convert ! autovideosink
```

`v4l2convert` autodetects the 2D graphics engine as a format-
conversion element. DMABUF imports/exports work across V4L2 M2M
element boundaries (GStreamer 1.26+).

## Upstream kernel documentation

- [V4L2 M2M API overview](https://docs.kernel.org/userspace-api/media/v4l/dev-mem2mem.html)
- [Pixel formats](https://docs.kernel.org/userspace-api/media/v4l/pixfmt.html)
- MESON deep-color formats: kernel source
  `Documentation/userspace-api/media/v4l/pixfmt-meson-deep-color.rst`
