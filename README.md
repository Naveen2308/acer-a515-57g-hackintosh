# Acer Aspire A515-57G — OpenCore EFI (macOS Sequoia)

Working OpenCore EFI for the **Acer Aspire A515-57G** (Alder Lake-P, Iris Xe) running
**macOS Sequoia 15.8.2 (24H232)** with a native internal panel, working Intel AX211
Wi-Fi and Bluetooth.

> ⚠️ **Read [SMBIOS: generate your own](#smbios-generate-your-own) before booting.**
> The serial number, board serial (MLB) and UUID in `config.plist` have been blanked
> on purpose — they must be unique to your machine.

---

## Status

| Feature | State | Notes |
|---|---|---|
| Internal display | ✅ Working | Native Iris Xe framebuffer, Metal 2, 1536 MB VRAM |
| External HDMI / USB-C DP | ✅ Working | via `WhateverXe.kext` |
| Graphics acceleration | ✅ Working | QE/CI, no `framebuffer-stolenmem` needed (DVMT 64 MB) |
| Wi-Fi | ✅ Working | Intel AX211 via `AirportItlwm` — 2.4/5 GHz only (see [Wi-Fi](#wi-fi-limits)) |
| Bluetooth | ✅ Working | Intel AX211 (8087:0033) — see [docs/BLUETOOTH.md](docs/BLUETOOTH.md) |
| Audio | ✅ Working | `alcid=11` (speakers, headphones, mic) |
| Ethernet | ✅ Working | Realtek RTL8111 |
| Keyboard / trackpad | ✅ Working | VoodooPS2 + VoodooI2C (I2C HID) |
| Battery status | ✅ Working | `SMCBatteryManager` + `ECEnabler` |
| USB ports | ✅ Working | `SSDT-USBX`, no port-limit patch |
| Sleep / wake | ⚠️ Not verified | Not tested on this build |
| Webcam | ⚠️ Not verified | Not tested |
| AirDrop / Handoff / Continuity / Universal Clipboard | ❌ Not working | `AirportItlwm` provides no `awdl0` |
| Sidecar | ❌ Not working | Needs `awdl0` |
| iMessage / FaceTime | ⚠️ Requires own SMBIOS | Works only with valid, unique serial + MLB |

---

## Hardware

| Component | Model |
|---|---|
| Laptop | Acer Aspire A515-57G (subsystem `1025:1616`) |
| CPU | Intel Core i5-1240P (Alder Lake-P, 4P + 8E) |
| iGPU | Intel Iris Xe, PCI `8086:46A6` rev `0x0C` (Gen12.2) |
| Panel | CMN N156HCA-EAB (CMN15E7), 15.6" 1920×1080, 8 bpc, 2-lane eDP 1.2 |
| DVMT | 64 MB preallocated (GGC `0x02C1` → GMS `0x02`), 256 MB aperture |
| Wi-Fi / BT | Intel AX211 (Wi-Fi `8086:51f0`, BT USB `8087:0033`) |
| Ethernet | Realtek RTL8168/8111 |
| Audio | Realtek ALC256 (layout 11) |
| Firmware | InsydeH2O, Secure Boot off, CFG Lock unlocked |

---

## Repository layout

```
EFI/
├── BOOT/
│   ├── bootx64.efi
│   ├── fbx64.efi
│   └── mmx64.efi
├── OC/
│   ├── OpenCore.efi            OpenCore 1.0.8
│   ├── config.plist            sanitized (SMBIOS blanked)
│   ├── ACPI/                   8 SSDTs + patched DSDT
│   ├── Drivers/                HfsPlus, OpenRuntime, ResetNvramEntry
│   ├── Kexts/                  28 kexts (see table below)
│   └── Tools/                  OpenShell.efi
└── UpdateCapsule/              empty
docs/
├── BLUETOOTH.md                how the AX211 Bluetooth fix works + build recipe
└── GRAPHICS.md                 Iris Xe / eDP notes
```

---

## Install

1. Mount your ESP (the EFI System Partition) and **back up** the existing `EFI` folder.
2. Copy `EFI/BOOT` and `EFI/OC` from this repo onto the ESP.
   *If you dual-boot Windows, do **not** delete `EFI/Microsoft`. It is not part of this repo.*
3. Generate your own SMBIOS (below) and put the values into `config.plist`.
4. Reboot. First boot with `-v` (already in `boot-args`) so you can see where it stops.

Nothing in this repo needs to be installed into `/Library/Extensions` or `/System/Library/Extensions`.
Everything is injected by OpenCore.

---

## SMBIOS: generate your own

`PlatformInfo → Generic` in `config.plist` is intentionally blanked:

| Key | Value in this repo |
|---|---|
| `SystemProductName` | `MacBookPro16,2` (keep — this is the SMBIOS model, not personal data) |
| `SystemSerialNumber` | *(empty)* |
| `MLB` | *(empty)* |
| `SystemUUID` | *(empty)* |
| `ROM` | `00000000 0000` |

Generate a valid set for **MacBookPro16,2** with
[GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) and paste serial, MLB and UUID into
`PlatformInfo → Generic`. `ROM` is derived from your Ethernet MAC — either set it to your
real MAC bytes or leave the zeros; it does not affect booting, only iServices.

Booting with an empty serial works, but iMessage/FaceTime/App Store activation will not.

---

## Kexts

| Kext | Version | Purpose |
|---|---|---|
| Lilu | 1.7.3 | kext patching framework (must be first) |
| VirtualSMC | 1.3.8 | SMC emulation |
| SMCProcessor | 1.3.8 | CPU sensors (disabled by default) |
| SMCSuperIO | 1.3.8 | fan / voltage sensors |
| SMCBatteryManager | 1.3.8 | battery percentage |
| SMCLightSensor | 1.3.8 | ambient light sensor |
| SMCDellSensors | 1.3.8 | Dell sensor set (harmless here) |
| ECEnabler | 1.0.6 | battery EC field widths |
| WhateverXe | 0.8.2 | Iris Xe (Gen12.2) graphics driver |
| AppleIntelTGLCompat | 0.7.0 | TGL graphics compatibility shim (disabled) |
| CpuTopologyRebuild | 2.0.2 | correct P/E core topology |
| AppleALC | 1.9.8 | audio (layout 11) |
| AppleALCU | 1.9.8 | USB audio variant |
| RealtekRTL8111 | 3.0.4 | Ethernet |
| AirportItlwm | 2.4.0 | Intel AX211 Wi-Fi |
| **IntelBluetoothFirmware** | **2.5.0** | uploads AX211 firmware (built from master) |
| **IntelBTPatcher** | **2.5.0** | Sequoia LE-scan fix (built from master) |
| **BlueToolFixup** | **2.7.2** | allows `bluetoothd` to start on unsupported hardware |
| AMFIPass | 1.4.1 | AMFI bypass for `WhateverXe` |
| RestrictEvents | 1.1.5 | hides unsupported CPU warnings |
| VoodooPS2Controller | 2.3.7 | PS/2 keyboard (+ mouse/trackpad plugins, disabled) |
| VoodooI2C | 2.9.1 | I2C HID trackpad |
| VoodooI2CHID | 1 | I2C HID devices |
| VoodooI2CSynaptics / AtmelMXT / ELAN / FTE | 1 | I2C touchpad controllers |

Load order matters: **Lilu → IntelBluetoothFirmware → IntelBTPatcher → BlueToolFixup → everything else.**

---

## boot-args

```
-wex-edp-keep -wex-no-edp-hpd -v keepsyms=1 debug=0x100 -allow3d
-disablegfxfirmware -vi2c-force-polling alcid=11
amfi_get_out_of_my_way=1 revcpu=1
```

| Arg | Meaning |
|---|---|
| `-allow3d` | start the native Iris Xe framebuffer |
| `-wex-edp-keep` | keep the firmware-trained eDP link (no retrain) |
| `-wex-no-edp-hpd` | mask panel HPD-A (prevents HPD storm after training) |
| `-v` `keepsyms=1` `debug=0x100` | verbose boot, panic symbolication, no watchdog |
| `-disablegfxfirmware` | skip Intel GuC/HuC firmware |
| `-vi2c-force-polling` | I2C polling for the touchpad |
| `alcid=11` | AppleALC layout for ALC256 |
| `amfi_get_out_of_my_way=1` | allow ad-hoc signed kexts |
| `revcpu=1` | correct CPU name in About This Mac |

Deliberately **not** used: `framebuffer-stolenmem` (DVMT is already 64 MB — overriding it
corrupts memory), `-igfxdvmt`, `-wex-tcN-legacy`, `-wex-retrain`.

---

## Wi-Fi limits

`AirportItlwm` 2.4.0 drives the AX211, but the 802.11ax (HE) path is not enabled in the
current driver build and 6 GHz sits behind `#ifdef notyet`. Practically:

* 2.4 GHz and 5 GHz work, at 802.11ac rates.
* No 6 GHz, no 160 MHz on 5 GHz.
* No `awdl0` → no AirDrop / Handoff / Continuity / Sidecar.

Bluetooth is fully independent of this limitation.

---

## Troubleshooting

**Bluetooth shows `State: Off` with chipset `BCM_4350C2` / `Transport: UART`.**
That is the fallback state when no Intel BT kext was injected. Check:

```bash
kmutil showloaded | grep -iE "zxystd|BlueTool"
```

If `IntelBluetoothFirmware` / `IntelBTPatcher` are missing, see
[docs/BLUETOOTH.md](docs/BLUETOOTH.md) — there are three separate traps.

**A kext is silently ignored.** OpenCore prepends `Kexts\` to `BundlePath` itself, so
`Kernel/Add` must contain `IntelBluetoothFirmware.kext`, **not** `Kexts/IntelBluetoothFirmware.kext`.

**Enable OpenCore file logging** to see injection results: `Misc → Debug → Target = 67`
writes `opencore-YYYY-MM-DD-HHMMSS.txt` to the ESP root.

---

## Credits

* [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg) — bootloader
* [Lilu](https://github.com/acidanthera/Lilu), [VirtualSMC](https://github.com/acidanthera/VirtualSMC), [AppleALC](https://github.com/acidanthera/AppleALC), [BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) (BlueToolFixup)
* [WhateverXe](https://github.com/jkbuha/WhateverXe-contrib) — Iris Xe driver
* [OpenIntelWireless](https://github.com/OpenIntelWireless) — AirportItlwm, IntelBluetoothFirmware
* [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C), [VoodooPS2](https://github.com/acidanthera/VoodooPS2)
* [CpuTopologyRebuild](https://github.com/b00t0x/CpuTopologyRebuild), [ECEnabler](https://github.com/1Revenger1/ECEnabler)

## Disclaimer

This is a personal configuration shared as-is. Hackintosh setups are machine- and
firmware-specific; flashing an EFI that does not match your hardware can prevent the
machine from booting. Keep a backup of your working ESP and a USB recovery drive.
No warranty of any kind.

---

Licensed under the Apache License 2.0 — see [LICENSE](LICENSE).
