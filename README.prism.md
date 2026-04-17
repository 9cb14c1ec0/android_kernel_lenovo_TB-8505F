# Prism fork — Linux 4.9 for Lenovo TB-8505F (MT6761)

Fork of [T4rp/Lenovo-TB-8505FS-Kernel](https://github.com/T4rp/Lenovo-TB-8505FS-Kernel) (Lenovo's GPL kernel drop) with the patches needed to build cleanly against AOSP 12's toolchain and boot into TWRP on the Lenovo Tab M8 FHD.

Upstream README is at `README` (kernel's own). This file documents the Prism deltas.

## Defconfig

Use `akita_row_wifi_defconfig`. The `TB-8505F_defconfig` named in the twrpdtgen BoardConfig doesn't exist in the Lenovo source drop — the real config is named after the board project (`akita_row_wifi`).

Two drivers are disabled vs upstream for build-time reasons (neither is needed for TWRP):
- `CONFIG_SND_SOC_AC107=n` — uses `-Wno-incompatible-pointer-types` (GCC 4.9 doesn't know it).
- `CONFIG_INPUT_SAR_SX9311=n` — uninitialised `reg_val` that GCC 4.9 flags as `-Werror=maybe-uninitialized`.

## Changes vs upstream

### 1. Python 2 → Python 3 in `tools/dct/`

MTK's DrvGen (Driver Generator) toolchain runs during `dtbs` build to produce `cust.dtsi` from `.dws` config. It was written for Python 2 and fails on modern distros where `python` is Python 3.

Patches applied:
- `print x` → `print(x)` (via `lib2to3`).
- `cmp(a, b)` replaced with a shim `def cmp(a,b): return (a>b)-(a<b)` at the top of each file using it.
- `string.atoi` → `int`, `string.atof` → `float`, `string.atol` → `int`.
- `configparser.ConfigParser(...)` now takes `strict=False` (Python 3 rejects duplicate option keys; the upstream `.cmp` config files have duplicates).
- In `tools/dct/data/EintData.py` and `tools/dct/obj/{Gpio,Eint}Obj.py`, a local variable named `list` shadowed the builtin — renamed to `lst` so `list(map.items())` (introduced by 2to3) doesn't collide.

### 2. `-Werror` tamed

GCC 4.9 is noisier than clang about a few idioms; several are set `-Werror=foo` by specific driver Makefiles in the tree. Added to top-level `Makefile`:

```
KBUILD_CFLAGS += -Wno-error \
                 -Wno-error=maybe-uninitialized \
                 -Wno-error=format -Wno-error=format= \
                 -Wno-error=format-security \
                 -Wno-error=unused-variable \
                 -Wno-error=unused-function \
                 -Wno-error=unused-but-set-variable
```

`sound/soc/mediatek/Makefile` had `subdir-ccflags-y += -Werror -Wno-incompatible-pointer-types` hardcoded — the `-Wno-incompatible-pointer-types` is unknown to GCC 4.9 and, under `-Werror`, fatal. Stripped from every `subdir-ccflags-y` line.

### 3. Floating-point in kernel code removed

Two s5k4h7 camera drivers had buggy `pow1` / `pow2` helpers declared `double pow(double, int)`. The kernel builds with `-mgeneral-regs-only` — the FP registers belong to userspace and aren't saved across kernel entry, so any FP instruction triggers `sorry, unimplemented: '-mgeneral-regs-only' and floating point code`.

The bodies don't actually compute `pow` — they're `c = a*b;` in a loop, always returning `2*j`. Rewritten with `int` signatures; callers' `pow(2.0, j)` → `pow(2, j)`.

Files:
- `drivers/misc/mediatek/imgsensor/src/common/v1/s5k4h7_mipi_raw/s5k4h7mipiraw_Sensor.c`
- `drivers/misc/mediatek/imgsensor/src/common/v1/s5k4h7_qtech_mipi_raw/s5k4h7qtechmipiraw_Sensor.c`

### 4. NULL guards for `ram_console` & `cpuidle_fp`

MTK's ram_console persistent-log driver expects the bootloader to reserve DRAM via a `/chosen/ram_console` DTB property. If that property isn't there (and our bootimg DTB doesn't have it), `ioremap_wc()` may still be called but leaves `ram_console_buffer` (virtual mapping) NULL while `ram_console_buffer_pa` is set. That triggers a NULL deref in anything that dereferences the macro `RR_LINUX_PA` (which goes through both pointers).

Symptom: kernel oops at `aee_rr_rec_fiq_cache_step_pa+0x14` called from `fiq_cache_init` at ~1.3s into boot. Device resets before TWRP init runs.

Fix in `drivers/misc/mediatek/ram_console/mtk_ram_console.c`:
- `if (ram_console_buffer_pa)` → `if (ram_console_buffer_pa && ram_console_buffer)` in the 3 `_pa` accessors.
- Moved `ram_console_buffer_pa` assignment inside the `ioremap_wc()` success branch, so a failed map doesn't leave a stale `_pa` around.

Same pattern in `drivers/misc/mediatek/base/power/cpuidle_v3/mtk_cpuidle.c`. The `cpuidle_fp()` inline assumes `cpuidle_fp_va` is always valid; secondary CPUs entering idle for the first time after TWRP starts would then NULL-deref at offset `cpu * 4`. Wrapped the two inlines in `if (cpuidle_fp_va)`.

## Building

Not meant to be built standalone — built as part of a TWRP 12.1 source tree. See the [device tree README](https://git.offlinesoftware.solutions/Prism/android_device_lenovo_TB-8505F) for the full bring-up.
