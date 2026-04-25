# H.264 Decoder Benchmarks

Measured frames-per-second on Linux 6.12.80, schedutil governor at
maximum supported frequency.  Source: H.264 testsrc High profile,
yuv420p, no `-re`, second pass after warmup.

## Throughput

| SoC      | 320x240 | 720p   | 1080p  | 1080i  | 1440p | 4K UHD |
|----------|---------|--------|--------|--------|-------|--------|
| S805X    | 65 fps  | 130 fps | 54 fps | 81 fps | --    | --     |
| A311D    | 1209 fps | 322 fps | 152 fps | 175 fps | 74 fps | 36 fps |
| S905D3   | 2612 fps | 322 fps | 153 fps | 173 fps | 65 fps | 32 fps |

S805X cannot decode resolutions above 1080p -- the firmware caps
high-resolution streams at 1920x1088.  All other supported SoCs
decode 4K UHD H.264 in real time.

The 320x240 numbers on A311D and S905D3 are derived from
wall-clock measurement -- the FFmpeg fps counter saturates at
this throughput.  720p / 1080p / 1080i / 1440p / 4K numbers come
from the FFmpeg sustained-rate summary line.

The 1080i throughput often exceeds 1080p.  Interlaced content
uses simpler intra-prediction (one mode per field) for the same
total pixel count.

## Quality

H.264 IDCT is fully bit-exact on every supported SoC.  Decoded
output matches the reference software decoder byte-for-byte
across:

- All profiles (Constrained Baseline, Baseline, Main, High)
- All resolutions from 208x160 up to the per-SoC maximum
- All bitrates up to ~824 Mbps at 1080p
- Interlaced (TFF, BFF, MBAFF, PAFF), portrait, multi-slice
- Cropped output (non-MB-aligned widths)

PSNR vs reference: infinite (bit-perfect) on every passing test.

## Memory (CMA) usage

Per-session CMA allocation, peak during steady-state decode:

| Resolution | Peak CMA | Notes |
|------------|----------|-------|
| 320x240 | ~7 MB | |
| 720p | ~5 MB | |
| 1080p | ~12 MB | |
| 4K UHD | ~50 MB | A311D / S905D3 only |

CMA is released within seconds of session end; no leak observed
across long-running test loops (10+ sequential decodes).

128 MB CMA pool (S805X 512 MB DDR boards) is sufficient for
1080p decode.  4K decode requires the 980 MB CMA pool that ships
on 4 GB A311D and S905D3 boards.

## High-bitrate behaviour

| Bitrate at 1080p | Behaviour |
|------------------|-----------|
| Up to 200 Mbps | Full real-time, no errors |
| 200-400 Mbps | Real-time, no errors |
| 400-824 Mbps (S905X / A311D) | All frames decoded, sub-real-time (~4-5 fps) |
| 500+ Mbps high-density content (S905D3) | Possible esparser overflow on 1+ MB NAL units; partial-frame loss |

The S905D3 sub-real-time threshold is a buffer-pool sizing limit,
not a silicon limit.  Real-world H.264 content (5-100 Mbps)
decodes at full frame rate on every supported SoC.

## Test methodology

- Source: synthetic H.264 stream generated via FFmpeg `testsrc`.
- Decoder: `h264_v4l2m2m` (FFmpeg) or `v4l2h264dec` (GStreamer).
- Disable kernel debug logging (`dmesg -n 1`) before measurement;
  default verbosity caps throughput at ~50 fps regardless of
  resolution.
- Run twice and use the second-pass number; first-pass is
  dominated by power-state and clock ramp-up.
