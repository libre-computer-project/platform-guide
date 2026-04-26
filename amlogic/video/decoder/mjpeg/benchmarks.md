# MJPEG Decoder Benchmarks

Measured frames-per-second on Linux 6.12.80, performance governor
at maximum supported VDEC frequency (667 MHz).  Source: MJPEG
quality 2 streams generated with `ffmpeg ... -pix_fmt yuvj420p
-c:v mjpeg -q:v 2`, second pass after warmup.

## Resolution coverage

Frame delivery validated on each supported SoC across the standard
resolution tiers, with substitutions where the firmware
max_height = 2048 ceiling cuts in:

| Resolution | S805X / S905X / S905D | A311D / S905D3 |
|------------|----------------------|----------------|
| 320x240 | 30/30 | 30/30 |
| 720x480 | 30/30 | 30/30 |
| 1280x720 | 90/90 | 90/90 |
| 1920x1080 | 30/30 | 30/30 |
| 2560x1440 | 30/30 | 30/30 |
| 3840x2048 (4K UHD letterbox) | 5/5 | 5/5 |
| 4096x2048 (DCI 4K letterbox) | 5/5 | 5/5 |
| 8192x2048 (16.7 MP ceiling) | 5/5 | 5/5 |

The firmware caps each axis at 8192 x 2048 px; 3840x2160 / 4096x2160
height-2160 4K rows are not advertised (driver clamps via
`VIDIOC_ENUM_FRAMESIZES` to height 2048).  The 8192x2048
ceiling row covers both wide-letterbox 4K equivalents and the
maximum panorama format.

## Throughput

| SoC | 320x240 | 1280x720 | 1920x1080 | 2560x1440 | 8192x2048 |
|-----|---------|----------|-----------|-----------|-----------|
| S805X | 102 fps | 49 fps | 21 fps | 15 fps | 2.5 fps |
| A311D | 216 fps | 101 fps | 39 fps | 29 fps | 1 fps |
| S905D3 | 77 fps | 44 fps | 17 fps | 12 fps | 2.5 fps |

A311D (12 nm Cortex-A73) is roughly 2x faster than S805X / S905X
/ S905D / S905D3 at every resolution -- the JPEG decoder hardware
is identical across SoCs, but the V4L2 m2m buffer-management path
is CPU-bound (QBUF / DQBUF syscall rate, DMA cache ops, recycle
thread scheduling).

720p decode exceeds 30 fps on every supported SoC, so MJPEG is
real-time at HD on the entire fleet.  1080p exceeds 30 fps only
on A311D.

8192x2048 (16.7 megapixels per frame) is the firmware ceiling and
runs sub-real-time on every SoC.

Measurements via:

```sh
gst-launch-1.0 filesrc location=<source.mjpeg> ! jpegparse \
    ! v4l2jpegdec ! fakesink sync=false
```

with `dmesg -n 1` set before measurement (default kernel
verbosity caps throughput at ~50 fps regardless of resolution).

## Quality

MJPEG decode is quantization-limited; the post-scaler block
produces 3-plane I420 from the firmware-decoded macroblocks with
high PSNR versus a software reference:

| Resolution | Y PSNR | U PSNR | V PSNR |
|------------|--------|--------|--------|
| 320x240 q2 | ~73 dB | ~70 dB | ~70 dB |
| 1280x720 q2 | ~73 dB | ~70 dB | ~70 dB |
| 1920x1080 q2 | ~73 dB | ~70 dB | ~70 dB |

The ~3 dB chroma vs luma gap is a property of the post-scaler
chroma filter (8-tap I420 downsample) and is consistent across
resolutions and SoCs.

## Memory (CMA) usage

Per-session CMA allocation, peak during steady-state decode:

| Resolution | Peak CMA | Notes |
|------------|----------|-------|
| 320x240 | ~3 MB | I420 capture, 8-buffer DPB |
| 720x480 | ~7 MB | |
| 1280x720 | ~14 MB | |
| 1920x1080 | ~30 MB | |
| 2560x1440 | ~52 MB | |
| 3840x2048 | ~96 MB | 4K UHD letterbox |
| 4096x2048 | ~102 MB | DCI 4K letterbox |
| 8192x2048 | ~125 MB | A 16.7 MP I420 frame is ~25 MB |

128 MB CMA (S805X 512 MB DDR boards) is sufficient for every
supported resolution including 8192x2048.  CMA is released within
seconds of session end.

## Bitrate / quality range

MJPEG bitrate is determined entirely by JPEG quality factor.
Decoded output frame count is unaffected by bitrate -- every
quality from extreme-coarse to near-lossless decodes
successfully:

| Quality (-q:v) | Approx frame size at 720x480 | Result |
|----------------|------------------------------|--------|
| 50 (extreme coarse) | ~15 KB | OK |
| 31 (smallest practical) | ~22 KB | OK |
| 15 (good) | ~35 KB | OK |
| 5 (very high) | ~67 KB | OK |
| 2 (near-lossless) | ~119 KB at 1080p | OK |
| 1 (worst / largest) | ~100 KB at 720x480 | OK |

The on-chip input FIFO is fed by the elementary-stream parser
asynchronously from decode, so even very large frames at 200+
Mbps are buffered without parser overflow.

## Test methodology

- Source: synthetic MJPEG streams generated via FFmpeg
  `ffmpeg -f lavfi -i testsrc2=size=WxH:rate=30 -pix_fmt yuvj420p
  -c:v mjpeg -q:v 2 out.mjpeg`.
- Decoder: `v4l2jpegdec` (GStreamer).
- Disable kernel debug logging (`dmesg -n 1`) before measurement.
- Run twice and use the second-pass number; first-pass is
  dominated by power-state and clock ramp-up.
- 30-frame clips for 320x240 / 1080p / 1440p / 8192x2048;
  90-frame clips for 720p.
