# H.264 Multi-View Coding (MVC)

Stereo 3D video decode -- H.264 Stereo High profile (118 / 128).
Two synchronized views from a single bitstream, used by HDMI 1.4a
3D Blu-ray and the Kodi / CoreELEC 3D-playback path.

## Supported boards

Available on every supported SoC -- see the board list in
[`README.md`](README.md).

## Output mode

Selected by the `mvc_output_mode` module parameter on the
`meson_vdec_h264_mvc` module:

| `mvc_output_mode` | Mode | CAPTURE buffer shape |
|-------------------|------|----------------------|
| 0 (default) | Frame-packed | `W x (H*2 + 45)` NV12, view 0 + 45-line gap + view 1 |
| 1 | Frame-sequential | One CAPTURE buffer per view; view 0 = `V4L2_FIELD_NONE`, view 1 = `V4L2_FIELD_TOP` |

Frame-packed is the default because the HDMI 1.4a frame-packing
geometry (1920x2205 = 1920x1080 + 45-line blanking + 1920x1080)
is the only format that consumes both views simultaneously
without requiring an MVC-aware userspace.

## V4L2 input format

OUTPUT queue: `M264` (`V4L2_PIX_FMT_H264_MVC`).

CAPTURE queue: `NM12` (`V4L2_PIX_FMT_NV12M`) -- same as the
non-MVC H.264 path, with the per-mode buffer shape above.

## Reference resolution

640x480 stereo is the firmware-native MVC reference resolution
and the validated configuration today.

## Multi-instance

Single MVC session at a time -- shares the H.264 hardware
decoder core.
