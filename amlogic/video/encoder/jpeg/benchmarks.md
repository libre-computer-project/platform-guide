# JPEG Encoder Benchmarks

Measured frames-per-second and average output size on Linux
6.12 with the encoder core at its default frequency (S805X /
S905X / S905D at 667 MHz; A311D and S905D3 at 800 MHz).
Source: synthetic NV12 testsrc content, sustained-rate
measurement after warmup.

## Throughput -- 720p quality 85

| SoC    | NV12 input |
|--------|-----------|
| S805X  |  78 fps |
| A311D  | 109 fps |
| S905D3 | 108 fps |

## Throughput -- per pixel format

Frames-per-second at 720p quality 85 across input pixel formats:

| Format | S805X | A311D | S905D3 |
|--------|-------|-------|--------|
| NV12   |  78 fps |  90 fps | 109 fps |
| NV21   |  77 fps | 109 fps | 106 fps |
| YUYV   |  71 fps | 122 fps |  98 fps |
| RGB24  |  78 fps | 147 fps | 121 fps |
| ABGR32 |  79 fps | 128 fps | 121 fps |

RGB24 is the fastest input on A311D (147 fps).  The hardware
RGB-to-YCbCr converter and the JPEG quantiser pipeline overlap
better with raster RGB input than with semi-planar NV12.

A311D and S905D3 expose a turbo overclock (1 GHz) via the
`hcodec_turbo` module parameter:

| Format | A311D 800 MHz | A311D 1 GHz | S905D3 800 MHz | S905D3 1 GHz |
|--------|---------------|-------------|----------------|--------------|
| NV12   |  90 fps |  93 fps | 109 fps | 113 fps |
| ABGR32 | 128 fps | 140 fps | 121 fps | 132 fps |

The +25 % clock yields about +4 % to +9 % throughput.

## Output size and quality -- resolution sweep, NV12 input, quality 85

| Resolution | Output size | PSNR (Y plane) |
|-----------|------------|----------------|
| 160x128   |   4.7 KB | 41.1 dB |
| 320x240   |   8.9 KB | 44.5 dB |
| 640x480   |  16.5 KB | 48.4 dB |
| 1280x720  |  33.9 KB | 51.8 dB |
| 1920x1088 |  64.3 KB | 54.0 dB |

Compression efficiency improves with resolution -- larger images
amortise the JFIF header and Huffman DC overhead better.  Bytes-
per-pixel converges around 0.06 at 1080p.

## Output size and quality -- quality sweep at 320x240

| Quality | Output size | PSNR (Y plane) |
|--------|------------|----------------|
|  10 |  3.3 KB | 31.0 dB |
|  25 |  4.4 KB | 34.5 dB |
|  50 |  5.8 KB | 37.7 dB |
|  75 |  7.5 KB | 41.3 dB |
|  85 |  8.9 KB | 44.5 dB |
|  95 | 12.5 KB | 53.9 dB |

Quality scales monotonically across the full range.  Quality 95
is about 36 % larger than quality 85.

## Bitrate envelope -- MJPEG at 30 fps

Implied MJPEG stream bitrate when emitting JPEG frames at 30 fps:

| Resolution | Q10 | Q25 | Q50 | Q75 | Q85  | Q95  |
|-----------|-----|-----|-----|-----|------|------|
| 160x128   | 0.4 Mbps | 0.5 Mbps | 0.7 Mbps | 0.8 Mbps |  1.0 Mbps |  1.3 Mbps |
| 320x240   | 0.8 Mbps | 1.2 Mbps | 1.5 Mbps | 1.9 Mbps |  2.3 Mbps |  3.0 Mbps |
| 640x480   | 2.4 Mbps | 3.3 Mbps | 4.2 Mbps | 5.2 Mbps |  6.1 Mbps |  8.2 Mbps |
| 1280x720  | 6.4 Mbps | 8.5 Mbps | 10.9 Mbps| 13.5 Mbps| 15.6 Mbps | 21.2 Mbps |
| 1920x1088 |13.7 Mbps |18.0 Mbps | 22.9 Mbps| 28.1 Mbps| 32.4 Mbps | 44.0 Mbps |

JPEG has no traditional CBR bitrate control -- quality is the
only rate knob.  To target a specific MJPEG bitrate, pick the
(resolution, quality) cell from this envelope.

## Compression efficiency at quality 85

| Resolution | Pixels    | Bytes per pixel |
|-----------|-----------|-----------------|
| 160x128   |    20,480 | 0.20 |
| 320x240   |    76,800 | 0.12 |
| 640x480   |   307,200 | 0.083 |
| 1280x720  |   921,600 | 0.071 |
| 1920x1088 | 2,088,960 | 0.065 |

## Cross-platform output equivalence

NV12 input produces byte-identical JPEG output across A311D and
S905D3 at the same resolution and quality -- the encoder IP and
firmware are the same.  Per-platform output may differ on the
S805X / S905X / S905D family due to a different firmware
binary, but quality remains in the same 41 .. 54 dB range across
the resolution sweep.

## Test methodology

- Encoder element: `v4l2jpegenc` (GStreamer) for throughput,
  `v4l2-ctl` for single-frame quality measurement.
- Source: synthetic NV12 testsrc.
- Quality measurement: decode the encoded JPEG with a software
  reference JPEG decoder and compute Y-plane PSNR against the
  original NV12 source.
- Disable kernel debug logging (`dmesg -n 1`) before measuring
  throughput.
- Warm the encoder with a discard pass before recording the
  measured pass.
