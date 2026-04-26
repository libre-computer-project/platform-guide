# HEVC Encoder Benchmarks

Measured frames-per-second on Linux 6.12 with the encoder core
at its maximum stable frequency (667 MHz).  Source: synthetic
NV12 content at 30 fps, sustained-rate measurement after
warmup.

## Throughput -- bitrate-targeting mode

Bitrate-targeting mode (`V4L2_CID_MPEG_VIDEO_BITRATE > 0`):

| SoC | 320x240 | 640x480 | 720p | 1080p |
|-----|---------|---------|------|-------|
| A311D  | 31 fps | 31 fps | 30 fps | 26 fps |

Bitrate-targeting mode reaches real-time 30 fps at 720p and
encodes 1080p at 26 fps on A311D.

## Throughput -- CQP mode

CQP mode (`V4L2_CID_MPEG_VIDEO_BITRATE = 0`,
`V4L2_CID_MPEG_VIDEO_HEVC_I_FRAME_QP = 20`):

| SoC | 320x240 | 640x480 | 720p | 1080p | 1440p | 4K UHD |
|-----|---------|---------|------|-------|-------|--------|
| A311D  | 17 fps | 17 fps | 16 fps | 16 fps | -- | -- |
| S905D3 | 20 fps | 20 fps | 20 fps | 18 fps | 13 fps | 8 fps |

CQP runs slower than the bitrate-targeting mode because the
encoder spends per-frame cycles applying the user-supplied QP
rather than running its rate-control loop.  Use the bitrate-
targeting mode for real-time pipelines and CQP for offline
quality-targeted encoding.

The 4K UHD bitstream stays well within the encoder's CTU
budget; the throughput cap is the per-frame compute cost of
4K HEVC encode.

## Quality -- CQP at QP=20

Average PSNR over the encoded frames, measured against the
reference NV12 source after decode:

| Resolution | Avg PSNR |
|-----------|---------|
| 256x128   | 46.6 dB |
| 320x240   | 49.1 dB |
| 640x480   | 51.0 dB |
| 720p      | 51.6 dB |
| 1080p     | 52.2 dB |
| 1440p     | 52.3 dB |
| 4K UHD    | 52.5 dB |
| DCI 4K (4096x2160) | 52.6 dB |
| 4096x2304 | 53.5 dB |

Hardware output is bit-identical between A311D and S905D3 at
the same QP -- the encoder IP and firmware are the same and
produce the same bitstream.

## Quality -- bitrate scaling

720p, 90-frame test content, bitrate-targeting mode:

| Target bitrate | Average PSNR | Output size |
|---------------|--------------|-------------|
| 0.5 Mbps  | 33.9 dB | 199 KB |
| 1 Mbps    | 36.4 dB | 407 KB |
| 2 Mbps    | 39.0 dB | 717 KB |
| 8 Mbps    | 49.0 dB | 2.4 MB |
| 15 Mbps   | 54.0 dB | 3.5 MB |

PSNR scales monotonically with bitrate up to the encoder's
saturation point near 15 Mbps for 720p test content.

## Pixel-format equivalence

NV12, NV21, and YUV420 input produce visually equivalent
encoded output at the same QP -- the encoder converts planar
input to its native semi-planar layout before encode.

## Multi-instance throughput

Aggregate frames-per-second across concurrent sessions, each
encoding the same source:

| Sessions x resolution | Aggregate fps |
|----------------------|---------------|
| 2 x QVGA (320x240) | ~50 fps total |
| 3 x QVGA (320x240) | ~60 fps total |
| 2 x 480p (854x480) | ~50 fps total |
| 3 x 480p (854x480) | ~55 fps total |
| 2 x 720p           | ~40 fps total |

## Test methodology

- Encoder element: `hevc_v4l2m2m` (FFmpeg) or `v4l2h265enc`
  (GStreamer).
- Source: synthetic NV12 testsrc, 30 fps base rate.
- Disable kernel debug logging (`dmesg -n 1`) before measurement;
  default verbosity caps throughput.
- Run twice and use the second-pass number; first-pass is
  dominated by clock ramp-up and the encoder's per-module
  initialisation.
- For quality measurements, encode in CQP mode at the listed
  I-frame QP, decode the output with a software reference HEVC
  decoder, and compute Y-plane PSNR against the original NV12
  source.
