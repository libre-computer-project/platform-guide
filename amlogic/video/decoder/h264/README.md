# H.264 Video Decoder

Hardware-accelerated H.264 / MPEG-4 AVC video decode on Amlogic
SoCs.  Exposed to userspace as a standard V4L2 stateful memory-to-memory
decoder at `/dev/videoN`.

## Supported boards

| SoC | Boards |
|-----|--------|
| S805X | AML-S805X-AC (La Frite), AML-S805X-AC-V2 (Das Frite) |
| S905X | AML-S905X-CC (Le Potato), AML-S905X-CC-V2 (Sweet Potato), AML-S905X-CC-V3 (Das Potato) |
| S905D | AML-S905D-PC (Tartiflette) |
| A311D | AML-A311D-CC (Alta), AML-A311D-CM (Alta CM) |
| S905D3 | AML-S905D3-CC (Solitude), AML-S905D3-CM (Solitude CM) |

## Capabilities

- **Profiles**: Constrained Baseline (66), Baseline (66), Main (77),
  High (100).  CABAC and CAVLC, B-frames, weighted prediction.
- **Bit depth / chroma**: 8-bit 4:2:0 only.  10-bit (High 10),
  4:2:2 (High 4:2:2), and 4:4:4 (High 4:4:4 Predictive) profiles
  are rejected at `VIDIOC_S_FMT` -- the silicon has no 10-bit or
  non-4:2:0 output path.
- **Frame types**: I / P / B in any pattern.  GOP up to 600,
  reference frames up to 16.
- **Interlaced content**: TFF, BFF, MBAFF, PAFF all decode
  correctly.  480i / 576i / 1080i tested.
- **Multi-View Coding (MVC, Stereo High profile 118/128)**:
  dual-view stereo 3D decode at the 640x480 reference resolution.
  See [`README-mvc.md`](README-mvc.md).
- **Scalable Video Coding (SVC)**: not supported (no hardware).

## Resolutions

| SoC | Max width | Max height | Notes |
|-----|-----------|------------|-------|
| S805X | 1920 px | 1088 px | Firmware caps high-resolution decode at 1080p |
| S905X | 4096 px | 4096 px | DCI 4K via SPS overflow recovery |
| S905D | 4096 px | 4096 px | Same family as S905X |
| A311D | 4096 px | 4096 px | DCI 4K, portrait 2304x4096 |
| S905D3 | 4096 px | 4096 px | DCI 4K, portrait 2304x4096; min width 258 px |

The resolution matrix is non-monotonic at the upper edge -- some
combinations above 8400 macroblocks (e.g. 4080x4080) are rejected
by firmware.  Refer to [`benchmarks.md`](benchmarks.md) for the
specific resolutions exercised on each platform.

## Pixel formats

The decoder advertises one or more of these formats on the CAPTURE
queue depending on the SoC:

| FourCC | Format | All boards |
|--------|--------|------------|
| `NM12` (`V4L2_PIX_FMT_NV12M`) | 2-plane NV12, hardware native | Yes |
| `NV12` (`V4L2_PIX_FMT_NV12`) | 1-plane NV12 | Yes |
| `NM21` (`V4L2_PIX_FMT_NV21M`) | 2-plane NV21 | A311D only |
| `NV21` (`V4L2_PIX_FMT_NV21`) | 1-plane NV21 | A311D only |
| `YM12` (`V4L2_PIX_FMT_YUV420M`) | 3-plane I420 | A311D only |

NV21 and 3-plane YUV420M require a canvas-controller feature that
S805X / S905X / S905D / S905D3 silicon does not have.  On those
SoCs the decoder advertises NM12 + NV12 only.

## Bitrate

No firmware-imposed bitrate ceiling at 1080p on any supported SoC.
Real-world H.264 content (5-40 Mbps 1080p, 15-100 Mbps 4K) decodes
at full frame rate.  Synthetic test streams above 200 Mbps may
trigger an esparser overflow and partial-frame loss; this is a
buffer-pool sizing limit, not a silicon limit.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-second
on each supported SoC at 320x240, 720p, 1080p, 1080i, 1440p, and
4K UHD.

## V4L2 API

See [`api.md`](api.md) for the V4L2 control surface and a minimal
working decode loop in C.

## Multi-instance

Single hardware decoder core per SoC -- one decode session at a time.
Concurrent V4L2 opens are accepted, but the second session blocks
until the first completes.

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-decoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/dev-mem2mem.rst`
- ITU-T H.264 / ISO/IEC 14496-10
