# H.264 Encoder API

The H.264 encoder is a standard V4L2 stateful memory-to-memory
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
`pixelformat = V4L2_PIX_FMT_H264` on the CAPTURE queue for H.264
encode.

GStreamer's `v4l2h264enc` element discovers the device automatically.
FFmpeg's `h264_v4l2m2m` encoder does the same when invoked with
`-c:v h264_v4l2m2m`.

## Pixel formats

OUTPUT (uncompressed input) and CAPTURE (compressed output)
formats are listed in [`README.md`](README.md).  Probe with
`VIDIOC_ENUM_FMT` -- input format support is identical across
all supported SoCs.

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_H264;
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_{width,height} give the SoC-specific
 * bounds.  Width and height step is 16 pixels. */
```

## V4L2 controls

Standard upstream V4L2 video-encode controls.  Discover the full
set with `v4l2-ctl --list-ctrls`.

### Stream

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_H264_PROFILE` | menu | Baseline, Main | per SoC |
| `V4L2_CID_MPEG_VIDEO_H264_LEVEL`   | menu | 1.0 .. 5.1 | 4.0 |

The driver advertises Baseline + Main; the actual encoded profile
is fixed by the SoC's encoder firmware (Baseline on S805X / S905X /
S905D, Main on A311D / S905D3).  The `level_idc` field of the
output SPS reflects the requested level value.

### Rate control

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_BITRATE_MODE`     | menu | VBR, CBR | VBR |
| `V4L2_CID_MPEG_VIDEO_BITRATE`          | int  | 0 .. 20 Mbps | 0 |
| `V4L2_CID_MPEG_VIDEO_FRAME_RC_ENABLE`  | bool | 0..1 | 0 |
| `V4L2_CID_MPEG_VIDEO_H264_MIN_QP`      | int  | 0 .. 51 | 26 |
| `V4L2_CID_MPEG_VIDEO_H264_MAX_QP`      | int  | 0 .. 51 | 51 |

A bitrate of zero selects the encoder's internal default rate.
The min / max QP controls set the per-frame QP value used by the
encoder.

### GOP

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_H264_I_PERIOD`   | int    | 1 .. 300 | 12 |
| `V4L2_CID_MPEG_VIDEO_FORCE_KEY_FRAME` | button | -- | -- |

### Per-fd semantics

V4L2 controls live on the file descriptor that opened them.
Setting a control via `v4l2-ctl --set-ctrl` and then re-opening
the device with `gst-launch-1.0` or `ffmpeg` does not preserve
the value.  Use `extra-controls=` on the GStreamer element, or
call `VIDIOC_S_EXT_CTRLS` from your application before queueing
the first OUTPUT buffer.

## GStreamer reference pipeline

Encode raw NV12 to a 4 Mbps H.264 file:

```sh
gst-launch-1.0 \
    videotestsrc num-buffers=300 \
    ! 'video/x-raw,format=NV12,width=1280,height=720,framerate=30/1' \
    ! v4l2h264enc extra-controls="encode,video_bitrate=4000000" \
    ! 'video/x-h264,profile=main,stream-format=byte-stream' \
    ! filesink location=output.h264
```

## FFmpeg reference invocation

Encode raw NV12 to H.264 at 4 Mbps:

```sh
ffmpeg -f rawvideo -pix_fmt nv12 -s 1280x720 -r 30 -i input.nv12 \
    -c:v h264_v4l2m2m -b:v 4M \
    -num_output_buffers 16 -num_capture_buffers 16 \
    output.h264
```

Always specify `-b:v` explicitly when measuring quality -- the
FFmpeg default of 200 kbps is below the encoder's effective
bitrate floor at 720p and above.

## Multi-instance

Single hardware encoder core.  A second `open()` of the device
returns `EBUSY` until the first session closes.  Concurrent
H.264 + JPEG sessions are not possible -- both modes share the
same hardware.

## Encode contract summary

- One `meson-video-encoder` device per SoC.
- 8-bit 4:2:0 input.  RGB and YUYV inputs are converted to NV12
  by the hardware before encode.
- Width and height step is 16 pixels.
- SPS + PPS prepended at stream start; bitstream is Annex B.
- Profile is fixed by the SoC's firmware (Baseline on GXL-family
  boards, Main on A311D and S905D3).
