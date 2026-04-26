# HEVC Video Encoder

Hardware-accelerated H.265 / HEVC video encode on Amlogic SoCs.
Exposed to userspace as a standard V4L2 stateful memory-to-memory
encoder at `/dev/videoN`.

## Supported boards

| SoC | Boards |
|-----|--------|
| A311D | AML-A311D-CC (Alta), AML-A311D-CM (Alta CM) |
| S905D3 | AML-S905D3-CC (Solitude), AML-S905D3-CM (Solitude CM) |

## Capabilities

- **Profiles**: Main, Main Still Picture, Main 10.
- **Bit depth / chroma**: 8-bit 4:2:0 (Main, Main Still Picture);
  10-bit 4:2:0 (Main 10) via `V4L2_PIX_FMT_P010` input.
- **Levels**: 1.0 through 5.1 selectable; default level 4.0.
- **Tier**: Main and High.
- **Frame types**: I / P / B in any pattern.  GOP up to 300,
  up to 3 B-frames between reference frames.
- **Rate control**: bitrate-targeting (CBR-style with HVS
  perceptual QP) and constant-quantization (CQP).  Bitrate up
  to 60 Mbps.  Per-frame I/P/B QP, min/max QP clamps, HRD VBV
  buffer up to 3000 ms.
- **Quality controls**: deblocking filter (mode + beta and Tc
  offsets), Y/Cb/Cr noise reduction, Cb/Cr chroma QP offset,
  constrained intra prediction, wavefront parallel processing,
  configurable maximum coding-unit partition depth, custom
  scaling list (quantization matrix).
- **Region of interest**: per-CTU importance map and absolute
  per-CTU QP map applied each frame.
- **Slices**: dependent and independent slices, configurable
  CTUs-per-slice.
- **Refresh**: cyclic intra refresh (no I-frames) and periodic
  CRA / IDR refresh; on-demand IDR via
  `V4L2_CID_MPEG_VIDEO_FORCE_KEY_FRAME`.
- **Reference structure**: long-term reference frames,
  generalised P / B reference shape, custom GOP with per-
  position lambda.
- **Stream metadata**: VUI (colour space, sample-aspect-ratio,
  HRD parameters), prefix and suffix SEI NAL injection (HDR10
  mastering metadata, content-light-level information, closed
  captions, timecode).
- **Hardware transforms**: 0 / 90 / 180 / 270 degree rotation
  and horizontal / vertical flip on the input frame.  90 / 270
  swap output width and height.
- **Lossless encoding**: bit-exact reconstruction at the cost
  of bitrate.
- **Headers**: VPS / SPS / PPS prepended to every IDR.  Output
  is Annex B byte-stream (4-byte start codes).
- **Runtime power management**: per-session sleep / wake with
  5-second autosuspend.

## Resolutions

| SoC | Min width | Min height | Max width | Max height |
|-----|-----------|------------|-----------|------------|
| A311D  | 256 px | 128 px | 4096 px | 2304 px |
| S905D3 | 256 px | 128 px | 4096 px | 2304 px |

Width and height are aligned up to the nearest 8-pixel multiple
at `VIDIOC_S_FMT`.

## Pixel formats

OUTPUT (uncompressed input) queue:

| FourCC | Format | Notes |
|--------|--------|-------|
| `NV12` (`V4L2_PIX_FMT_NV12`)     | 1-plane Y/UV 4:2:0 | Default, fastest |
| `NV21` (`V4L2_PIX_FMT_NV21`)     | 1-plane Y/VU 4:2:0 | |
| `YU12` (`V4L2_PIX_FMT_YUV420`)   | 1-plane planar I420 | Converted to semi-planar internally |
| `NM12` (`V4L2_PIX_FMT_NV12M`)    | 2-plane NV12 | |
| `NM21` (`V4L2_PIX_FMT_NV21M`)    | 2-plane NV21 | |
| `YM12` (`V4L2_PIX_FMT_YUV420M`)  | 3-plane I420 | |
| `P010` (`V4L2_PIX_FMT_P010`)     | 1-plane Y/UV 4:2:0, 10-bit | Main 10 only |

Stride alignment: 32 bytes.

CAPTURE (compressed output) queue:

| FourCC | Format |
|--------|--------|
| `HEVC` (`V4L2_PIX_FMT_HEVC`) | H.265 / HEVC Annex B byte-stream |

Default CAPTURE buffer size is 2 MB.  Override via
`VIDIOC_S_FMT.fmt.pix_mp.plane_fmt[0].sizeimage` for high-bitrate
or high-resolution streams.

## Bitrate

| Mode | Trigger | Behaviour |
|------|---------|-----------|
| Bitrate-targeting | `V4L2_CID_MPEG_VIDEO_BITRATE > 0` | Rate control with HVS perceptual QP, CTU-level RC |
| CQP | `V4L2_CID_MPEG_VIDEO_BITRATE = 0` | Constant QP, set via `V4L2_CID_MPEG_VIDEO_HEVC_I_FRAME_QP` |

Bitrate range: up to 60 Mbps.  Use the bitrate-targeting mode
for real-time pipelines and CQP for offline quality-targeted
encoding.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-second
on each supported SoC at 320x240, 720p, 1080p, 1440p, and 4K UHD,
plus quality (PSNR) at multiple bitrates.

## V4L2 API

See [`api.md`](api.md) for the V4L2 control surface, device
discovery, and reference encode pipelines.

## Multi-instance

Up to 3 concurrent encode sessions per SoC at 480p, and up to 2
concurrent sessions at 720p.  Single hardware encoder core --
sessions are time-sliced.

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-encoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/dev-mem2mem.rst`
- ITU-T H.265 / ISO/IEC 23008-2
