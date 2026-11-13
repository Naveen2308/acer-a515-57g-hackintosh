# Iris Xe (Gen12.2) graphics notes — A515-57G

The internal panel is driven by [WhateverXe](https://github.com/jkbuha/WhateverXe-contrib)
0.8.2 on top of Apple's TGL framebuffer. This machine needs a specific eDP setup; the
notes below are what makes it work and what breaks it.

## Configuration

| Item | Value |
|---|---|
| iGPU | `8086:46A6` rev `0x0C`, Iris Xe Gen12.2 |
| Panel | CMN N156HCA-EAB (CMN15E7), 1920×1080, 8 bpc |
| Link | 2-lane eDP 1.2, firmware-trained |
| DVMT | 64 MB preallocated (`GGC 0x02C1` → `GMS 0x02`), 256 MB aperture |
| Framebuffer | native, Metal 2, 1536 MB VRAM |
| WhateverXe.kext | 0.8.2 (`org.b00t0x.WhateverXe`) |
| AppleIntelTGLCompat.kext | 0.7.0 in `EFI/OC/Kexts` (entry **disabled**) |

The `AppleIntelTGLCompat` entry is disabled in `config.plist` on purpose: the newer
graphics support package is installed into macOS itself rather than injected from the ESP.
If you are building from scratch, enable it and install the matching package.

## Required boot-args

```
-allow3d -wex-edp-keep -wex-no-edp-hpd -disablegfxfirmware
```

| Arg | Effect |
|---|---|
| `-allow3d` | start the native framebuffer (without it: no acceleration) |
| `-wex-edp-keep` | keep the firmware-trained eDP link, do not retrain |
| `-wex-no-edp-hpd` | keep panel HPD-A masked, prevents an HPD storm after training |
| `-disablegfxfirmware` | skip GuC/HuC firmware loading |

## What NOT to add

| Option | Why not |
|---|---|
| `framebuffer-stolenmem` | DVMT is already 64 MB; setting a lower value corrupts memory |
| `-igfxdvmt` | WhateverXe reads GGC (`0x02C1`) directly |
| `-wex-tcN-legacy` | only needed for VBTs that mislabel a fixed DP port |
| `-wex-retrain` | retrains the eDP link and produces a black panel on this unit |

CMTG handover is not used: `TRANS_DDI_FUNC_CTL2` (`0x60404`) reads `0x0` here, so there is
no firmware CMTG secondary to preserve.

## Known quirks

* After the first native boot the Dock process was wedged once (no Dock visible).
  `killall Dock` fixed it and it has not recurred.
* External output works over HDMI and USB-C DP. Hot-plug detection on the USB-C port is
  handled by `-wex-no-edp-hpd` only for the internal panel — external HPD is untouched.

## Recovering a black panel

Boot with `-v` and no `-allow3d` to get the VESA framebuffer back, then remove the last
graphics change. Keep the previous `config.plist` next to the new one on the ESP — the
OpenCore picker cannot help you if the panel never lights up.
