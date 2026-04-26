# JPEG Encoder

Hardware-accelerated baseline JPEG encode on Amlogic SoCs.
Exposed to userspace as a standard V4L2 stateful memory-to-memory
encoder at `/dev/videoN`.

## Supported boards

| SoC | Boards |
|-----|--------|
| S805X  | AML-S805X-AC (La Frite), AML-S805X-AC-V2 (Das Frite) |
| S905X  | AML-S905X-CC (Le Potato), AML-S905X-CC-V2 (Sweet Potato), AML-S905X-CC-V3 (Das Potato) |
| S905D  | AML-S905D-PC (Tartiflette) |
| A311D  | AML-A311D-CC (Alta), AML-A311D-CM (Alta CM) |
| S905D3 | AML-S905D3-CC (Solitude), AML-S905D3-CM (Solitude CM) |

## Capabilities

- **Container**: JFIF (JPEG File Interchange Format).
- **Profile**: Baseline DCT (SOF0).
- **Bit depth**: 8 bits per component.
- **Chroma subsampling**: 4:2:0, 4:2:2, and 4:4:4 -- selected
  implicitly by the input pixel format.
- **Quality**: 1 .. 100, default 85, via
  `V4L2_CID_JPEG_COMPRESSION_QUALITY`.  Standard IJG quality
  scaling derives the quantisation tables from the JPEG spec
  Annex K.1 (luma) and K.2 (chroma) base tables.
- **Custom quantisation tables**: per-component luma and chroma
  tables can be loaded via the V4L2 compound controls
  (`V4L2_CID_JPEG_LUMA_QUANTIZATION` /
  `V4L2_CID_JPEG_CHROMA_QUANTIZATION`).
- **Standard Huffman tables**: JPEG spec Annex K, hardcoded by
  the driver (no API to override).
- **Output**: complete JFIF file (SOI + DQT + DHT + SOF0 + SOS +
  scan data + EOI) per encoded frame.

## Resolutions

| SoC | Min width | Min height | Max width | Max height |
|-----|-----------|------------|-----------|------------|
| All supported | 160 px | 128 px | 1920 px | 1088 px |

Width and height step is 16 pixels (MCU-aligned).  Non-aligned
heights are rounded up at `VIDIOC_S_FMT` (1080 -> 1088); pad
the source to the aligned height before queueing the frame.

## Pixel formats

OUTPUT (uncompressed input) queue:

| FourCC | Format | JPEG output subsampling |
|--------|--------|------------------------|
| `NV12` (`V4L2_PIX_FMT_NV12`)    | 1-plane Y/UV 4:2:0 | 4:2:0 |
| `NV21` (`V4L2_PIX_FMT_NV21`)    | 1-plane Y/VU 4:2:0 | 4:2:0 |
| `NM12` (`V4L2_PIX_FMT_NV12M`)   | 2-plane NV12 | 4:2:0 |
| `NM21` (`V4L2_PIX_FMT_NV21M`)   | 2-plane NV21 | 4:2:0 |
| `YUYV` (`V4L2_PIX_FMT_YUYV`)    | 1-plane 4:2:2 packed | 4:2:2 |
| `RGB3` (`V4L2_PIX_FMT_RGB24`)   | 24-bit RGB | 4:4:4 |
| `AR24` (`V4L2_PIX_FMT_ABGR32`)  | 32-bit BGRA | 4:4:4 |

Stride alignment: 32 bytes.  RGB inputs are converted to YCbCr
by the hardware before encode.

CAPTURE (compressed output) queue:

| FourCC | Format |
|--------|--------|
| `JPEG` (`V4L2_PIX_FMT_JPEG`) | JFIF baseline DCT |

The same device node also advertises `H264`
(`V4L2_PIX_FMT_H264`) on CAPTURE for H.264 encode -- see
[`../h264/`](../h264/).  A given session selects one CAPTURE
format and encodes that codec until close.

## Output format

Each `V4L2_BUF_FLAG_LAST` CAPTURE buffer carries one complete
JFIF file:

- SOI marker
- Two DQT segments (luma + chroma, scaled from the requested
  quality)
- Four DHT segments (JPEG spec Annex K standard tables)
- SOF0 segment with the encoded resolution and chroma sampling
  factors
- SOS segment introducing the scan
- Hardware-generated VLC scan data
- EOI marker

No Exif metadata, thumbnails, or restart markers are emitted.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-
second and average output size on each supported SoC across
quality and resolution sweeps, plus per-pixel-format throughput.

## V4L2 API

See [`api.md`](api.md) for the V4L2 control surface, device
discovery, and reference encode pipelines.

## Multi-instance

Single hardware encoder core per SoC.  A second `open()` of the
device returns `EBUSY` until the first session closes -- this
covers both H.264 and JPEG modes (they share the same hardware).

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-encoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/pixfmt-compressed.rst`
- ITU-T T.81 / ISO/IEC 10918-1
