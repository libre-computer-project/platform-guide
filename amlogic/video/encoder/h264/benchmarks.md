# H.264 Encoder Benchmarks

Measured frames-per-second on Linux 6.12 with the encoder core
at its default frequency (S805X / S905X / S905D at 667 MHz;
A311D and S905D3 at 800 MHz).  Source: synthetic NV12 testsrc
content at 30 fps, sustained-rate measurement after warmup.

## Throughput -- NV12 input

| SoC    | QCIF | 320x240 | 640x480 | 720p | 1080p |
|--------|------|---------|---------|------|-------|
| S805X  | 1304 fps | 746 fps | 281 fps | 116 fps | -- |
| A311D  | 1099 fps | 845 fps | 331 fps | 125 fps | 60 fps |
| S905D3 |  478 fps | 416 fps | 267 fps | 146 fps | 78 fps |

S805X / S905X / S905D 512 MB boards can encode up to 720p; 1080p
encode requires the 1 GB or larger CMA configuration.

## Throughput -- per pixel format

Frames-per-second at 720p across input pixel formats:

| Format | S805X | A311D | S905D3 |
|--------|-------|-------|--------|
| NV12   | 53 fps | 68 fps | 84 fps |
| NV21   | 53 fps | 67 fps | 84 fps |
| YUYV   | 52 fps | 73 fps | 84 fps |
| RGB24  | 54 fps | 79 fps | 84 fps |
| ABGR32 | 70 fps | 106 fps | 111 fps |

ABGR32 is the fastest input format -- the hardware DMAs the
buffer as a flat raster and converts to YCbCr inline.  The other
RGB / YUYV inputs go through the same hardware converter at a
lower rate.

A311D and S905D3 also expose a turbo overclock (1 GHz) via the
`hcodec_turbo` module parameter.  At turbo:

| Format | A311D 800 MHz | A311D 1 GHz | S905D3 800 MHz | S905D3 1 GHz |
|--------|---------------|-------------|----------------|--------------|
| NV12   | 68 fps | 72 fps | 84 fps | 90 fps |
| ABGR32 | 106 fps | 115 fps | 111 fps | 120 fps |

The +25 % clock yields about +6 % to +9 % throughput -- the
encoder is mostly bound by DRAM bandwidth and pipeline overhead,
not core clock.

## Quality -- IDR-only at default QP

Average PSNR over the encoded frames, single-IDR clip,
`V4L2_CID_MPEG_VIDEO_H264_I_PERIOD = 1`:

| SoC    | 320x240 | 640x480 | 1280x720 | 1920x1080 |
|--------|---------|---------|----------|-----------|
| S805X  | 40.5 dB | 44.0 dB | 45.9 dB | 46.8 dB |
| A311D  | 39.6 dB | 43.3 dB | 45.2 dB | 46.7 dB |
| S905D3 | 39.8 dB | 43.4 dB | 45.2 dB | 46.7 dB |

## Quality -- bitrate scaling

720p, VBR target bitrate, average PSNR over a 30-frame clip:

| Target bitrate | Average PSNR |
|---------------|--------------|
| 1 Mbps  | 28 dB |
| 2 Mbps  | 30 dB |
| 5 Mbps  | 33 dB |
| 10 Mbps | 38 dB |

Quality scales monotonically with bitrate.  The minimum effective
bitrate at 720p is around 3.2 Mbps -- below that the encoder's QP
floor (default 26) caps the rate.

## QP scaling

At 320x240, encoding 30 frames with `V4L2_CID_MPEG_VIDEO_H264_MIN_QP`
set to a fixed value:

| QP | Output size | Notes |
|----|------------|-------|
| 10 | 5.9 MB | High quality, large output |
| 26 | 3.1 MB | Default |
| 40 | 1.8 MB | |
| 51 | 1.3 MB | Maximum quantisation, smallest output |

A 4.6x size ratio across the QP range -- useful for offline
size-targeted encodes.

## Test methodology

- Encoder element: `h264_v4l2m2m` (FFmpeg) or `v4l2h264enc`
  (GStreamer).
- Source: synthetic NV12 testsrc, 30 fps base rate.
- Disable kernel debug logging (`dmesg -n 1`) before measurement;
  default verbosity caps throughput.
- Warm the encoder with a discard pass before recording the
  measured pass (clock ramp-up dominates first-pass numbers).
- For quality measurements, decode the encoded output with a
  software reference H.264 decoder and compute Y-plane PSNR
  against the original NV12 source.
