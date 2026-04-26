# VP9 Decoder API

The VP9 decoder is a standard V4L2 stateful memory-to-memory device.
Userspace works it via the upstream
[V4L2 stateful decoder API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html).

## Device discovery

The decoder advertises itself in sysfs as `meson-video-decoder` (the
same node serves H.264, HEVC, MPEG-1/2/4, and VP9).  The
`/dev/videoN` index varies by SoC and boot order -- always discover
by name, never hardcode the index.

```c
/* Find the decoder by sysfs name. */
DIR *d = opendir("/sys/class/video4linux");
struct dirent *e;
char dev[64], name[64];
int fd = -1;
while ((e = readdir(d))) {
    if (strncmp(e->d_name, "video", 5)) continue;
    snprintf(dev,  sizeof(dev),  "/sys/class/video4linux/%s/name",  e->d_name);
    int nf = open(dev, O_RDONLY);
    if (nf < 0) continue;
    int n = read(nf, name, sizeof(name) - 1);
    close(nf);
    if (n <= 0) continue;
    name[n] = 0;
    if (n && name[n-1] == '\n') name[n-1] = 0;
    if (strcmp(name, "meson-video-decoder") == 0) {
        snprintf(dev, sizeof(dev), "/dev/%s", e->d_name);
        fd = open(dev, O_RDWR);
        break;
    }
}
closedir(d);
```

GStreamer's `v4l2vp9dec` element discovers the device automatically.
FFmpeg's `vp9_v4l2m2m` decoder does the same when invoked with
`-c:v vp9_v4l2m2m`.

## Pixel formats

OUTPUT (compressed input) queue:

| FourCC | Format |
|--------|--------|
| `VP90` (`V4L2_PIX_FMT_VP9`) | VP9 IVF / Matroska / WebM elementary stream |

CAPTURE (decoded output) queue: see the table in
[`README.md`](README.md).  Probe with `VIDIOC_ENUM_FMT` -- the set
differs by SoC.  AFBC-scatter formats (`AMS8`, `AMSA`) are A311D /
S905D3 only.

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_VP9;
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_width and {min,max}_height give the
 * SoC-specific bounds.  The driver computes the upper bound at
 * probe time from the available CMA pool. */
```

## V4L2 controls

The VP9 decoder exposes no driver-specific V4L2 controls -- all
parameters are derived from the bitstream or computed at probe time.
Use `VIDIOC_S_FMT`, `VIDIOC_REQBUFS`, and the standard
`V4L2_DEC_CMD_*` commands to drive a session.

## Reference applications

### GStreamer

```sh
gst-launch-1.0 \
    filesrc location=input.webm ! matroskademux ! vp9parse \
    ! v4l2vp9dec \
    ! videoconvert ! autovideosink
```

For low-latency capture-to-disk:

```sh
gst-launch-1.0 \
    filesrc location=input.webm ! matroskademux ! vp9parse ! v4l2vp9dec \
    ! 'video/x-raw,format=NV12' \
    ! filesink location=output.nv12
```

### FFmpeg

```sh
ffmpeg -c:v vp9_v4l2m2m -i input.webm -f null -
```

### libva (VA-API)

The Amlogic V4L2 m2m decoder is also available through the
`libva-v4l2-m2m` backend.  Set:

```sh
export LIBVA_DRIVER_NAME=v4l2_m2m
ffmpeg -hwaccel vaapi -hwaccel_output_format vaapi -i input.webm \
    -vf 'hwdownload,format=nv12' -pix_fmt nv12 -f rawvideo output.yuv
```

To switch the capture queue to AFBC-scatter (zero-copy to display) on
A311D / S905D3:

```sh
export LIBVA_V4L2_M2M_FBC=8     # 8-bit (Profile 0, AMS8)
export LIBVA_V4L2_M2M_FBC=10    # 10-bit (Profile 2, AMSA)
```

`LIBVA_V4L2_M2M_FBC=1` selects whichever of `AMS8` / `AMSA` matches
the stream's bit depth.

### Minimal V4L2 decode loop

A complete C example is out of scope here; refer to the upstream
kernel sample at
[`Documentation/userspace-api/media/v4l/dev-decoder.rst`](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html#decoding).
The Amlogic decoder follows that contract verbatim, including the
`V4L2_EVENT_SOURCE_CHANGE` and `V4L2_DEC_CMD_STOP` flow.

## AFBC export and scanout

When `AMS8` or `AMSA` is selected on the CAPTURE queue, exported
DMA-BUF handles carry an Amlogic AFBC modifier.  Userspace can
import the buffer into DRM with `drmModeAddFB2WithModifiers` and
scan it out directly via the meson VPU's VD1 plane:

| CAPTURE fourcc | DRM format | DRM modifier |
|----------------|------------|--------------|
| `AMS8` | `DRM_FORMAT_YUV420_8BIT` | `DRM_FORMAT_MOD_AMLOGIC_FBC(SCATTER, MEM_SAVING)` |
| `AMSA` | `DRM_FORMAT_YUV420_10BIT` | `DRM_FORMAT_MOD_AMLOGIC_FBC(SCATTER, 0)` |

The meson DRM driver advertises these modifier variants on the VD1
overlay plane on A311D and S905D3.

## Error reporting

The decoder reports recoverable errors via `V4L2_BUF_FLAG_ERROR` on
dequeued CAPTURE buffers.  Hard failures (firmware watchdog,
unsupported profile, exhausted CMA at session start) are reported via
the `V4L2_EVENT_*` event queue and `errno` on `VIDIOC_QBUF` /
`VIDIOC_DQBUF`.

Corrupt or partially-truncated input is handled gracefully: the
decoder skips invalid frames, decodes what it can, and signals `EOS`
cleanly when input runs out.

## Decode contract summary

- Bit depth: 8-bit Profile 0 on every supported SoC; 10-bit Profile 2
  on A311D and S905D3.
- Chroma: 4:2:0 only.  Profile 1 (4:2:2) and Profile 3 (4:4:4) input
  is rejected at `VIDIOC_S_FMT` and at frame parse.
- One decode session at a time per SoC (single hardware core).
- Maximum resolution and pixel-format set vary by SoC; see
  [`README.md`](README.md).
