# Team Win Recovery Project (TWRP) 3.7.0 for Infinix HOT 10T (`X689C`)

Official-grade custom recovery device tree for the **Infinix HOT 10T** (`X689C` / `Infinix-X689C` / `HOT10T`), powered by the **MediaTek Helio G80 (`MT6768`)** processor running **Android 11**.

---

## Device Specifications

| Feature | Specification |
| :--- | :--- |
| **Device Model** | Infinix HOT 10T (`X689C`) |
| **Codename** | `X689C`, `Infinix-X689C`, `HOT10T` |
| **Chipset / SoC** | MediaTek Helio G80 (`MT6768` / `MT6769V/CU`) |
| **CPU Architecture** | 8-core 64-bit ARM (2x Cortex-A75 @ 2.0GHz, 6x Cortex-A55 @ 1.8GHz) |
| **Android Version** | Android 11 (XOS 7.6 / AOSP 11) |
| **Kernel Version** | Linux `4.14.186` |
| **Partition Scheme** | Virtual A/B (VAB), Recovery-as-Boot |
| **Dynamic Partitions** | Yes (`super` container: `system`, `system_ext`, `vendor`, `product`) |
| **Display Resolution** | 720 x 1640 (20.5:9 aspect ratio, 90Hz) |
| **Brightness Range** | 12-bit hardware curve (`0` – `4095`) |

---

## Status & Verified Features

| Component / Feature | Status | Notes |
| :--- | :---: | :--- |
| **TWRP GUI Rendering** | ✅ Working | Double-buffered fbdev graphics at 720x1640 |
| **Touchscreen** | ✅ Working | Multi-touch input fully responsive |
| **Haptic Feedback / Vibrations** | ✅ Working | Tactile feedback on touch interactions |
| **12-bit Screen Brightness** | ✅ Working | Custom scaling (0–4095) via `/sys/class/leds/lcd-backlight/brightness` |
| **Dynamic Partitions Mounting** | ✅ Working | `system`, `system_ext`, `vendor`, `product` mapped cleanly |
| **MediaTek Protected Partitions** | ✅ Working | `metadata`, `tranfs`, `nvdata`, `nvcfg`, `protect_f`, `protect_s`, `persist` |
| **External Micro SD Card** | ✅ Working | Formatted as FAT32 / exFAT / NTFS (`/external_sd`) |
| **USB OTG Support** | ✅ Working | Mass storage mounting supported |
| **USB ADB (Shell & Sideload)** | ✅ Working | ConfigFS `musb-hdrc` gadget with instant auto-enumeration on boot |
| **USB MTP File Transfer** | ✅ Working | Switchable via TWRP Mount menu |
| **Full Partition Backups** | ✅ Working | Verified full backup engine with SHA-256 digests to MicroSD |
| **Internal Storage (`/data`)** | ⚠️ Encrypted | Stock Android uses FBE v2 + Beanpod TEE; MicroSD/OTG used for storage |

---

## The Road to Bringup: Technical Challenges & Solutions

Bringing up TWRP on the Infinix HOT 10T presented unique MediaTek-specific and Virtual A/B hurdles. If you are porting or compiling for this device (or similar MT6768 / MT6769 devices), here are the critical issues encountered and how they were solved:

### 1. The Kernel Physical Memory Offset Trap
* **The Issue:** `twrpdtgen` generated a `BoardConfig.mk` with default ARM64 base `0x40000000` and zero offsets. Booting resulted in an immediate silent reboot before kernel execution.
* **The Root Cause:** MediaTek Helio G80 bootloader (Little Kernel / LK) strictly enforces physical memory load addresses for the kernel, ramdisk, tags, and DTB.
* **The Fix:** Configured exact hardware offsets matching the factory Infinix boot header:
  ```make
  BOARD_KERNEL_BASE := 0x40000000
  BOARD_KERNEL_OFFSET := 0x00080000
  BOARD_RAMDISK_OFFSET := 0x07c80000
  BOARD_KERNEL_TAGS_OFFSET := 0x0bc80000
  BOARD_DTB_OFFSET := 0x0bc80000
  BOARD_KERNEL_PAGESIZE := 2048
  BOARD_BOOT_HEADER_VERSION := 2
  ```

### 2. Recovery-as-Boot Architecture
* **The Issue:** Attempting to build or flash a separate `recovery.img` will fail. The device has **no dedicated recovery partition**.
* **The Fix:** Recovery is embedded directly inside the `boot` partition ramdisk (`Recovery-as-Boot`):
  ```make
  BOARD_USES_RECOVERY_AS_BOOT := true
  AB_OTA_UPDATER := true
  BOARD_BUILD_SYSTEM_ROOT_IMAGE := false
  ```

### 3. Android Verified Boot (AVB) 1.0 Footer Requirement
* **The Issue:** Flashing an unsigned boot image caused Little Kernel to reject the partition and trigger an `LK_CRASH` panic loop.
* **The Fix:** Little Kernel requires an authentic AVB hash footer at the exact partition boundary (32MB / `33554432` bytes). Every build must be finalized using `avbtool`:
  ```bash
  python3 avbtool.py add_hash_footer \
    --image boot.img \
    --partition_size 33554432 \
    --partition_name boot
  ```

### 4. Little Kernel Panic (`LK_CRASH: 0x48534152435f4b4c`) on Slot B
* **The Issue:** Flashing TWRP to `boot_b` initially resulted in a Little Kernel crash.
* **The Fix:** Infinix's factory partition table left Slot B's VBMeta partitions uninitialized. Little Kernel checks `vbmeta_b`, `vbmeta_system_b`, and `vbmeta_vendor_b` when booting Slot B. Cloning the Slot A vbmeta partitions to Slot B resolved the crash completely:
  ```bash
  dd if=/dev/block/by-name/vbmeta_a of=/dev/block/by-name/vbmeta_b
  dd if=/dev/block/by-name/vbmeta_system_a of=/dev/block/by-name/vbmeta_system_b
  dd if=/dev/block/by-name/vbmeta_vendor_a of=/dev/block/by-name/vbmeta_vendor_b
  ```

### 5. AOSP 11 TWRP Build Conflict (`could not make way for new symlink: root/vendor`)
* **The Issue:** Compiling against the minimal AOSP 11 TWRP tree crashed at 99% during target image packaging due to a directory vs symlink conflict between `/root/vendor` and `system/vendor`.
* **The Fix:** Applied a pre-packaging exclusion filter in the GitHub Actions build script to remove the stale vendor directory before building `bootimage`:
  ```bash
  sed -i 's/--exclude=sdcard/--exclude=sdcard --exclude=vendor --exclude=\/root\/vendor/g' $(find build/ -name Makefile -path "*/core/*") || true
  mka bootimage || (rm -rf out/target/product/X689C/recovery/root/vendor && mka bootimage)
  ```

### 6. CPIO Ramdisk Ownership & Kernel Panic
* **The Issue:** When repacking the TWRP ramdisk locally, if files are packed under host user permissions (e.g., UID 1000), the Linux kernel panics at early init because non-root users cannot execute `/init`.
* **The Fix:** All ramdisk cpio archives MUST be created with forced `root:root` (0:0) ownership:
  ```bash
  find . | cpio -o -H newc -R 0:0 | gzip -9 > ../ramdisk.cpio.gz
  ```

### 7. USB ConfigFS Startup Race & Automatic Enumeration
* **The Issue:** On boot, the PC did not recognize the phone over ADB until MTP was manually disabled and re-enabled in TWRP's Mount menu.
* **The Root Cause:** In `init.recovery.usb.rc`, the startup trigger action executed `rm /config/usb_gadget/g1/configs/b.1/f1`. Because `f1` does not exist on a fresh boot, `rm` failed (`-ENOENT`). In Android `init`, **if any command in an action block fails, execution of the entire block halts**. The subsequent command that writes `musb-hdrc` to the UDC controller never ran.
* **The Fix:** Rewrote [init.recovery.usb.rc](recovery/root/init.recovery.usb.rc) to link `f1` to `ffs.adb` statically in `on fs`, and bound `musb-hdrc` unconditionally upon `sys.usb.ffs.ready=1`:
  ```rc
  on fs
      symlink /config/usb_gadget/g1/functions/ffs.adb /config/usb_gadget/g1/configs/b.1/f1
      start adbd

  on property:sys.usb.ffs.ready=1
      write /config/usb_gadget/g1/UDC "musb-hdrc"
      setprop sys.usb.state ${sys.usb.config}
  ```
  Now, ADB connects automatically within 2 seconds of boot without touching anything.

### 8. A/B Slot Switching & MediaTek Boot Control HAL
* **The Issue:** Tapping "Slot A" or "Slot B" in the TWRP Reboot menu failed with:
  ```text
  E:Error getting bootctrl module.
  ```
* **The Root Cause:** TWRP's GUI button calls `hw_get_module("bootctrl", ...)`. MediaTek Android 11 uses the HIDL HAL (`android.hardware.boot@1.0-impl-1.1-mtkimpl.so`), not the legacy wrapper.
* **The Workaround:** Slot switching is natively and safely handled by Little Kernel via Fastboot:
  ```bash
  fastboot --set-active=a  # Boot Stock Android OS
  fastboot --set-active=b  # Boot TWRP Recovery
  ```
  Or from within rooted Android using any A/B slot switcher app.

---

## Safe Dual-Boot Setup (Stock OS + TWRP)

Because this device uses Virtual A/B, you can maintain your **Stock Android OS on Slot A** and **TWRP on Slot B**, giving you an indestructible safety net:

### Flashing TWRP to Slot B:
```bash
# 1. Reboot to bootloader
adb reboot bootloader

# 2. Flash TWRP specifically to Slot B
fastboot flash boot_b boot.img

# 3. Set Slot B as active and reboot to TWRP
fastboot --set-active=b
fastboot reboot recovery
```

### Returning to Stock Android OS on Slot A:
```bash
# In Fastboot:
fastboot --set-active=a
fastboot reboot
```

---

## Emergency Unbrick / BROM Recovery

In the unlikely event of a hard brick (preloader or bootloop), the Helio G80 SoC supports hardware BROM bypass via `mtkclient`:

```bash
# 1. Clone mtkclient
git clone https://github.com/bkerler/mtkclient.git
cd mtkclient && pip3 install -r requirements.txt

# 2. Power off phone, hold Volume Up + Volume Down, and plug in USB
# 3. Restore stock boot image
python3 mtk.py w boot_a /path/to/stock_boot.img
python3 mtk.py da seccfg unlock
python3 mtk.py reset
```

---

## Building from Source

### Automated GitHub Actions CI (Recommended)
This repository includes a fully configured workflow at `.github/workflows/twrp.yml`. Simply fork the repository, enable Actions, and trigger the workflow to compile a release-ready `boot.img`.

### Local Compilation (Ubuntu 20.04 / 22.04)
```bash
# 1. Initialize TWRP AOSP 11 manifest
mkdir twrp-11 && cd twrp-11
repo init -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b twrp-11
repo sync -c -j$(nproc --all) --force-sync --no-clone-bundle --no-tags

# 2. Clone this device tree
git clone https://github.com/idkreally001/twrp_infinix_X689C.git device/infinix/X689C

# 3. Set up build environment
source build/envsetup.sh
lunch omni_X689C-eng

# 4. Compile bootimage
mka bootimage
```

---

## Credits & Acknowledgements

* **TeamWin** for the TWRP recovery project.
* **Infinix Mobility** for the stock Android 11 firmware and MT6768 kernel.
* **bkerler** for the invaluable `mtkclient` BROM utility.
* **Google DeepMind / Antigravity Pair-Programming** for end-to-end hardware debugging, AVB analysis, and bringup.
