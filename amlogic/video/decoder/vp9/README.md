# VP9 Video Decoder

Hardware-accelerated VP9 video decode on Amlogic SoCs.  Exposed to
userspace as a standard V4L2 stateful memory-to-memory decoder at
`/dev/videoN`.

## Supported boards

| SoC | Boards |
|-----|--------|
| S805X | AML-S805X-AC (La Frite), AML-S805X-AC-V2 (Das Frite) |
| S905X | AML-S905X-CC (Le Potato), AML-S905X-CC-V2 (Sweet Potato), AML-S905X-CC-V3 (Das Potato) |
| S905D | AML-S905D-PC (Tartiflette) |
| A311D | AML-A311D-CC (Alta), AML-A311D-CM (Alta CM) |
| S905D3 | AML-S905D3-CC (Solitude), AML-S905D3-CM (Solitude CM) |

## Capabilities

- **Profile 0** (8-bit 4:2:0): supported on every listed SoC.
- **Profile 2** (10-bit 4:2:0): supported on A311D and S905D3.  Not
  supported on S805X / S905X / S905D (the older silicon has no 10-bit
  data path).  See "Pixel formats" below for how to consume 10-bit
  output.
- **Profile 1 / Profile 3** (4:2:2 and 4:4:4 chroma): rejected at
  `VIDIOC_S_FMT` and at frame-parse time -- the silicon outputs 4:2:0
  only.
- **Frame types**: I / P frames, alt-refs, golden refs, hidden frames.
- **Show_existing_frame**: handled correctly (reference recycle).
- **Tile groups, super-frames**: standard VP9 wire format throughout.
- **Resolution change**: dynamic up to the per-SoC maximum.  Capture
  buffers are renegotiated via the standard V4L2
  `V4L2_EVENT_SOURCE_CHANGE` event.

## Resolutions

| SoC | Max width | Max height | Notes |
|-----|-----------|------------|-------|
| S805X | 1920 px | 1080 px | Limited by 128 MB CMA pool |
| S905X | 1920 px | 1080 px | Limited by 256 MB CMA pool |
| S905D | 1920 px | 1080 px | Limited by 256 MB CMA pool |
| A311D | 5120 px | 2880 px | 5K (5120x2880); 4K+ requires 4 GB RAM (980 MB CMA) |
| S905D3 | 5120 px | 2880 px | 5K (5120x2880); 4K+ requires 4 GB RAM (980 MB CMA) |

6K and 8K Profile 0 streams are accepted by the firmware but require a
CMA pool larger than 980 MB to allocate the per-frame workspace -- they
fail with `-ENOMEM` on stock 4 GB boards.

`VIDIOC_ENUM_FRAMESIZES` reports the SoC-specific upper bound; the
driver computes it at probe time from the available CMA pool.

## Pixel formats

The decoder advertises one or more of these formats on the CAPTURE
queue depending on the SoC:

| FourCC | Format | S805X / S905X / S905D | A311D / S905D3 |
|--------|--------|------------------------|----------------|
| `NM12` (`V4L2_PIX_FMT_NV12M`) | 2-plane NV12 | Yes | Yes |
| `NV12` (`V4L2_PIX_FMT_NV12`) | 1-plane NV12 | Yes | Yes |
| `NM21` (`V4L2_PIX_FMT_NV21M`) | 2-plane NV21 | -- | Yes |
| `NV21` (`V4L2_PIX_FMT_NV21`) | 1-plane NV21 | -- | Yes |
| `YM12` (`V4L2_PIX_FMT_YUV420M`) | 3-plane I420 | -- | Yes |
| `AMS8` (`V4L2_PIX_FMT_MESON_FBC08_SCATTER`) | 8-bit AFBC compressed body | -- | Yes |
| `AMSA` (`V4L2_PIX_FMT_MESON_FBC10_SCATTER`) | 10-bit AFBC compressed body | -- | Yes |

`AMS8` / `AMSA` are the AFBC scatter-gather variants used by the
Amlogic display engine and the GPU.  Selecting these on the CAPTURE
queue routes the decoded frame to the AFBC compressed body directly
and lets the same buffer be scanned out by the VPU without an
intermediate copy.  10-bit Profile 2 content keeps full bit depth in
the AFBC body (the NV12 capture path truncates to 8 bits).

The AFBC modifier on exported buffers is
`DRM_FORMAT_MOD_AMLOGIC_FBC(SCATTER, MEM_SAVING)` for `AMS8` and
`DRM_FORMAT_MOD_AMLOGIC_FBC(SCATTER, 0)` for `AMSA`.

## Bitrate

No firmware-imposed bitrate ceiling at any supported resolution.
Real-world VP9 content (web video at 5-15 Mbps 1080p, YouTube 4K at
20-50 Mbps) decodes at full frame rate on A311D and S905D3.

## Memory

A311D and S905D3 4 GB boards reserve 980 MB of CMA at boot and
support every advertised resolution.  S805X / S905X / S905D 1 GB
boards reserve 256 MB and are limited to 1080p.  S805X 512 MB boards
reserve 128 MB and are limited to 1080p.

The kernel module parameter `meson_vdec_core.fbc_sg` (default `Y`)
enables a scatter-gather allocator for the per-frame AFBC body.  This
fits the 6K / 8K workspace into the standard 980 MB CMA on platforms
where an `AFBC_SCATTER` format is selected on the capture queue.
Setting it to `N` falls back to a single contiguous allocation.

## Performance

See [`benchmarks.md`](benchmarks.md) for measured frames-per-second on
each supported SoC at 720p, 1080p, and 4K UHD, plus per-resolution CMA
usage.

## V4L2 API

The decoder is a standard V4L2 stateful memory-to-memory device --
discovery, format negotiation, the `V4L2_EVENT_SOURCE_CHANGE` flow,
and `V4L2_DEC_CMD_STOP` drain are all upstream contract.  See
[`api.md`](api.md) for the device-discovery snippet, the GStreamer /
FFmpeg invocations, and the AFBC export path.

## Multi-instance

Single hardware decoder core per SoC -- one decode session at a time.
Concurrent V4L2 opens are accepted, but the second session blocks
until the first completes.

## Related upstream documentation

- Linux kernel `Documentation/userspace-api/media/v4l/dev-decoder.rst`
- Linux kernel `Documentation/userspace-api/media/v4l/dev-mem2mem.rst`
- VP9 Bitstream and Decoding Process Specification (Google, 2016-03-31)
