# H.264 Video Encoder

Hardware-accelerated H.264 / MPEG-4 AVC video encode on Amlogic
SoCs.  Exposed to userspace as a standard V4L2 stateful memory-
to-memory encoder at `/dev/videoN`.

## Supported boards

| SoC | Boards |
|-----|--------|
| S805X  | AML-S805X-AC (La Frite), AML-S805X-AC-V2 (Das Frite) |
| S905X  | AML-S905X-CC (Le Potato), AML-S905X-CC-V2 (Sweet Potato), AML-S905X-CC-V3 (Das Potato) |
| S905D  | AML-S905D-PC (Tartiflette) |
| A311D  | AML-A311D-CC (Alta), AML-A311D-CM (Alta CM) |
| S905D3 | AML-S905D3-CC (Solitude), AML-S905D3-CM (Solitude CM) |

## Capabilities

- **Profiles**: Baseline (CAVLC entropy coding) on S805X / S905X /
  S905D; Main (CABAC entropy coding) on A311D and S905D3.  The
  encoder selects the entropy-coding firmware by SoC; profile is
  driven by capability rather than a runtime knob.
- **Bit depth / chroma**: 8-bit 4:2:0 only.
- **Levels**: 1.0 through 5.1 selectable via
  `V4L2_CID_MPEG_VIDEO_H264_LEVEL`; the value is written to the
  SPS `level_idc`.  Default level 4.0.
- **Frame types**: I and P frames in any pattern.  GOP up to 300.
- **Rate control**: VBR (default) and CBR via
  `V4L2_CID_MPEG_VIDEO_BITRATE_MODE`.  Target bitrate up to 20
  Mbps via `V4L2_CID_MPEG_VIDEO_BITRATE`.  Per-frame QP via
  `V4L2_CID_MPEG_VIDEO_H264_MIN_QP` /
  `V4L2_CID_MPEG_VIDEO_H264_MAX_QP`.
- **GOP control**: intra period via
  `V4L2_CID_MPEG_VIDEO_H264_I_PERIOD`; on-demand IDR via
  `V4L2_CID_MPEG_VIDEO_FORCE_KEY_FRAME`.
- **Noise reduction**: built-in 3-D NR enabled by default
  (configurable via the `nr_mode` module parameter).
- **Headers**: SPS + PPS NAL units prepended at stream start;
  output is Annex B byte-stream (4-byte start codes).

## Resolutions

| SoC | Min width | Min height | Max width | Max height |
|-----|-----------|------------|-----------|------------|
| S805X  | 160 px | 128 px | 1920 px | 1088 px |
| S905X  | 160 px | 128 px | 1920 px | 1088 px |
| S905D  | 160 px | 128 px | 1920 px | 1088 px |
| A311D  | 160 px | 128 px | 2048 px | 2176 px |
| S905D3 | 160 px | 128 px | 2048 px | 2176 px |

Width and height step is 16 pixels (MB-aligned).  Non-aligned
dimensions are rounded up by `VIDIOC_S_FMT`.  S805X 512 MB
boards have a smaller CMA pool; 1080p encode requires the 1 GB
or larger configuration.

## Pixel formats

OUTPUT (uncompressed input) queue:

| FourCC | Format | Notes |
|--------|--------|-------|
| `NV12` (`V4L2_PIX_FMT_NV12`)    | 1-plane Y/UV 4:2:0 | Default |
| `NV21` (`V4L2_PIX_FMT_NV21`)    | 1-plane Y/VU 4:2:0 | |
| `NM12` (`V4L2_PIX_FMT_NV12M`)   | 2-plane NV12 | |
| `NM21` (`V4L2_PIX_FMT_NV21M`)   | 2-plane NV21 | |
| `YM12` (`V4L2_PIX_FMT_YUV420M`) | 3-plane I420 | |
| `YUYV` (`V4L2_PIX_FMT_YUYV`)    | 1-plane 4:2:2 packed | Downsampled to 4:2:0 by hardware |
| `RGB3` (`V4L2_PIX_FMT_RGB24`)   | 24-bit RGB | RGB-to-YCbCr converted by hardware |
| `AR24` (`V4L2_PIX_FMT_ABGR32`)  | 32-bit BGRA | Linear DMA, fastest for software-rendered sources |

Stride alignment: 32 bytes.

CAPTURE (compressed output) queue:

| FourCC | Format |
|--------|--------|
| `H264` (`V4L2_PIX_FMT_H264`) | H.264 Annex B byte-stream |

The same device node also advertises `JPEG`
(`V4L2_PIX_FMT_JPEG`) on CAPTURE for JPEG encode -- see
[`../jpeg/`](../jpeg/).  A given session selects one CAPTURE
format and uses the encoder for that codec until close.

## Bitrate

Target bitrate is set via `V4L2_CID_MPEG_VIDEO_BITRATE`
(0 .. 20 Mbps; 0 selects the encoder's default rate).  Mode is
selected via `V4L2_CID_MPEG_VIDEO_BITRATE_MODE`:

| Mode | V4L2 value | Behaviour |
|------|-----------|-----------|
| VBR  | `V4L2_MPEG_VIDEO_BITRATE_MODE_VBR` (default) | Frame-level rate control toward the target |
| CBR  | `V4L2_MPEG_VIDEO_BITRATE_MODE_CBR` | Bitrate-targeting with tighter short-term variation |

The minimum effective bitrate scales with resolution and the
encoder's QP floor (default QP 26).  Targets below the per-
resolution floor encode at the floor's bitrate.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-
second on each supported SoC at QCIF, 320x240, 640x480, 720p,
and 1080p, plus per-pixel-format throughput and quality.

## V4L2 API

See [`api.md`](api.md) for the V4L2 control surface, device
discovery, and reference encode pipelines.

## Multi-instance

Single hardware encoder core per SoC.  A second `open()` of the
device returns `EBUSY` until the first session closes -- this
covers both H.264 and JPEG modes (they share the same hardware).

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-encoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/dev-mem2mem.rst`
- ITU-T H.264 / ISO/IEC 14496-10
