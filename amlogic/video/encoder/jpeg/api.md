# JPEG Encoder API

The JPEG encoder is a standard V4L2 stateful memory-to-memory
device.  Userspace works it via the upstream
[V4L2 stateful encoder API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-encoder.html).

## Device discovery

The encoder advertises itself in sysfs as `meson-video-encoder`.
The `/dev/videoN` index varies by SoC and boot order -- always
discover by name, never hardcode the index.

```c
/* Find the encoder by sysfs name. */
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
    if (strcmp(name, "meson-video-encoder") == 0) {
        snprintf(dev, sizeof(dev), "/dev/%s", e->d_name);
        fd = open(dev, O_RDWR);
        break;
    }
}
closedir(d);
```

The same `meson-video-encoder` device handles both H.264 and JPEG
encode -- the CAPTURE pixel format selects the codec.  Set
`pixelformat = V4L2_PIX_FMT_JPEG` on the CAPTURE queue for JPEG
encode.

GStreamer's `v4l2jpegenc` element discovers the device automatically.

FFmpeg does not currently ship a `mjpeg_v4l2m2m` encoder wrapper.
Use GStreamer or call `VIDIOC_S_FMT` directly to drive JPEG
encode from C.

## Pixel formats

OUTPUT (uncompressed input) and CAPTURE (compressed output)
formats are listed in [`README.md`](README.md).  Probe with
`VIDIOC_ENUM_FMT`.

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_JPEG;
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_{width,height} give the SoC-specific
 * bounds.  Width and height step is 16 pixels (MCU-aligned). */
```

`VIDIOC_S_FMT` rounds the requested width and height up to the
nearest 16-pixel multiple.  Read back the returned format and
feed source frames at the rounded dimensions.

## V4L2 controls

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_JPEG_COMPRESSION_QUALITY` | int  | 1 .. 100 | 85 |
| `V4L2_CID_JPEG_LUMA_QUANTIZATION`   | u8 array (64) | -- | spec Annex K.1 (scaled by quality) |
| `V4L2_CID_JPEG_CHROMA_QUANTIZATION` | u8 array (64) | -- | spec Annex K.2 (scaled by quality) |

Setting the luma or chroma quantisation control overrides the
quality-scaled table for that component.

### Per-fd semantics

V4L2 controls live on the file descriptor that opened them.
Setting a control via `v4l2-ctl --set-ctrl` and then re-opening
the device with `gst-launch-1.0` does not preserve the value.
Use `extra-controls=` on the GStreamer element, or call
`VIDIOC_S_EXT_CTRLS` from your application before queueing the
first OUTPUT buffer.

## GStreamer reference pipeline

Encode raw NV12 to a sequence of JPEG files at quality 85:

```sh
gst-launch-1.0 \
    videotestsrc num-buffers=30 \
    ! 'video/x-raw,format=NV12,width=1280,height=720,framerate=30/1' \
    ! v4l2jpegenc extra-controls="controls,compression_quality=85" \
    ! multifilesink location=frame_%05d.jpg
```

For 4:2:2 output, feed YUYV input.  For 4:4:4 output, feed
ABGR32 input.

## v4l2-ctl reference invocation

Single-shot encode of one raw NV12 frame to JPEG:

```sh
v4l2-ctl -d /dev/videoN \
    --set-ctrl compression_quality=85 \
    --set-fmt-video-out=width=320,height=240,pixelformat=NV12 \
    --set-fmt-video=width=320,height=240,pixelformat=JPEG \
    --stream-out-mmap --stream-mmap --stream-count=1 \
    --stream-from=input.nv12 --stream-to=output.jpg
```

The control set and the encode must run in the same `v4l2-ctl`
invocation -- controls do not persist across opens.

## 1080p input pad note

The encoder requires a 16-pixel-aligned height.  1080 is not 16-
aligned (1080 / 16 = 67.5).  `VIDIOC_S_FMT` with height = 1080
returns 1088; the application must pad the source to 1088 lines
before queueing.  GStreamer pipelines using `videotestsrc
height=1080` will fail at caps negotiation -- request
`height=1088` in the source caps, or insert a `videobox` /
`videoconvert` element to pad.

## Multi-instance

Single hardware encoder core.  A second `open()` of the device
returns `EBUSY` until the first session closes.  Concurrent
H.264 + JPEG sessions are not possible -- both modes share the
same hardware.

## Encode contract summary

- One `meson-video-encoder` device per SoC.
- 8-bit input.  RGB inputs are converted to YCbCr by hardware.
- Width and height step is 16 pixels.
- Each CAPTURE buffer carries one complete JFIF file.
- Subsampling is implicit from the input format
  (4:2:0 NV12 / NV21 -> 4:2:0 JPEG, YUYV -> 4:2:2 JPEG, ABGR32
  -> 4:4:4 JPEG).
