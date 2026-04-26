# MJPEG Decoder API

The MJPEG decoder is a standard V4L2 stateful memory-to-memory
device.  Userspace works it via the upstream
[V4L2 stateful decoder API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html).

## Device discovery

The MJPEG decoder advertises itself in sysfs as
`meson-video-decoder-jpeg` -- a separate node from the main
multi-codec stateful decoder (`meson-video-decoder`).  The
`/dev/videoN` index varies by SoC and boot order; always discover
by sysfs name:

```c
/* Find the MJPEG decoder by sysfs name. */
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
    if (strcmp(name, "meson-video-decoder-jpeg") == 0) {
        snprintf(dev, sizeof(dev), "/dev/%s", e->d_name);
        fd = open(dev, O_RDWR);
        break;
    }
}
closedir(d);
```

The MJPEG node implements the V4L2 `V4L2_CAP_VIDEO_M2M_MPLANE`
class with the JPEG decoder bit set.

GStreamer's `v4l2jpegdec` element discovers the device
automatically (filtered on the JPEG decoder class).

## Pixel formats

OUTPUT (compressed input) queue:

| FourCC | Format |
|--------|--------|
| `MJPG` (`V4L2_PIX_FMT_MJPEG`) | Motion-JPEG / JFIF / single JPEG |

CAPTURE (decoded output) queue:

| FourCC | Format |
|--------|--------|
| `YM12` (`V4L2_PIX_FMT_YUV420M`) | 3-plane I420 |

Probe with `VIDIOC_ENUM_FMT`.  Userspace must select `YM12` on
the CAPTURE queue -- 1-plane I420 / NV12 are not offered.

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_MJPEG;
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_width gives 1..8192 px;
 * fs.stepwise.{min,max}_height gives 1..2048 px. */
```

## V4L2 controls

The MJPEG decoder exposes no driver-specific V4L2 controls -- all
parameters (subsampling, restart-marker handling, output stride)
are derived from the JPEG bitstream itself.  Use `VIDIOC_S_FMT`,
`VIDIOC_REQBUFS`, and the standard `V4L2_DEC_CMD_*` commands to
drive a session.

## Reference applications

### GStreamer

Motion-JPEG (raw concatenated stream):

```sh
gst-launch-1.0 \
    filesrc location=input.mjpeg ! jpegparse \
    ! v4l2jpegdec \
    ! videoconvert ! autovideosink
```

Motion-JPEG inside an AVI container:

```sh
gst-launch-1.0 \
    filesrc location=input.avi ! avidemux ! jpegparse \
    ! v4l2jpegdec \
    ! videoconvert ! autovideosink
```

For low-latency capture-to-disk:

```sh
gst-launch-1.0 \
    filesrc location=input.mjpeg ! jpegparse ! v4l2jpegdec \
    ! 'video/x-raw,format=I420' \
    ! filesink location=output.i420
```

### FFmpeg

```sh
ffmpeg -c:v mjpeg_v4l2m2m -i input.mjpeg -f null -
```

### libva (VA-API)

The Amlogic V4L2 m2m MJPEG decoder is also available through the
`libva-v4l2-m2m` backend, exposed as `VAProfileJPEGBaseline`.
Set:

```sh
export LIBVA_DRIVER_NAME=v4l2_m2m
ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
    -hwaccel_output_format vaapi -i input.mjpeg \
    -vf 'hwdownload,format=nv12' -pix_fmt nv12 \
    -f rawvideo output.yuv
```

The libva backend reconstructs the JPEG bitstream from VA buffer
parameters, routes the session to the dedicated MJPEG V4L2 node,
and forces 3-plane I420 on the CAPTURE queue.  Decoded frames are
returned through the standard VAAPI surface flow; userspace can
either `vaDeriveImage` to read NV12 (libva does the I420->NV12
conversion in `hwdownload`) or `vaExportSurfaceHandle` to obtain a
DMA-BUF for downstream consumers.

If the input bitstream is not 4:2:0, the kernel rejects it at
queue stage and libva propagates the error to the application
immediately (no firmware stall).

### Single JPEG still image

The same V4L2 device decodes one-shot JPEG still images.  Feed a
single JPEG frame on the OUTPUT queue, send `V4L2_DEC_CMD_STOP`
to flush, dequeue the decoded I420 frame on the CAPTURE queue.
The decoder makes no distinction between an MJPEG sequence and a
single still.

`gst-launch-1.0 filesrc ! jpegparse ! v4l2jpegdec ! filesink`
works for a one-frame `.jpg` file as well as a multi-frame
`.mjpeg`.

### Minimal V4L2 decode loop

A complete C example is out of scope here; refer to the upstream
kernel sample at
[`Documentation/userspace-api/media/v4l/dev-decoder.rst`](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html#decoding).
The Amlogic MJPEG decoder follows that contract verbatim,
including the `V4L2_EVENT_SOURCE_CHANGE` and `V4L2_DEC_CMD_STOP`
flow.

## Input subsampling

The decoder accepts 4:2:0 input only.  Non-4:2:0 (4:2:2, 4:4:4,
grayscale) JPEGs are rejected at the queue stage with `-EINVAL`
before the firmware sees them.  The driver parses the SOF0 / SOF1
marker and rejects any source whose Y component sampling factor
is not 0x22 (2H x 2V) or whose component count is below three.

FFmpeg's MJPEG encoder defaults to 4:4:4 at some resolutions.
When generating MJPEG content for this decoder, always pass
`-pix_fmt yuvj420p` explicitly:

```sh
ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=30 \
    -pix_fmt yuvj420p -c:v mjpeg -q:v 2 out.mjpeg
```

Resolutions where ffmpeg defaults to 4:4:4 (and therefore need
the override): 640x480, 800x600, 1280x1024.

## Error reporting

The decoder reports per-frame errors via `V4L2_BUF_FLAG_ERROR` on
dequeued CAPTURE buffers.  Hard failures (truncated SOI,
malformed SOF, unsupported subsampling) are reported via the V4L2
event queue and `errno` on `VIDIOC_QBUF` / `VIDIOC_DQBUF`.  Bad
input is rejected immediately rather than allowed to time out
inside the firmware.

Corrupt or partially-truncated input is handled gracefully: the
decoder skips invalid markers, decodes what it can per frame, and
signals `EOS` cleanly when input runs out.

## Decode contract summary

- Bit depth: 8-bit only.
- Chroma: 4:2:0 only.  4:2:2, 4:4:4, and grayscale input is
  rejected at the queue stage with `-EINVAL`.
- Output: 3-plane I420 (`V4L2_PIX_FMT_YUV420M`) -- the only
  advertised CAPTURE format.
- One decode session at a time per SoC (single hardware core).
- Maximum resolution: 8192x2048 on every supported SoC.
