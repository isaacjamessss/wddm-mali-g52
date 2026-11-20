# wddm-mali-g52, gs101 branch: Mali-G78 on the Google Pixel 6

A Windows (WDDM) driver for the Arm Mali-G78 MP20 GPU in the Google Pixel 6
(`oriole`, Tensor gs101), running Windows 11 on ARM on the phone itself.

This is a fork of [gahingwoo/wddm-mali-g52](https://github.com/gahingwoo/wddm-mali-g52),
gahingwoo's Windows driver for the Mali-G52 in the Rockchip RK3576. The whole
design comes from there: Mesa's Panfrost as the Direct3D and OpenGL user-mode
drivers, the libdrm shim in [win32-compat/](win32-compat/) and the WDDM 1.3
kernel driver that owns the display. All credit for making a Mali draw under
Windows in the first place goes to that project; this branch ports it to the
Pixel 6's Valhall GPU and display.

## What works on the Pixel 6

- **Desktop composited on the GPU**: DWM renders straight into two
  framebuffers that the display controller (DECON/DPP0) flips between, with
  no CPU copies, and no flicker (flips complete only once the display has
  latched them).
- **Direct3D 10/11** (feature level 10.1) through Mesa's d3d10umd on Panfrost.
- **OpenGL 3.1** on the GPU through Mesa's WGL frontend, for ARM64, x64 and
  x86 programs (ARM64X forwarders). Minecraft 1.12.2 runs natively on ARM64.
- **Landscape and portrait**, rotated by Windows (source modes stay
  panel-sized on a rotated path, as in Microsoft's KMDOD sample).
- **Brightness** from the Windows slider (MIPI DCS to the s6e3fc3 OLED).
- **GPU clock governor** (151-848 MHz by load; ACPM scales the GPU voltage).
- **Screen off** for the power key (with pixelbuttons).

## Branch contents

- [mesa-patches/](mesa-patches/): upstream's patches plus the gs101 ones:
  G78 (Valhall) support in the Windows build, shader compiles on a 16 MiB
  stack, landscape by a GPU rotation (0073), WGL present by a GPU blit with
  an exact swap interval (0074), AFBC textures updated in parts by the CPU
  (0075).
- [win32-compat/](win32-compat/): the libdrm shim, with completion events on
  the D3DKMT path so OpenGL programs no longer wait for the whole GPU queue.
- CI builds the user-mode DLLs (`maliumd_*` for Direct3D, `maligl_*` for
  OpenGL) for ARM64, x64 and x86.

The Pixel 6 kernel driver (`maliwddm` 0.9.11 for gs101) and a ready-to-install
package with these DLLs are in
[isaacjamessss/pixel6-woa-drivers](https://github.com/isaacjamessss/pixel6-woa-drivers)
(`maliwddm/`), next to the phone's other drivers (touch, Wi-Fi, UFS, audio,
battery, thermal, buttons).

## Hardware

| | |
|---|---|
| SoC | Google Tensor (gs101) |
| GPU | Mali-G78 MP20 (Valhall, job manager) |
| Registers | `0x1c500000` |
| Interrupts | ACPI GSIV 452, 453, 451 |
| Power | GPU rails set through ACPM by the driver at start |
| Display | Samsung s6e3fc3 OLED, 1080x2400, DSIM0, DECON0/DPP0 |

There is no ACPI on the stock phone. Windows boots through **acpiboot**, a
custom UEFI loader that provides the ACPI tables (`ACPI\PXLG0001`, `GPU0`),
lays out both framebuffers and hands over to Windows Boot Manager. Test
signing must be on.

## Known issues

- Vsync is emulated by a 16 ms timer (the panel's TE is not wired up).
- The display controller cannot rotate RGB, so landscape costs a rotation on
  the GPU.
- OpenGL frames still reach the window through a GPU blit and a CPU copy
  (GDI). x64 OpenGL programs run under emulation and are much slower than
  ARM64 ones.
- Never send the panel DCS display off (0x28): the picture does not come back
  until a reboot.

## Upstream

The original project, its milestones and its RK3576 results (the first
Windows desktop composed on a Mali, FurMark through OpenGL) are on the
[`main` branch](https://github.com/gahingwoo/wddm-mali-g52/tree/main) and in
[docs/](docs/README.md).

## License

GPL-2.0, see [LICENSE](LICENSE), as upstream. Code taken from Mesa keeps its
own licence, mostly MIT.
