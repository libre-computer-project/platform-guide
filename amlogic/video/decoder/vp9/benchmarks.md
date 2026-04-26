# VP9 Decoder Benchmarks

Measured frames-per-second on Linux 6.12.80, performance governor at
maximum supported VDEC frequency (667 MHz).  Source: VP9 Profile 0
streams from the WebM reference test set, no `-re`, second pass after
warmup.

## Throughput

| SoC | 720p | 1080p | 4K UHD |
|-----|------|-------|--------|
| S805X | 33.0 fps | 21.8 fps | -- |
| A311D | 47.8 fps | 39.5 fps | 8.8 fps |
| S905D3 | 37.4 fps | 26.0 fps | 6.3 fps |

S805X / S905X / S905D are limited to 1080p by their CMA pool.  A311D
and S905D3 decode every advertised resolution; 4K is sub-real-time
on these platforms.

Measurements are 300-frame averages via:

```sh
gst-launch-1.0 filesrc location=<source.ivf> ! ivfparse ! vp9parse \
    ! v4l2vp9dec ! fakesink sync=false
```

with `dmesg -n 1` set before measurement (default kernel verbosity
caps throughput at ~50 fps regardless of resolution).

## Quality

VP9 decode is bit-exact against the reference software decoder
(libvpx) for every Profile 0 stream tested at 320x240 through 5K on
A311D and S905D3.  PSNR vs reference: infinite (bit-perfect) on every
non-EOS frame.

## Memory (CMA) usage

Per-session CMA allocation, peak during steady-state decode:

| Resolution | Peak CMA | Notes |
|------------|----------|-------|
| 720p | ~50 MB | NV12 capture, 12-buffer DPB |
| 1080p | ~110 MB | NV12 capture, 12-buffer DPB |
| 4K UHD | ~388 MB | NV12 capture, 12-buffer DPB |
| 5K (5120x2880) | ~660 MB | NV12 capture, 12-buffer DPB |

CMA is released within seconds of session end; no monotonic leak
observed across 10 sequential decode sessions (within-session
oscillation of 5-10 MB is kernel allocator fragmentation, not
session-scoped).

The `AFBC_SCATTER` capture path (`AMS8` / `AMSA`) reduces CMA
pressure at 5K and above by ~165 MB versus NV12 -- the same buffer
serves the decoder's reference pool and the VPU's display engine, so
no separate FBC reference allocation is needed.

128 MB CMA (S805X 512 MB DDR boards) is sufficient for 1080p VP9
decode.  4K decode requires the 980 MB CMA pool that ships on 4 GB
A311D and S905D3 boards.

## Test methodology

- Source: VP9 Profile 0 streams from the WebM reference test set.
- Decoder: `v4l2vp9dec` (GStreamer) or `vp9_v4l2m2m` (FFmpeg) or
  `libva-v4l2-m2m` (VA-API).
- Disable kernel debug logging (`dmesg -n 1`) before measurement.
- Run twice and use the second-pass number; first-pass is dominated
  by power-state and clock ramp-up.
- 300-frame clips for 720p / 1080p, 60-frame clips for 4K.
