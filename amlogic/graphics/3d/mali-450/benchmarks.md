# Mali-450 Benchmarks

Measured performance figures for Mali-450 MP3 across Libre Computer
GXL boards. Numbers are headless using upstream Mesa Lima.

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

## Default GPU clocks

The kernel sets a per-SoC default `max_freq` cap that the GPU governor
respects:

| SoC | Default cap | Clock path |
|-----|-------------|------------|
| S805X | 666 MHz | `fclk_div3` (fixed-divider, no PLL involvement) |
| S905X | 744 MHz | `gp0_pll` (M=62, OD=1, DCO=1488 MHz) |

Userspace can lift the cap at runtime via:

```
echo <freq_hz> | sudo tee /sys/class/devfreq/d00c0000.gpu/max_freq
```

OC rates above the default cap are exposed via `gpu-turbo-*` device
tree overlays in `libretech-wiring-tool`. Validated headroom on a
shipping S905X die under sustained shader-bound workload is **828 MHz**
(via `gpu-turbo-828`, M=69 OD=1, DCO=1656); above that the GPU's
internal control bus times out under load. S805X dies are best
operated at the 666 MHz default -- the fclk_div3 path is the most
power-efficient and most thermally headroom on the S805X package.

Always verify an OC rate after applying:

1. `cat /sys/class/devfreq/*.gpu/cur_freq` (kernel target)
2. PLL `LOCK` bit: `sudo busybox devmem 0xc883c040 32` -- bit 31 must be 1
3. Hardware measurement: `sudo cat /sys/kernel/debug/meson-clk-msr/measure_summary | grep mali` -- must be within ±2 MHz of the target

If any of the three disagrees with the target, the rate is not
delivered to the GPU regardless of what `cur_freq` reports.

## glmark2-es2-wayland build:use-vbo

640x480 surfaceless. CPU performance governor.

| Board | GPU clock | glmark2 FPS |
|-------|----------:|------------:|
| Sweet Potato (AML-S905X-CC-V2) | 666 MHz | 646 |
| Sweet Potato (AML-S905X-CC-V2) | 744 MHz (default) | 664 |
| Sweet Potato (AML-S905X-CC-V2) | 804 MHz | 674 |
| Sweet Potato (AML-S905X-CC-V2) | 828 MHz | 675 |
| La Frite (AML-S805X-AC) | 666 MHz (default) | 493 |

`build:use-vbo=true` is CPU-bound; FPS plateaus once the GPU is no
longer the bottleneck. Use the shader-bound table below to evaluate
GPU clock scaling.

## Shader throughput (lima-bench-shader, 512x512 surface, Mpix/s)

GPU-bound shader-fill measurement. Eight shader-complexity levels
exercised at each clock; numbers are pixel-fill throughput in
megapixels per second after a 3-second warm run.

#### Sweet Potato (AML-S905X-CC-V2, S905X)

| Shader | 666 MHz | 744 MHz | 804 MHz | 828 MHz | Scaling 666→828 |
|--------|--------:|--------:|--------:|--------:|----------------:|
| passthrough (varying only) | 248.6 | 277.3 | 294.5 | 303.6 | 1.22× |
| ALU light (mul + add + clamp) | 191.9 | 209.9 | 222.3 | 229.7 | 1.20× |
| ALU medium (mix + dot + sat) | 108.3 | 116.6 | 127.3 | 131.3 | 1.21× |
| ALU heavy (8-iter loop) | 34.9 | 38.5 | 41.7 | 42.8 | 1.23× |
| trig (sin + cos, 3 calls) | 115.1 | 126.2 | 133.4 | 137.8 | 1.20× |
| trig heavy (4-octave, 12 calls) | 40.1 | 45.2 | 48.7 | 50.1 | 1.25× |
| branching (5-way if/else) | 121.8 | 132.5 | 143.2 | 146.4 | 1.20× |
| uniform heavy (8 vec4) | 107.4 | 118.5 | 127.7 | 130.7 | 1.22× |

Clock ratio 666→828 = 1.24×. Measured scaling 1.20-1.25× across all
eight shader workloads -- linear with clock, confirming the GPU is
delivering the programmed rate.

#### La Frite (AML-S805X-AC, S805X)

| Shader | 666 MHz |
|--------|--------:|
| passthrough | 296.2 |
| ALU light | 213.7 |
| ALU medium | 125.6 |
| ALU heavy | 35.7 |
| trig | 192.3 |
| trig heavy | 62.9 |
| branching | 148.6 |
| uniform heavy | 108.3 |

The S805X shader throughput at 666 MHz is consistently higher than
the S905X at the same clock (e.g., passthrough 296 vs 249 Mpix/s).
Same Mali-450 MP3 IP block; the difference reflects per-die silicon
process variation across the GXL family.

## dEQP-GLES2 conformance (Le Potato, AML-S905X-CC, 666 MHz)

| Subset | Pass | Fail | Notes |
|--------|------|------|-------|
| depth_stencil | 621 / 621 | 0 | All depth and base stencil pass |
| fbo | 618 / 711 | 93 | Format-coverage and RGBA mismatches |
| indexing | 568 / 568 | 0 | Full pass |

Stencil compare and replace operations work correctly. Stencil
arithmetic write masks (INCR / DECR / WRAP on d24s8) are a Mali-450
silicon characteristic -- the proprietary blob exhibits the same
pattern.

## Notes on benchmarking

- Always set CPU and GPU governors to `performance` before
  benchmarking. The default `schedutil` / `simple_ondemand` introduces
  ~35% variance at low draw counts.
- `glmark2-es2-drm` does not work on headless boards because the
  KMS device (`meson-vpu`) and the GPU render device (`lima`) are
  separate DRM cards. Use the Wayland-headless method shown in
  [api.md](api.md).
- Run each benchmark twice and take the second result. The first
  run primes Mesa shader and pipeline caches.
