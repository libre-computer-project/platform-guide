# H.264 Decoder API

The H.264 decoder is a standard V4L2 stateful memory-to-memory
device.  Userspace works it via the upstream
[V4L2 stateful decoder API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html).

## Device discovery

The decoder advertises itself in sysfs as
`meson-video-decoder`.  The `/dev/videoN` index varies by SoC and
boot order -- always discover by name, never hardcode the index.

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

GStreamer's `v4l2h264dec` element discovers the device automatically.
FFmpeg's `h264_v4l2m2m` decoder does the same when invoked with
`-c:v h264_v4l2m2m`.

## Pixel formats

OUTPUT (compressed input) queue:

| FourCC | Format |
|--------|--------|
| `H264` (`V4L2_PIX_FMT_H264`) | H.264 byte-stream / Annex B |

CAPTURE (decoded output) queue: see the table in
[`README.md`](README.md).  Probe with `VIDIOC_ENUM_FMT` --
the set differs by SoC.

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_H264;
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_width and {min,max}_height give the
 * SoC-specific bounds.  The driver computes the upper bound at
 * probe time from the available CMA pool. */
```

## Custom V4L2 controls

Six driver-specific controls expose H.264-firmware behaviour to
userspace.  All are read/write, runtime-tunable, and have sensible
defaults.

| CID | Type | Range | Default | Purpose |
|-----|------|-------|---------|---------|
| `0x00981900` (`i_frame_only_decode`) | bool | 0..1 | 0 | Skip P/B frames; decode I-frames only.  Useful for thumbnail extraction. |
| `0x00981901` (`decode_timeout_ms`) | integer | 0..30000 | 200 | Per-macroblock-row watchdog timeout.  0 disables. |
| `0x00981902` (`error_processing_policy`) | bitmask | 0x3FFFFFF | 0x7FCFF6 | Per-error-class behaviour (drop / output-anyway / reset).  Bit 0 gates error-frame output; bit 14 enables reset-on-watchdog. |
| `0x00981903` (`fast_output_enable`) | bitmask | 0xF | 0 | Firmware fast-output hints.  Latency vs throughput trade-off. |
| `0x00981904` (`max_allocated_buffers`) | integer | 0..64 | 0 | Cap on driver-allocated CAPTURE buffers.  0 = use level-implied default. |
| `0x00981905` (`dpb_size_margin`) | integer | 0..32 | 7 | Extra DPB slots beyond the SPS-derived minimum. |

Controls are per-fd: setting via `v4l2-ctl --set-ctrl` then
re-opening the device with `gst-launch-1.0` will not preserve the
setting.  Use `extra-controls=...` on the GStreamer element, or
the `VIDIOC_S_EXT_CTRLS` ioctl in your application.

## Reference applications

### GStreamer

```sh
gst-launch-1.0 \
    filesrc location=input.mp4 ! qtdemux ! h264parse \
    ! v4l2h264dec \
    ! videoconvert ! autovideosink
```

For low-latency capture-to-disk:

```sh
gst-launch-1.0 \
    filesrc location=input.mp4 ! qtdemux ! h264parse ! v4l2h264dec \
    ! 'video/x-raw,format=NV12' \
    ! filesink location=output.nv12
```

### FFmpeg

```sh
ffmpeg -c:v h264_v4l2m2m -i input.mp4 -f null -
```

### Minimal V4L2 decode loop

A complete C example is out of scope here; refer to the upstream
kernel sample at
[`Documentation/userspace-api/media/v4l/dev-decoder.rst`](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html#decoding).
The Amlogic decoder follows that contract verbatim, including the
`V4L2_EVENT_SOURCE_CHANGE` and `V4L2_DEC_CMD_STOP` flow.

## Multi-View Coding (MVC)

The decoder advertises a separate input pixel format for stereo
3D MVC streams:

| FourCC | Format |
|--------|--------|
| `M264` (`V4L2_PIX_FMT_H264_MVC`) | H.264 MVC base + enhancement layer |

Two output modes via the `mvc_output_mode` module parameter:

| Mode | Value | CAPTURE buffer shape | Consumer |
|------|-------|---------------------|----------|
| Frame-packed (default) | 0 | `W x (H*2 + 45)` NV12, view 0 + 45-line gap + view 1 | HDMI 1.4a, Kodi, Blu-ray 3D |
| Frame-sequential | 1 | One CAPTURE buffer per view, shared timestamp; view 0 = `V4L2_FIELD_NONE`, view 1 = `V4L2_FIELD_TOP` | GStreamer multiview pipelines |

See [`README-mvc.md`](README-mvc.md) for capability and limits.

## Error reporting

The decoder reports recoverable errors via `V4L2_BUF_FLAG_ERROR`
on dequeued CAPTURE buffers.  Hard failures (firmware watchdog,
unsupported profile) are reported via the `V4L2_EVENT_*` event
queue and `errno` on `VIDIOC_QBUF` / `VIDIOC_DQBUF`.

Corrupt or partially-truncated input is handled gracefully: the
decoder skips invalid NAL units, decodes what it can, and signals
`EOS` cleanly when input runs out.

## Decode contract summary

- Bit depth: 8-bit 4:2:0.  10-bit and 4:2:2 / 4:4:4 input is
  rejected at `VIDIOC_S_FMT`.
- One decode session at a time per SoC (single hardware core).
- Maximum resolution and pixel-format set vary by SoC; see
  [`README.md`](README.md).
