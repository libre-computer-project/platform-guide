# HEVC Encoder API

The HEVC encoder is a standard V4L2 stateful memory-to-memory
device.  Userspace works it via the upstream
[V4L2 stateful encoder API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-encoder.html).

## Device discovery

The encoder advertises itself in sysfs as `wave420l-enc`.  The
`/dev/videoN` index varies by SoC and boot order -- always
discover by name, never hardcode the index.

```c
/* Find the HEVC encoder by sysfs name. */
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
    if (strcmp(name, "wave420l-enc") == 0) {
        snprintf(dev, sizeof(dev), "/dev/%s", e->d_name);
        fd = open(dev, O_RDWR);
        break;
    }
}
closedir(d);
```

GStreamer's `v4l2h265enc` element discovers the device automatically.
FFmpeg's `hevc_v4l2m2m` encoder does the same when invoked with
`-c:v hevc_v4l2m2m`.

## Pixel formats

OUTPUT (uncompressed input) and CAPTURE (compressed output)
formats are listed in [`README.md`](README.md).

## Resolution probing

```c
struct v4l2_frmsizeenum fs = { 0 };
fs.pixel_format = V4L2_PIX_FMT_HEVC;
fs.index = 0;
ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &fs);
/* fs.stepwise.{min,max}_{width,height} give the SoC-specific
 * bounds.  Width and height step is 8 pixels. */
```

## V4L2 controls

The driver exposes the upstream HEVC stateful-encoder control
set plus the standard V4L2 video-encode controls.  Discover the
full set with `v4l2-ctl --list-ctrls`.

### Stream

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_HEVC_PROFILE` | menu | Main, Main Still Picture, Main 10 | Main |
| `V4L2_CID_MPEG_VIDEO_HEVC_LEVEL`   | menu | 1.0 .. 5.1 | 4.0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_TIER`    | menu | Main, High | Main |

### Rate control

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_BITRATE_MODE`     | menu | VBR, CBR, CQ | VBR |
| `V4L2_CID_MPEG_VIDEO_BITRATE`          | int  | 0 .. 60 Mbps | 4 Mbps |
| `V4L2_CID_MPEG_VIDEO_FRAME_RC_ENABLE`  | bool | 0..1 | 1 |
| `V4L2_CID_MPEG_VIDEO_HEVC_I_FRAME_QP`  | int  | 0 .. 51 | 20 |
| `V4L2_CID_MPEG_VIDEO_HEVC_P_FRAME_QP`  | int  | 0 .. 63 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_B_FRAME_QP`  | int  | 0 .. 63 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_MIN_QP`      | int  | 0 .. 51 | 8 |
| `V4L2_CID_MPEG_VIDEO_HEVC_MAX_QP`      | int  | 0 .. 51 | 51 |
| `V4L2_CID_MPEG_VIDEO_VBV_SIZE`         | int  | 0 .. 3000 | 3000 |

A bitrate of zero selects CQP mode and uses the I-frame QP
control as the fixed quantizer.

### GOP and reference structure

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_GOP_SIZE`             | int  | 1 .. 300 | 30 |
| `V4L2_CID_MPEG_VIDEO_B_FRAMES`             | int  | 0 .. 3 | 0 |
| `V4L2_CID_MPEG_VIDEO_FORCE_KEY_FRAME`      | button | -- | -- |
| `V4L2_CID_MPEG_VIDEO_HEVC_REFRESH_TYPE`    | menu | None, CRA, IDR | CRA |
| `V4L2_CID_MPEG_VIDEO_HEVC_REFRESH_PERIOD`  | int  | 0 .. 2047 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_GENERAL_PB`      | bool | 0..1 | 0 |

### Loop filter and quality

| CID | Type | Range | Default |
|-----|------|-------|---------|
| `V4L2_CID_MPEG_VIDEO_HEVC_LOOP_FILTER_MODE`     | menu | Disabled, Enabled, Disabled at slice boundary | Enabled |
| `V4L2_CID_MPEG_VIDEO_HEVC_LF_BETA_OFFSET_DIV2`  | int  | -6 .. 6 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_LF_TC_OFFSET_DIV2`    | int  | -6 .. 6 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_CHROMA_QP_OFFSET_CB`  | int  | -12 .. 12 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_CHROMA_QP_OFFSET_CR`  | int  | -12 .. 12 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_CONST_INTRA_PRED`     | bool | 0..1 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_WAVEFRONT`            | bool | 0..1 | 0 |
| `V4L2_CID_MPEG_VIDEO_HEVC_MAX_PARTITION_DEPTH`  | int  | 0 .. 2 | 2 |
| `V4L2_CID_MPEG_VIDEO_NOISE_REDUCTION`           | int  | -- | 0 |

Cb / Cr noise-reduction strength and a few hardware-specific
features (per-CTU ROI, per-CTU absolute QP map, custom GOP,
lossless mode, force picture type, hardware rotation, slice
shape, prefix / suffix SEI insertion) are exposed as additional
driver controls under `V4L2_CID_USER_BASE`.  Enumerate with
`v4l2-ctl --list-ctrls`.

### Per-fd semantics

V4L2 controls live on the file descriptor that opened them.
Setting a control via `v4l2-ctl --set-ctrl` and then re-opening
the device with `gst-launch-1.0` or `ffmpeg` does not preserve
the value.  Use `extra-controls=` on the GStreamer element, or
call `VIDIOC_S_EXT_CTRLS` from your application before queueing
the first OUTPUT buffer.

## GStreamer reference pipeline

Encode raw NV12 to a 4 Mbps HEVC file:

```sh
gst-launch-1.0 \
    videotestsrc num-buffers=300 \
    ! 'video/x-raw,format=NV12,width=1280,height=720,framerate=30/1' \
    ! v4l2h265enc extra-controls="encode,video_bitrate=4000000" \
    ! 'video/x-h265,profile=main,stream-format=byte-stream' \
    ! filesink location=output.h265
```

## FFmpeg reference invocation

Encode raw NV12 to HEVC at 4 Mbps:

```sh
ffmpeg -f rawvideo -pix_fmt nv12 -s 1280x720 -r 30 -i input.nv12 \
    -c:v hevc_v4l2m2m -b:v 4M \
    -num_output_buffers 16 -num_capture_buffers 16 \
    output.hevc
```

For constant-quantization mode, set `-b:v 0`; the encoder uses
the I-frame QP control as the fixed quantizer.  Always specify
`-b:v` explicitly for quality measurement -- the FFmpeg default
of 200 kbps is below the encoder's useful range at 720p and
above.

## Region of interest

Per-CTU importance maps (relative QP delta) and absolute per-CTU
QP maps are accepted through driver-specific controls.  Each map
is a buffer sized to ceil(W / 64) x ceil(H / 64) bytes (CTU grid
at 64x64) and is queued through `V4L2_CTRL_TYPE_U8` controls
before each frame.

## SEI insertion

Prefix and suffix SEI NAL units (HDR10 mastering metadata,
content-light-level information, closed captions, timecode) are
inserted into the bitstream at the next IDR.  The driver
allocates a DMA staging buffer for each direction and copies the
caller-supplied SEI payload bytes verbatim.

## Custom GOP

Custom GOP shape (per-position lambda, reference list, picture
type) is accepted through the standard V4L2 HEVC custom-GOP
control payload.  Set `gop_preset_idx = 0` to enable, then queue
the custom-GOP descriptor.

## Multi-instance

Open the device multiple times for parallel sessions.  Up to 3
sessions encode concurrently at 480p, up to 2 sessions at 720p.

## Encode contract summary

- One `wave420l-enc` device per SoC.
- 8-bit 4:2:0 input for Main and Main Still Picture; 10-bit
  4:2:0 input for Main 10.
- Width and height stepped to 8 pixels at `VIDIOC_S_FMT`.
- VPS / SPS / PPS prepended to every IDR; bitstream is Annex B.
