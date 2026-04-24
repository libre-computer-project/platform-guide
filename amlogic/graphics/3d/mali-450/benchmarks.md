# Mali-450 Benchmarks

Measured performance figures for Mali-450 MP3 across Libre Computer
GXL boards. Numbers are headless (Wayland headless backend) using
upstream Mesa Lima.

## Configuration

| Parameter | Value |
|-----------|-------|
| GPU | Mali-450 MP3 (1 GP + 3 PP, Utgard) |
| Mesa | upstream main (>= 25.0) |
| Kernel | mainline 6.12.x with `lima` and `meson-drm` |
| Surface | offscreen pbuffer / FBO (no scanout) |
| Color buffer | RGBA8888 D24 S0 (no MSAA) |
| CPU governor | `performance` (locked to max) |
| GPU governor | `performance` (locked to indicated frequency) |

## glmark2 score (offscreen, 1024x768)

`glmark2-es2-wayland` against headless `weston`, single run aggregate
score.

| Board | SoC | RAM | GPU clock | glmark2 score |
|-------|-----|-----|-----------|---------------|
| Le Potato (AML-S905X-CC) | S905X | 2 GB | 666 MHz (default) | 81 |
| Sweet Potato (AML-S905X-CC-V2) | S905X | 2 GB | 666 MHz (default) | 81 |
| La Frite (AML-S805X-AC) | S805X | 1 GB | 792 MHz (turbo) | 530 |
| La Frite (AML-S805X-AC) | S805X | 1 GB | 1008 MHz (overclock) | 536 |
| La Frite (AML-S805X-AC) | S805X | 1 GB | 1200 MHz (overclock) | 553 |
| La Frite (AML-S805X-AC) | S805X | 1 GB | 1392 MHz (overclock) | 537 |
| La Frite (AML-S805X-AC) | S805X | 1 GB | 1584 MHz (overclock) | 534 |

Above ~1 GHz, glmark2 becomes CPU-bound: the test scripts spend more
time submitting draws than the GPU spends rendering them, so the
overall score plateaus. GPU-bound workloads continue to scale linearly
with clock -- see the shader-throughput table below.

The S805X is the only Libre Computer GXL platform validated above the
upstream stock 666 MHz cap. S905X boards run the upstream-defined OPP
table and top out at 666 MHz in shipping configurations.

## Shader throughput (lima-bench-shader, 512x512 surface, FPS)

GPU-bound shader fill-rate, measured at 744 MHz (stock turbo) and
1584 MHz (2.13x overclock):

| Shader workload | 744 MHz (Mpix/s) | 1584 MHz (Mpix/s) | Scaling vs clock ratio |
|-----------------|-----------------:|------------------:|----------------------:|
| fillrate / mrt | 90.3 | 178.1 | 1.97x (-7.5% vs ideal) |
| discard | 42.0 | 82.5 | 1.96x |
| gtf | 26.5 | 51.9 | 1.96x |
| blur | 24.9 | 49.1 | 1.97x |
| msaa-max | 19.9 | 39.3 | 1.97x |
| lod | 16.7 | 33.1 | 1.98x |
| simplex | 12.1 | 23.7 | 1.96x |

A 2.13x clock ratio yields 1.96-1.98x throughput. The ~8% gap is
fixed-cost per-frame overhead (tile setup, polygon list parse,
writeback to system memory) that does not scale with clock.

## Sustained-load thermal stability

60-second sustained `glmark2 build:use-vbo=true` load with passive
heatsink (no active cooling), ambient ~22 C:

| Frequency | Sustained 60s | Steady-state SoC temp |
|-----------|---------------|-----------------------|
| 666 MHz (S905X stock) | yes | ~70 C |
| 792 MHz (S805X turbo) | yes | ~80 C |
| 1008 MHz (S805X OC) | yes | ~66 C |
| 1200 MHz (S805X OC) | yes | ~69 C |
| 1392 MHz (S805X OC) | yes | ~70 C |
| 1584 MHz (S805X OC) | yes | ~72 C |

All overclock points are at 1.0 V VDDEE (boot default, no voltage
scaling). Use a heatsink or active cooling for sustained workloads
above stock clock.

## dEQP-GLES2 conformance

| Board | Subset | Pass | Fail | Notes |
|-------|--------|------|------|-------|
| Le Potato (S905X) | depth_stencil | 621 / 621 | 0 | All depth and base stencil pass |
| Le Potato (S905X) | fbo | 618 / 711 | 93 | Format-coverage + RGBA mismatches |
| Le Potato (S905X) | indexing | 568 / 568 | 0 | Full pass |
| Le Potato (S905X) | stencil_ops INCR/DECR/WRAP (d24s8) | 161 / 593 | 432 | Silicon limit on stencil arithmetic write masks |
| La Frite (S805X) overclock | full GLES2 (subset 1318) | 1318 / 1318 | 0 | Stable at 1200, 1392, 1584 MHz |

The stencil-ops arithmetic case is a Mali-450 silicon limitation, not
a driver gap -- the proprietary blob has the same failure pattern.

## Frequency overlay reference

For the S805X boards, the kernel ships a base 125-792 MHz OPP table.
DT overlays in the wiring tool extend this to overclock points:

| Overlay | Target frequency | PLL setting |
|---------|------------------|-------------|
| `gpu-turbo-792` | 792 MHz | M=66, OD=1 |
| `gpu-turbo-1008` | 1008 MHz | M=42, OD=0 |
| `gpu-turbo-1200` | 1200 MHz | M=50, OD=0 |
| `gpu-turbo-1392` | 1392 MHz | M=58, OD=0 |
| `gpu-turbo-1584` | 1584 MHz | M=66, OD=0 |

Apply via `ldto enable gpu-turbo-1008`, then reboot. Verify with
`cat /sys/class/devfreq/*.gpu/cur_freq`.

## Notes on benchmarking

- Always set CPU and GPU governors to `performance` before
  benchmarking. Leaving the default `schedutil` / `simple_ondemand`
  introduces ~35% variance at low draw counts.
- glmark2 `-s 1024x768` is a fixed budget; numbers are not directly
  comparable to other resolutions.
- `glmark2-es2-drm` does not work on headless boards because the
  meson-vpu KMS device and the lima render device are separate DRM
  cards. Use the Wayland-headless method shown in [api.md](api.md).
- Run each benchmark twice and take the second result. The first run
  primes Mesa shader and pipeline caches.
