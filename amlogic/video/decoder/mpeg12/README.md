# MPEG-1 / MPEG-2 Video Decoder

Hardware-accelerated MPEG-1 and MPEG-2 video decode on Amlogic SoCs.
Exposed to userspace as a standard V4L2 stateful memory-to-memory
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

- **MPEG-1**: ISO/IEC 11172-2 (Constrained Parameter Set is the
  common compliance target; the decoder accepts beyond-CPS streams
  up to the resolution ceiling below).
- **MPEG-2**: ISO/IEC 13818-2 Main Profile.  Simple Profile is a
  strict subset of Main Profile and decodes on the same path.
  High Profile (4:2:2 / >1080p) is not supported by the silicon.
- **Frame types**: I / P / B in any pattern.  GOP structure is
  unconstrained -- the decoder handles I-only, IP, IPB, and
  variable-length GOPs.
- **Bit depth / chroma**: 8-bit 4:2:0 only.  This is the limit of
  the MPEG-2 Main Profile specification.
- **Field / frame coding**: progressive and interlaced (TFF / BFF)
  both decode.  Interlaced output reports the appropriate
  `V4L2_FIELD_INTERLACED_TB` / `V4L2_FIELD_INTERLACED_BT` per
  buffer.
- **Aspect ratio**: parsed from the sequence header and reported
  via `VIDIOC_G_FMT` / `VIDIOC_G_SELECTION`.
- **Multiple sequences**: a stream containing back-to-back MPEG
  sequences (sequence_end_code followed by a fresh sequence header
  with a different resolution) re-initializes cleanly via the
  source-change event flow.

## Resolutions

| SoC | Max width | Max height | Notes |
|-----|-----------|------------|-------|
| S805X | 1920 px | 1080 px | MPEG-2 Main Profile ceiling |
| S905X | 1920 px | 1080 px | Same |
| S905D | 1920 px | 1080 px | Same |
| A311D | 1920 px | 1080 px | Same |
| S905D3 | 1920 px | 1080 px | Same |

The decoder exposes the per-SoC maximum via
`VIDIOC_ENUM_FRAMESIZES` on the OUTPUT (compressed input) queue
at probe time.  Standard MPEG resolutions (CIF 352x288, SIF
352x240, D1 720x480 / 720x576, 720p 1280x720, 1080p 1920x1080) all
decode without firmware adjustment.

## Pixel formats

OUTPUT (compressed input) queue:

| FourCC | Format |
|--------|--------|
| `MPG1` (`V4L2_PIX_FMT_MPEG1`) | MPEG-1 elementary stream |
| `MPG2` (`V4L2_PIX_FMT_MPEG2`) | MPEG-2 elementary stream |

CAPTURE (decoded output) queue:

| FourCC | Format | All boards |
|--------|--------|------------|
| `NM12` (`V4L2_PIX_FMT_NV12M`) | 2-plane NV12, hardware native | Yes |
| `NV12` (`V4L2_PIX_FMT_NV12`) | 1-plane NV12 | Yes |

## Bitrate

No firmware-imposed bitrate ceiling at 1080p on any supported SoC.
Real-world MPEG-2 content (DVD 6-8 Mbps, ATSC 19 Mbps, BluRay
40 Mbps) decodes at full frame rate.  Synthetic high-bitrate
streams (50 Mbps and above at 1080p) decode bit-correctly without
overflow.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-second
and PSNR on each supported SoC across the standard resolution
range.  All supported SoCs decode 30 fps real-time MPEG-1 and
MPEG-2 content at every resolution from CIF up to 1080p.

## V4L2 API

See [`api.md`](api.md) for the V4L2 control surface and a minimal
working decode loop in C.

## Multi-instance

Single hardware decoder core per SoC -- one decode session at a
time.  Concurrent V4L2 opens are accepted, but the second session
blocks until the first completes.

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-decoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/dev-mem2mem.rst`
- ITU-T H.262 / ISO/IEC 13818-2 (MPEG-2 Video)
- ISO/IEC 11172-2 (MPEG-1 Video)
