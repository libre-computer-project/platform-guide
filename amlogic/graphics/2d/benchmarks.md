# Benchmarks

Measured throughput of the 2D graphics engine across three SoCs.
All numbers are at the highest stable clock rate (667 MHz turbo)
with maximum AXI burst tuning, using a V4L2 M2M pipeline with
double-buffered `QBUF` / `DQBUF` and 100 iterations per cell.

## Platform summary

| SoC | Representative board | Memory | 4K copy | Max stable clock |
|-----|----------------------|--------|--------:|-----------------:|
| S805X  | AML-S805X-AC (La Frite) | DDR3 / LPDDR3 | 25.9 fps | 667 MHz |
| S905X  | AML-S905X-CC (Le Potato) | DDR3 / LPDDR3 | 25.9 fps | 667 MHz |
| A311D  | AML-A311D-CC (Alta)      | DDR4 | 38.4 fps | 667 MHz |
| S905D3 | AML-S905D3-CC (Solitude) | DDR4 | 35.4 fps | 667 MHz |

At resolutions up to 1080p, per-blit overhead dominates and the
engine saturates at ~53 fps on all SoCs. At 4K, DDR bandwidth
becomes the hard ceiling: DDR4 platforms (A311D, S905D3) reach
33-38 fps, DDR3 platforms (S805X, S905X) cap at ~25 fps.

## Resolution vs throughput (12 representative format pairs)

### AML-A311D-CC (A311D)

| Source -> Destination | 640x480 | 1280x720 | 1920x1080 | 3840x2160 |
|---|---:|---:|---:|---:|
| XRGB32 -> XRGB32 (identity copy) | 53.0 | 52.9 | 52.9 | 38.4 |
| XRGB32 -> BGRX32 (byte swap)     | 52.9 | 52.9 | 53.4 | 38.6 |
| XRGB32 -> YUYV (RGB -> YUV 4:2:2) | 52.9 | 52.9 | 53.4 | 38.6 |
| YUYV -> XRGB32 (YUV -> RGB)       | 53.5 | 52.9 | 53.3 | 38.6 |
| XRGB32 -> NV12M (RGB -> YUV 4:2:0) | 53.5 | 53.0 | 52.9 | 38.6 |
| NV12M -> XRGB32 (YUV -> RGB)       | 53.5 | 53.4 | 53.0 | 38.4 |
| XRGB32 -> RGB24 (32-bit -> 24-bit) | 52.9 | 52.9 | 52.9 | 38.6 |
| XRGB32 -> RGB565 (32-bit -> 16-bit) | 53.4 | 52.9 | 53.4 | 38.4 |
| YUYV -> NV12M                       | 52.9 | 52.9 | 53.4 | 34.5 |
| NV12M -> YU12 (semi-planar -> planar) | 52.9 | 53.4 | 52.9 | 38.6 |
| XRGB32 -> AM4A (8-bit -> 10-bit YUV 4:4:4) | 52.9 | 52.9 | 52.9 | 38.6 |
| XRGB32 -> AMRA (8-bit -> 10-bit RGB)       | 52.9 | 53.0 | 52.9 | 38.6 |

### AML-S905D3-CC (S905D3)

| Source -> Destination | 640x480 | 1280x720 | 1920x1080 | 3840x2160 |
|---|---:|---:|---:|---:|
| XRGB32 -> XRGB32 | 53.5 | 53.4 | 53.4 | 38.5 |
| XRGB32 -> BGRX32 | 52.9 | 53.4 | 53.3 | 38.5 |
| XRGB32 -> YUYV   | 53.5 | 53.4 | 53.4 | 38.5 |
| YUYV -> XRGB32   | 53.5 | 52.9 | 53.4 | 38.3 |
| XRGB32 -> NV12M  | 53.5 | 52.9 | 53.4 | 38.5 |
| NV12M -> XRGB32  | 52.9 | 53.4 | 53.4 | 38.5 |
| XRGB32 -> RGB24  | 53.5 | 53.4 | 53.4 | 38.3 |
| XRGB32 -> RGB565 | 53.4 | 52.9 | 53.4 | 38.3 |
| YUYV -> NV12M    | 53.5 | 53.4 | 53.3 | 34.4 |
| NV12M -> YU12    | 53.5 | 53.4 | 53.4 | 38.4 |
| XRGB32 -> AM4A   | 52.9 | 52.9 | 53.4 | 38.5 |
| XRGB32 -> AMRA   | 53.5 | 53.4 | 53.4 | 38.5 |

### AML-S805X-AC (S805X)

| Source -> Destination | 640x480 | 1280x720 | 1920x1080 | 3840x2160 |
|---|---:|---:|---:|---:|
| XRGB32 -> XRGB32 | 53.7 | 53.2 | 50.5 | 25.0 |
| XRGB32 -> BGRX32 | 53.7 | 53.6 | 45.1 | 25.1 |
| XRGB32 -> YUYV   | 53.2 | 53.7 | 52.4 | 30.1 |
| YUYV -> XRGB32   | 53.2 | 53.6 | 46.6 | 27.6 |
| XRGB32 -> NV12M  | 48.1 | 53.7 | 53.4 | 30.0 |
| NV12M -> XRGB32  | 48.5 | 53.2 | 53.6 | 35.0 |
| XRGB32 -> RGB24  | 53.2 | 48.1 | 52.6 | 27.2 |
| XRGB32 -> RGB565 | 48.5 | 53.7 | 53.0 | 28.4 |
| YUYV -> NV12M    | 48.5 | 53.7 | 53.6 | 31.5 |
| NV12M -> YU12    | 53.7 | 48.5 | 53.6 | 35.9 |
| XRGB32 -> AM4A   | rejected (no deep-color support) |
| XRGB32 -> AMRA   | rejected (no deep-color support) |

## Scaler performance

3 format types x 7 ratios x 3 filters at 1080p, 100 iterations.

### Source 1920x1080 -> varied destination

| Format | src -> dst | bilinear | bicubic | triangle |
|--------|------------|---------:|--------:|---------:|
| XRGB32 | 1920x1080 -> 1920x1080 (identity) | 53.1 | 53.1 | 53.1 |
| XRGB32 | 1920x1080 ->  960x540  (2x down)  | 53.0 | 53.0 | 53.0 |
| XRGB32 | 1920x1080 ->  480x270  (4x down)  | 53.0 | 53.0 | 53.0 |
| XRGB32 | 1920x1080 ->  240x135  (8x down)  | 53.0 | 53.0 | 53.0 |
| YUYV   | 1920x1080 -> 1920x1080 | 53.0 | 53.0 | 53.0 |
| YUYV   | 1920x1080 ->  480x270  | 53.0 | 53.0 | 53.0 |
| NV12M  | 1920x1080 -> 1920x1080 | 53.0 | 53.0 | 53.0 |
| NV12M  | 1920x1080 ->  480x270  | 53.0 | 53.0 | 53.0 |

Upscale 2x / 4x / 8x hits the same ~53 fps ceiling. Filter choice
has no measurable throughput impact -- all three polyphase tables
run at the same rate.

## Format-pair matrix (1080p)

Across the full 40 x 40 format-pair matrix at 1920x1080 and 667
MHz turbo, the scaler is disabled (identity blit with format
conversion only):

| Platform | Valid pairs | Min fps | Median fps | Max fps | Mean fps |
|----------|------------:|--------:|-----------:|--------:|---------:|
| A311D    | 1600        | 52.9    | 53.4       | 53.4    | 53.1     |
| S905D3   | 1600        | 52.9    | 53.4       | 53.4    | 53.1     |
| S805X    | 1296        | 44.4    | 51.8       | 54.5    | 51.2     |

On S805X the 304 remaining pairs involve the four MESON deep-
color formats, which the hardware does not support on that SoC;
the driver returns `-EINVAL` from `VIDIOC_S_FMT`.

Pair-level distribution is narrow: 90 % of A311D / S905D3 cells
land between 52.9 and 53.4 fps; the 44-50 fps tail on S805X is
32-bit byte-swap and some CSC combinations, where the DDR3
subsystem and the absence of AXI burst tuning create a bandwidth
ceiling below the per-blit overhead ceiling.

## Clock frequency scaling

Four clock states via the thermal cooling framework. The tables
above are at state 0 (turbo). Lower states reduce fps roughly
linearly through 1080p and are DMA-bound at 4K.

| State | Rate | 4K fps (A311D) | 4K fps (S905D3) | 4K fps (S805X) |
|-------|-----:|---------------:|----------------:|---------------:|
| 0 (turbo)     | 667 MHz | 38.4 | 35.4 | 25.9 |
| 1 (rated)     | 500 MHz | 33.1 | 31.0 | 25.9 |
| 2 (throttle)  | 400 MHz | 27.2 | 27.2 | 18.9 |
| 3 (emergency) | 250 MHz | 20.4 | 20.4 | 18.9 |

Turbo is enabled via `/sys/devices/platform/*.ge2d/turbo`:

```sh
echo 1 | sudo tee /sys/devices/platform/soc/*ge2d/turbo
```

The thermal framework auto-throttles when the CPU thermal zone
reaches its trip point.

## Color space conversion accuracy

PSNR against a software BT.601 limited-range reference (ffmpeg),
measured on A311D:

| Test | PSNR (dB) |
|------|----------:|
| Identity XRGB 640x480                     | inf (bit-exact) |
| NV12 -> XRGB 640x480 (CSC only)           | 21.9 |
| Scale XRGB 640x480 -> 320x240 (no CSC)    | 35.6 |
| CSC + downscale NV12 640 -> XRGB 320      | 23.7 |
| CSC + upscale NV12 320 -> XRGB 640        | 18.6 |

The deep-color pipeline (AM4A / AM2A / AM2C destinations from
XRGB32 source) produces standard BT.601 limited-range output
within +/-1 rounding across 14 controlled RGB inputs on A311D.

## Blend performance

Blend adds zero throughput overhead versus a plain copy -- the
second source read pipelines with the destination write, and
fill-color blend eliminates the SRC2 memory read entirely.

| Operation (A311D, 1080p) | fps |
|---------------------------|----:|
| Copy (no SRC2)            | 52.9 |
| Fill-color blend          | 52.9 |
| DMABUF blend              | 52.9 |

| Operation (S805X, 1080p) | fps |
|---------------------------|----:|
| Copy (no SRC2)            | 47.4 |
| Fill-color blend          | 47.4 |
| DMABUF blend              | 47.4 |

## How these numbers were measured

Each cell is 100 iterations through the V4L2 M2M multiplanar
pipeline with 2 buffer pairs (double-buffered so `QBUF` /
`DQBUF` pipelines with the hardware). The driver's in-IRQ
`job_finish` wakes the next `device_run` before userspace
re-queues, so per-blit overhead is not dominated by userspace
round-trips. A `poll()` timeout of 3 seconds per `DQBUF` guards
against pipeline stalls; no cell in the runs reported above
tripped the timeout.
