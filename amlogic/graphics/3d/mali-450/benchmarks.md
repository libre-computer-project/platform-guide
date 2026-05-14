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

## Default GPU caps and OC headroom

Per-SoC default `max_freq` caps (2026-05-14, kernel
`6.12.80-00991+`):

| SoC | Default cap | Headroom over fclk_div3 | PLL setting |
|-----|-------------|-------------------------|-------------|
| S805X (`amlogic,s805x`) | 666 MHz | 0 (fclk_div3, no PLL) | (none -- fclk path) |
| S905X (`amlogic,s905x`) | 744 MHz | +12% via GP0_PLL | M=62, OD=1, DCO=1488 |

Per-die overclock ceilings under canonical methodology (PLL `LOCK`
bit + `meson-clk-msr` HW clock cross-check + concurrent rate
sampling during the benchmark):

| SoC + die | Stable shader-bound ceiling | Failure mode above ceiling |
|-----------|----------------------------:|----------------------------|
| S905X-V2 die `82d7f3ee6f5d` | **828 MHz** (M=69 OD=1 DCO=1656) | `gp task error status=a` + `pp bus stop timeout` (kernel survives, jobs hang) |
| S805X die `0a153ae40ad2` | **708 MHz** (M=59 OD=1 DCO=1416, light-load only; shader-bound re-verify pending) | **SError 0x96000210** at `lima_l2_cache_flush` -- kernel panics |

Per-die variance is real and significant -- the 0a153 and 82d7f dies
have different silicon ceilings and different failure modes. The
744 MHz S905X default cap leaves ~10% headroom below the measured
82d7f die ceiling. S805X's 666 MHz cap leaves ~6% headroom below
0a153 die's 708 MHz ceiling while staying on the fclk_div3 path
(no PLL relock cost, no exposure to gp0_pll lock-failure class).

## glmark2-es2-wayland build:use-vbo

640x480 surfaceless. CPU performance governor, PLL `LOCK` + HW
clk_measure verified before each run, dmesg deltas checked.

| Board | Die | Rate | glmark2 FPS | Temp Δ |
|-------|-----|-----:|------------:|-------:|
| Sweet Potato (AML-S905X-CC-V2) | `82d7f3ee6f5d` | 666 MHz | 646 | +5°C |
| Sweet Potato (AML-S905X-CC-V2) | `82d7f3ee6f5d` | 744 MHz (stock cap) | 664 | +4°C |
| Sweet Potato (AML-S905X-CC-V2) | `82d7f3ee6f5d` | 804 MHz | 674 | +4°C |
| Sweet Potato (AML-S905X-CC-V2) | `82d7f3ee6f5d` | 828 MHz (die ceiling) | 675 | +4°C |
| La Frite (AML-S805X-AC) | `0a153ae40ad2` | 666 MHz (stock cap) | 493 | flat |

`build:use-vbo=true` is CPU-bound; FPS plateaus once the GPU isn't
the bottleneck. Use the shader-bound table below to evaluate clock
scaling.

## Shader throughput (lima-bench-shader, 512x512, Mpix/s)

GPU-bound test from `~/git/mesa/tools/lima-bench-shader.c`,
surfaceless EGL, 3 s per shader. Eight shader complexity levels.

#### S905X-V2 die `82d7f3ee6f5d`:

| Shader | 666 MHz | 744 MHz | 804 MHz | 828 MHz | Scaling 666→828 |
|--------|--------:|--------:|--------:|--------:|----------------:|
| passthrough (varying only) | 248.6 | 277.3 | 294.5 | 303.6 | 1.22× |
| ALU light (mul+add+clamp) | 191.9 | 209.9 | 222.3 | 229.7 | 1.20× |
| ALU medium (mix+dot+sat) | 108.3 | 116.6 | 127.3 | 131.3 | 1.21× |
| ALU heavy (8-iter loop) | 34.9 | 38.5 | 41.7 | 42.8 | 1.23× |
| trig (sin+cos, 3 calls) | 115.1 | 126.2 | 133.4 | 137.8 | 1.20× |
| trig heavy (4-octave, 12 calls) | 40.1 | 45.2 | 48.7 | 50.1 | 1.25× |
| branching (5-way if/else) | 121.8 | 132.5 | 143.2 | 146.4 | 1.20× |
| uniform heavy (8 vec4) | 107.4 | 118.5 | 127.7 | 130.7 | 1.22× |

Clock ratio 666→828 = 1.24×. Measured scaling 1.20-1.25× — linear
with clock, confirming the GPU genuinely runs at the programmed
rate.

#### S805X die `0a153ae40ad2`:

| Shader | 666 MHz | "672"* | "684"* | "696"* | "708"* |
|--------|--------:|-------:|-------:|-------:|-------:|
| passthrough | 296.2 | 299.0 | 294.6 | 298.6 | 289.3 |
| ALU light | 213.7 | 214.0 | 213.4 | 214.6 | 214.5 |
| ALU medium | 125.6 | 129.2 | 130.0 | 131.4 | 131.0 |
| ALU heavy | 35.7 | 35.7 | 35.7 | 35.8 | 35.7 |
| trig | 192.3 | 192.7 | 192.2 | 193.0 | 192.2 |
| trig heavy | 62.9 | 62.8 | 62.7 | 62.9 | 62.8 |
| branching | 148.6 | 149.8 | 149.1 | 148.3 | 149.7 |
| uniform heavy | 108.3 | 108.9 | 108.6 | 108.6 | 108.8 |

\* The "OC" columns are misleading. PLL locks at the target, pre-test
mali HW measure shows the target rate (671, 683, 695, 707 MHz), but
during the benchmark the lima driver silently scales mali back to
666 MHz, with no dmesg trace and well below the 80°C thermal trip.
**All "OC" columns are effectively at 666 MHz** — shader throughput
is identical to the 666 baseline within ±2% noise, and the post-test
HW measure confirms `mali=666 MHz` at every rate.

Above 708 MHz (720+), the lima soft scale-down apparently doesn't
fire fast enough and the GPU panics with **SError 0x96000210** at
`lima_l2_cache_flush`. So on this die there is no useful OC
operating point: lower rates are silently scaled back, and higher
rates panic.

This is why the S805X default cap is 666 MHz on the fclk_div3 path
— it avoids both classes of failure.

> **Note on prior published tables.** A previous version of this
> page carried a shader-throughput table with shaders named
> `fillrate / mrt / lod / gtf / msaa-max / discard / simplex / blur`
> at 744 vs 1584 MHz with "1.97x linear scaling". Those shader names
> do not exist in the `lima-bench-shader` binary or its source
> (`~/git/mesa/tools/lima-bench-shader.c`, single commit history);
> the table originated from a sub-agent fabrication in the Apr 10
> 2026 session and was never cross-checked before publication. See
> the project audit doc
> `amlogic/software/gpu/lima/audit/gp0-pll-dco-ceiling-per-die.md`
> "2026-05-14 PM3" for the full retraction. The tables above are
> the canonical-methodology replacement.

## dEQP-GLES2 conformance

| Board | Die | Subset | Pass | Fail | Notes |
|-------|-----|--------|------|------|-------|
| Le Potato (S905X) | (legacy) | depth_stencil | 621 / 621 | 0 | All depth and base stencil pass |
| Le Potato (S905X) | (legacy) | fbo | 618 / 711 | 93 | Format-coverage + RGBA mismatches |
| Le Potato (S905X) | (legacy) | indexing | 568 / 568 | 0 | Full pass |
| Le Potato (S905X) | (legacy) | stencil_ops INCR/DECR/WRAP (d24s8) | 161 / 593 | 432 | Silicon limit on stencil arithmetic write masks |
| La Frite (S805X) | `6a44addf0371` (one die only) | full GLES2 subset 1318 @ 1584 MHz | 1318 / 1318 | 0 | Parent-direct SSH verified 2026-04-10. Methodology pre-canonical (no PLL `LOCK` bit / HW clk_measure verify). Per-die variance: not reproducible on 0a153 die. |

The stencil-ops arithmetic case is a Mali-450 silicon limitation,
not a driver gap.

## Frequency overlay reference (advanced OC -- per-die validation required)

Per-die variance is significant; `gpu-turbo-*` overlays expose
out-of-spec rates but **must be validated per-die before deploying**.

| Overlay | Target | PLL setting | S805X die `0a153` | S905X-V2 die `82d7f` |
|---------|------:|-------------|-------------------|----------------------|
| (stock) | 666 MHz | fclk_div3 | 🟢 default | 🟢 |
| `gpu-turbo-744` | 744 MHz | M=62, OD=1 | UNTESTED (likely 🔴, well above 708 MHz panic line) | 🟢 stock cap |
| `gpu-turbo-792` | 792 MHz | M=66, OD=1 | 🔴 hangs board | UNTESTED |
| `gpu-turbo-828` | 828 MHz | M=69, OD=1 | 🔴 (extrapolated above 720 MHz panic) | 🟢 die ceiling |
| `gpu-turbo-1008` | 1008 MHz | M=42, OD=0 | 🔴 PANIC (SError 0x96000210) | UNTESTED |
| `gpu-turbo-1200` | 1200 MHz | M=50, OD=0 | 🔴 PANIC (same class) | UNTESTED |
| `gpu-turbo-1392` | 1392 MHz | M=58, OD=0 | UNTESTED | UNTESTED |
| `gpu-turbo-1584` | 1584 MHz | M=66, OD=0 | 🟡 one historical die (`6a44`) passed 2026-04-10, pre-canonical methodology | UNTESTED |

Apply via `ldto enable gpu-turbo-744`, then reboot. **Always verify
with all three checks** before trusting an OC rate:

1. `cat /sys/class/devfreq/*.gpu/cur_freq` (CCF target)
2. `sudo busybox devmem 0xc883c040 32` -- bit 31 must be 1 (PLL `LOCK`)
3. `sudo cat /sys/kernel/debug/meson-clk-msr/measure_summary | grep mali` -- HW measure within ±2 MHz of target

If any of the three disagrees with the target, the rate is invalid
even if `cur_freq` reports the requested value.

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
