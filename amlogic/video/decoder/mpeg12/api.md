# MPEG-1 / MPEG-2 Decoder API

The MPEG-1 / MPEG-2 decoder is a standard V4L2 stateful
memory-to-memory device.  Userspace works it via the upstream
[V4L2 stateful decoder API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html).

## Device discovery

The decoder advertises itself in sysfs as `meson-video-decoder`,
the same name as the H.264, HEVC, and VP9 decoders on this SoC
family -- they share one `/dev/videoN` node and the input pixel
format selects which firmware path runs.  The `/dev/videoN` index
varies by SoC and boot order; always discover by name, never
hardcode the index.

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

GStreamer's `v4l2mpeg2dec` element discovers the device
automatically when present in the GStreamer build.  FFmpeg's
`mpeg1_v4l2m2m` and `mpeg2_v4l2m2m` decoders do the same when
invoked with `-c:v mpeg1_v4l2m2m` or `-c:v mpeg2_v4l2m2m`.

## Pixel formats

OUTPUT (compressed input) queue:

| FourCC | Format |
|--------|--------|
| `MPG1` (`V4L2_PIX_FMT_MPEG1`) | MPEG-1 elementary stream |
| `MPG2` (`V4L2_PIX_FMT_MPEG2`) | MPEG-2 elementary stream |

Set the output format with `VIDIOC_S_FMT` on `V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE`
before queuing any input.

CAPTURE (decoded output) queue: see the table in
[`README.md`](README.md).  Probe with `VIDIOC_ENUM_FMT` --
both NM12 (multi-plane NV12) and NV12 (single-plane NV12) are
advertised on every supported SoC.

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_MPEG2;  /* or V4L2_PIX_FMT_MPEG1 */
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_width and {min,max}_height give the
 * bounds.  Maximum is 1920x1080 across all supported SoCs;
 * minimum is 64x64. */
```

## Custom V4L2 controls

The MPEG-1 / MPEG-2 path exposes no codec-specific controls.
The standard V4L2 stateful decoder ioctls and events
(`V4L2_EVENT_SOURCE_CHANGE`, `V4L2_DEC_CMD_STOP`,
`V4L2_BUF_FLAG_LAST`) are all that userspace needs.

## Reference applications

### GStreamer

```sh
gst-launch-1.0 \
    filesrc location=input.mpg ! mpegvideoparse \
    ! v4l2mpeg2dec \
    ! videoconvert ! autovideosink
```

For MPEG-1:

```sh
gst-launch-1.0 \
    filesrc location=input.mpg ! mpegvideoparse \
    ! v4l2mpeg2dec \   # the same element handles MPEG-1 streams
    ! videoconvert ! autovideosink
```

`v4l2mpeg2dec` handles both MPEG-1 and MPEG-2 input -- the parser
auto-detects.  When the GStreamer build does not include
`v4l2mpeg2dec`, fall back to FFmpeg.

For low-latency capture-to-disk:

```sh
gst-launch-1.0 \
    filesrc location=input.mpg ! mpegvideoparse ! v4l2mpeg2dec \
    ! 'video/x-raw,format=NV12' \
    ! filesink location=output.nv12
```

### FFmpeg

```sh
# MPEG-2
ffmpeg -c:v mpeg2_v4l2m2m -i input.mpg -f null -

# MPEG-1
ffmpeg -c:v mpeg1_v4l2m2m -i input.mpg -f null -
```

For a YUV dump:

```sh
ffmpeg -c:v mpeg2_v4l2m2m -i input.mpg \
    -f rawvideo -pix_fmt nv12 output.nv12
```

### Minimal V4L2 decode loop

A complete C example is out of scope here; refer to the upstream
kernel sample at
[`Documentation/userspace-api/media/v4l/dev-decoder.rst`](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-decoder.html#decoding).
The Amlogic decoder follows that contract verbatim, including the
`V4L2_EVENT_SOURCE_CHANGE` and `V4L2_DEC_CMD_STOP` flow.

## Sequence boundaries

A stream with multiple back-to-back MPEG sequences (each ended
with a `sequence_end_code`, optionally followed by a new sequence
with a different resolution) is supported.  When the resolution
changes mid-stream the decoder fires `V4L2_EVENT_SOURCE_CHANGE`;
userspace re-allocates CAPTURE buffers and the decode resumes.

## End of stream

`V4L2_DEC_CMD_STOP` flushes any internally buffered frames (the
B-frame display-reorder buffer) and signals end-of-stream via a
final CAPTURE buffer with `V4L2_BUF_FLAG_LAST`.  Every queued
input frame is delivered to userspace before the EOS marker.

## Error reporting

The decoder reports recoverable errors via `V4L2_BUF_FLAG_ERROR`
on dequeued CAPTURE buffers.  Hard failures (firmware watchdog,
unsupported profile) are reported via the `V4L2_EVENT_*` event
queue and `errno` on `VIDIOC_QBUF` / `VIDIOC_DQBUF`.

Corrupt or partially-truncated input is handled gracefully: the
decoder skips invalid macroblock rows, decodes what it can, and
signals `EOS` cleanly when input runs out.

## Decode contract summary

- Bit depth: 8-bit 4:2:0.  4:2:2 / 4:4:4 input is rejected at
  `VIDIOC_S_FMT`.
- Maximum resolution: 1920x1080 on every supported SoC
  (MPEG-2 Main Profile ceiling).
- One decode session at a time per SoC (single hardware core).
- B-frame display reorder: every input frame is delivered to
  userspace at end-of-stream; no frames are lost on the EOS
  flush path.
