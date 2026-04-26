# MJPEG Video Decoder

Hardware-accelerated Motion-JPEG (and JPEG still) video decode on
Amlogic SoCs.  Exposed to userspace as a standard V4L2 stateful
memory-to-memory decoder at `/dev/videoN`.

## Supported boards

| SoC | Boards |
|-----|--------|
| S805X | AML-S805X-AC (La Frite), AML-S805X-AC-V2 (Das Frite) |
| S905X | AML-S905X-CC (Le Potato), AML-S905X-CC-V2 (Sweet Potato), AML-S905X-CC-V3 (Das Potato) |
| S905D | AML-S905D-PC (Tartiflette) |
| A311D | AML-A311D-CC (Alta), AML-A311D-CM (Alta CM) |
| S905D3 | AML-S905D3-CC (Solitude), AML-S905D3-CM (Solitude CM) |

## Capabilities

- **Bit depth / chroma**: 8-bit 4:2:0 only.  Non-4:2:0 input
  (yuvj422p / yuvj444p, grayscale) is rejected at queue time with
  `-EINVAL` before reaching the firmware.
- **Output**: 3-plane I420 (`V4L2_PIX_FMT_YUV420M`).  This is the
  only output format -- the post-scaler block always writes
  3-plane I420.
- **Frame types**: every frame is an independent JPEG (intra-only).
  No prediction across frames.  Random access at any frame
  boundary.
- **JPEG markers**: SOF0 / SOF1 (baseline / extended baseline) with
  three components.  SOI / EOI handled per-frame; restart markers
  honored.
- **Quality range**: any quantization table -- the firmware
  processes whatever the JPEG header carries, from extremely-coarse
  (`-q:v 50`) to near-lossless (`-q:v 1` / `-q:v 2`).
- **JPEG stills**: the same device decodes single-frame JPEG
  images (still photos).  Use the same V4L2 fourcc and feed one
  frame at a time.

## Resolutions

| SoC | Max width | Max height | Notes |
|-----|-----------|------------|-------|
| S805X | 8192 px | 2048 px | 16.7 megapixels per frame |
| S905X | 8192 px | 2048 px | Same firmware; same limit |
| S905D | 8192 px | 2048 px | Same firmware; same limit |
| A311D | 8192 px | 2048 px | Same firmware; same limit |
| S905D3 | 8192 px | 2048 px | Same firmware; same limit |

The width/height ceiling is a firmware constraint -- the JPEG
specification has no resolution limit, but the on-chip decoder
caps each axis at the values above.  `VIDIOC_ENUM_FRAMESIZES`
reports the ceiling at probe time.

Supported common formats:

- Up to 4K-wide letterbox: 3840x2048, 4096x2048, 8192x2048
- Standard photo / video: 320x240, 720x480, 1280x720, 1920x1080,
  2560x1440
- 4K UHD (3840x2160) is **not** supported -- the height ceiling
  is 2048 px.  Resize to 3840x2048 or process tile-by-tile if
  16:9 4K output is required.

## Pixel formats

OUTPUT (compressed input) queue:

| FourCC | Format |
|--------|--------|
| `MJPG` (`V4L2_PIX_FMT_MJPEG`) | Motion-JPEG / JFIF / single JPEG |

CAPTURE (decoded output) queue:

| FourCC | Format |
|--------|--------|
| `YM12` (`V4L2_PIX_FMT_YUV420M`) | 3-plane I420 |

NV12 / NV21 capture is not advertised on this device -- the
post-scaler hardware writes I420 only.  Userspace that needs NV12
output must convert via `videoconvert` (GStreamer) or
`-vf format=nv12` (FFmpeg).

## Bitrate

MJPEG bitrate is determined entirely by per-frame quality and
resolution.  No firmware-imposed ceiling.  Tested at 320x240
quality 1 (~100 KB/frame) up to 1920x1080 near-lossless
(~120 KB/frame); all quality levels decode correctly.  The
firmware's input FIFO is fed asynchronously, so even high-bitrate
frames at 200+ Mbps are buffered without overflow.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-second
on each supported SoC across the standard resolution range.

## V4L2 API

The decoder is a standard V4L2 stateful memory-to-memory device.
See [`api.md`](api.md) for the device-discovery snippet, the
GStreamer / FFmpeg invocations, and JPEG-still notes.

## Multi-instance

Single hardware decoder core per SoC -- one decode session at a
time.  Concurrent V4L2 opens are accepted, but the second session
blocks until the first completes.

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-decoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/dev-mem2mem.rst`
- ITU-T T.81 / ISO/IEC 10918-1 (JPEG)

## Relationship to JPEG encoder

JPEG / still-image encode is a separate hardware path on the same
SoC.  The encoder is a different `/dev/videoN` node; see the
companion encoder page for capability and API details.  This
decoder also accepts single-JPEG input on the same fourcc -- there
is no distinct still-decode path.
