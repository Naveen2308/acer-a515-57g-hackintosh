# Intel AX211 Bluetooth on macOS Sequoia

Getting Intel Bluetooth working on Sequoia has three traps that all look the same from the
outside: `system_profiler SPBluetoothDataType` reports

```
Address: NULL
State: Off
Chipset: BCM_4350C2
Transport: UART
```

That is the generic fallback state — macOS is describing a controller it never reached.
The USB device itself is present the whole time:

```
Bluetooth HCI:
  Product ID: 0x0033
  Vendor ID:  0x8087  (Intel Corporation)
  Built-In:   Yes
```

## The three kexts

| Kext | Why it is needed |
|---|---|
| `IntelBluetoothFirmware` | uploads `ibt-0041-0041.sfi` to the controller and renames the USB host controller so `bluetoothd` accepts it |
| `IntelBTPatcher` | patches the LE-scan path; required on Monterey and newer |
| `BlueToolFixup` | from BrcmPatchRAM — lets `bluetoothd` start on hardware Apple does not list |

**Do not use `IntelBluetoothInjector.kext`** — it is for macOS 10.14 and older and is
actively harmful on Monterey+.

Load order in `Kernel → Add`:

```
Lilu.kext
IntelBluetoothFirmware.kext
IntelBTPatcher.kext
BlueToolFixup.kext
```

## Trap 1 — `BundlePath` must not include `Kexts/`

OpenCore already prepends the kext directory. From `Library/OcMainLib/OpenCoreKernel.c`:

```c
#define OPEN_CORE_KEXT_PATH  L"Kexts\\"
...
OcUnicodeSafeSPrint (FullPath, sizeof (FullPath),
  IsForced ? L"%a\\%a" : OPEN_CORE_KEXT_PATH "%a\\%a",
  BundlePath, PlistPath);
```

So an entry of `Kexts/IntelBluetoothFirmware.kext` makes OpenCore look for

```
EFI/OC/Kexts/Kexts/IntelBluetoothFirmware.kext/Contents/Info.plist
```

which does not exist. OpenCore logs `OC: Plist ... is missing for injected kext` and
**skips the kext without failing the boot**. Every other kext in the config uses the plain
name, which is why only the Bluetooth ones silently vanished.

Correct:

```xml
<key>BundlePath</key><string>IntelBluetoothFirmware.kext</string>
```

## Trap 2 — `kexts must have a _kmod_info symbol`

The official `IntelBluetooth-v2.4.0.zip` release predates Sequoia, so these kexts have to be
built from master. On a machine with only Command Line Tools (`xcodebuild` unavailable)
the manual `clang` link produces a kext that macOS refuses:

```
cannot be placed in kernel collection because: kexts must have a _kmod_info symbol
```

`_kmod_info` (plus `__realmain` / `__antimain`) is normally injected by **`kextsymboltool`**,
which ships only with full Xcode. Without it the link succeeds but the binary is not a
valid kext.

Fix: link `MacKernelSDK/Library/x86_64/libkmod.a` (it provides `__start` / `__stop` and
`OSKextGetCurrent*`) together with a small stub that defines `_kmod_info` and the two
function pointers:

```cpp
// kmod_ibf.cpp
#include <mach/kmod.h>

extern "C" {
extern "C" kern_return_t _start(kmod_info_t *, void *);
extern "C" kern_return_t _stop(kmod_info_t *, void *);

static kern_return_t ibf_kern_start(kmod_info_t *, void *) { return KERN_SUCCESS; }
static kern_return_t ibf_kern_stop(kmod_info_t *, void *) { return KERN_SUCCESS; }

__attribute__((used)) kmod_start_func_t *_realmain = ibf_kern_start;
__attribute__((used)) kmod_stop_func_t  *_antimain = ibf_kern_stop;

__attribute__((visibility("default")))
KMOD_EXPLICIT_DECL(com.zxystd.IntelBluetoothFirmware, "2.5.0", _start, _stop)
}
```

Link with:

```bash
clang++ -mkernel -fapple-kext -nostdlib -Wl,-kext \
  -o IntelBluetoothFirmware <all objects> kmod_ibf.o \
  MacKernelSDK/Library/x86_64/libkmod.a
```

Verify before installing:

```bash
nm -m IntelBluetoothFirmware | grep -E "kmod_info|__realmain|__antimain"
#  __DATA,__data  external _kmod_info
#  __DATA,__data  external __realmain
#  __DATA,__data  external __antimain
```

`kmod_info.start` must point at the libkmod `__start` wrapper (which jumps through
`__realmain`), exactly like a stock acidanthera kext.

## Trap 3 — unexpanded build-setting macros in Info.plist

Xcode substitutes `$(PRODUCT_BUNDLE_PACKAGE_TYPE)` and friends at build time. If you copy
the raw `Info.plist` from the source tree, `CFBundlePackageType` stays literal and the
kext fails with:

```
Unable to create extension from bundle: executable Unsupported ($(PRODUCT_BUNDLE_PACKAGE_TYPE))
```

Substitute at minimum:

| Key | Value |
|---|---|
| `CFBundleExecutable` | `IntelBluetoothFirmware` |
| `CFBundleIdentifier` | `com.zxystd.IntelBluetoothFirmware` |
| `CFBundleName` | `IntelBluetoothFirmware` |
| `CFBundlePackageType` | `KEXT` |
| `CFBundleVersion` | `2.5.0` |
| `CFBundleShortVersionString` | `2.5.0` |

## Why master and not the 2.4.0 release

Sequoia changed how the HCI memory descriptor is prepared. The fix landed after v2.4.0:

```
IntelBTPatcher: compat for Sequoia, different from other version os macOS,
the HCI memory descriptor is not prepared and can not be used directly.
```

So build from `OpenIntelWireless/IntelBluetoothFirmware` master (which also carries
"bump support version to Tahoe"), not from the release zip.

## Build recipe (no Xcode)

```bash
git clone --depth=1 https://github.com/OpenIntelWireless/IntelBluetoothFirmware.git
cd IntelBluetoothFirmware
git clone --depth=1 https://github.com/acidanthera/MacKernelSDK.git

# generate FwBinary.cpp from IntelBluetoothFirmware/fw/*.sfi|.ddc
python3 -c 'import sys;sys.path.append("scripts/");import zlib_compress_fw;
zlib_compress_fw.process_files("IntelBluetoothFirmware/FwBinary.cpp","IntelBluetoothFirmware/fw/")'

SDK=$(xcrun --show-sdk-path)
FLAGS="-x c++ -std=gnu++17 -fapple-kext -fno-rtti -fno-exceptions -fno-common \
       -mkernel -DKERNEL -isysroot $SDK -mmacosx-version-min=10.13 -I MacKernelSDK/Headers"

for f in BtIntel BtIntelFw BtIntelVSC IntelBluetoothFirmware \
         IntelBluetoothOpsGen1 IntelBluetoothOpsGen2 IntelBluetoothOpsGen3 \
         USBDeviceController; do
  clang++ -c $FLAGS -I IntelBluetoothFirmware -o obj/$f.o IntelBluetoothFirmware/$f.cpp
done
clang++ -c $FLAGS -O1 -I IntelBluetoothFirmware -o obj/FwBinary.o IntelBluetoothFirmware/FwBinary.cpp
clang++ -c $FLAGS -I IntelBluetoothFirmware -o obj/zutil.o IntelBluetoothFirmware/zutil.c
```

`IntelBTPatcher` additionally needs Lilu headers (`plugin_start.cpp` lives inside
`Lilu.kext/Contents/Resources/Library/`).

Notes:

* `-DKERNEL` is required, otherwise `MacKernelSDK/Headers/USBDriverKit/*.h` is parsed in
  user-space mode and fails with `unknown type name 'OSClassLoadInformation'`.
* `zutil.c` contains `extern "C" { ... }`, so compile it as C++ — not as C.
* Sign the finished bundles ad-hoc: `codesign --force --sign - <kext>`.

## Verify after reboot

```bash
kmutil showloaded | grep -iE "zxystd|BlueTool"
system_profiler SPBluetoothDataType
```

Expected:

```
com.zxystd.IntelBluetoothFirmware (2.5.0)
com.zxystd.IntelBTPatcher (2.5.0)
as.acidanthera.BlueToolFixup (2.7.2)
```

and `State: On` with a real controller address and `Chipset: BCM_4350C2` replaced by the
Intel chipset name.

Live-loading a kext to test it is **not** equivalent to a boot load:

* `kmutil load` of an ad-hoc signed kext fails with
  `not approved to load. Please approve using System Settings.` — OpenCore-injected kexts
  skip this gate because they are in the boot kernel collection.
* `kmutil load` of `IntelBTPatcher` alone fails with
  `could not find a kext with 'as.vit9696.Lilu' bundle-id` — the dependency is present at
  boot but not in the temporary collection `kmutil` builds.

## What Bluetooth gives you here

* Bluetooth audio (A2DP), mice, keyboards, gamepads, phones for tethering/file transfer
* BLE devices (subject to Apple's own Intel-BT quirks)

Not available, because they need `awdl0` from the Wi-Fi driver:
AirDrop, Handoff, Continuity, Universal Clipboard, Sidecar.
