# Alldocube iPlay 70 Mini Pro (T813)
## Native LineageOS Port — Research Notes

**Project status:** Hardware and firmware investigation  
**Target:** Native LineageOS device port  
**Device:** Alldocube iPlay 70 Mini Pro  
**Model:** T813

---

## 1. Platform identification

| Component | Finding |
|---|---|
| Marketing chipset name | MT8791 |
| Actual platform identifier | MT6877 |
| Device tree model | MT6877 |
| Device tree compatible | `mediatek,MT6877` |
| CPU architecture | ARM64 |
| Android version | Android 15 |
| Kernel | 6.6.50-android15-8 |
| GPU | ARM Mali-G68 MC4 |
| GPU driver | Mali r49 |
| Display resolution reported by Android | 1200×1920 |
| Supported refresh rates | 60 Hz and 90 Hz |

The MT6877 identification is confirmed by Android properties and the extracted device tree.

## 2. Boot architecture

The device uses:
- A/B slots
- Dynamic partitions
- Virtual A/B
- AVB
- `boot`
- `init_boot`
- `vendor_boot`
- `dtbo`
- `super`

Current active slot: `_a`

Extracted images:
- `boot_a.img`
- `init_boot_a.img`
- `vendor_boot_a.img`
- `dtbo_a.img`

The vendor boot image contains:
- Kernel-independent vendor ramdisk fragments
- MT6877 fstab files
- Kernel module loading configuration
- Vendor init scripts
- Device tree table containing one DTB

The separate DTBO image contains one overlay table entry.

## 3. Kernel investigation

Kernel release reported by the running system:

`6.6.50-android15-8-gba6679cc6fdc-ab12886967-4k`

Vendor modules include MT6877-specific drivers for:
- Mali GPU
- MediaTek display pipeline
- CPU scheduling and DVFS
- Thermal management
- Wi-Fi and Bluetooth
- Camera
- Audio
- Touchscreen

Most vendor modules examined so far are loaded in the running system.

**Unresolved:** Kernel source availability, exact source provenance, configuration and compatibility with a future LineageOS build.

Some module files report a different vermagic (`6.6.56`). This discrepancy needs investigation.

## 4. Display hardware

### Confirmed

- Display controller: MT6877 DSI0
- DSI controller address: `0x14013000`
- DSI PHY: D-PHY
- Display subsystem: MT6877 MMSYS
- Display resolution reported by Android: 1200×1920
- Supported refresh rates: 60 Hz and 90 Hz
- Backlight: MediaTek PWM, `lcd-backlight`
- Bias driver dependency: RT4831A

### Panel investigation

The DTBO bootloader property identifies:

`td4330_fhdp_dsi_cmd_auo_rt4831_drv`

The vendor kernel contains:

`panel_truly_td4330_cmd`

The bootloader identifier and vendor kernel module reference TD4330.
However, this conflicts with the runtime LCM configuration name.
The physical panel identity is not conclusively established.

The DTBO also contains two alternative 1200×2000 panel profiles:

- `m100t_ft8201_xy2210d4001a_wuxga2000_pt2`
- `m100t_jd9366tc_sq110aq7ez51394l501_wuxga2000_jst_864`

Both specify four DSI lanes, RGB888, command mode and 60 FPS.

Their presence does not establish that either is the active Android panel configuration.

### Runtime panel identification (confirmed)

Obtained from the running device tree:

- DSI controller: `dsi@14013000`
- MIPI-DSI device: `14013000.dsi.0`
- Panel node: `dsi@14013000/panel1@0`
- Compatible: `wb,drm_lcm`
- Bound driver: `panel_wb_common_lcm`
- LCM name: `n813z_ft8201p_hjr085005d_a0_wuxga_90hz`

The runtime LCM configuration name is:
n813z_ft8201p_hjr085005d_a0_wuxga_90hz

This identifies the configured runtime panel profile, but does not independently establish the physical panel manufacturer.

### Panel driver module location (confirmed)

- Module: `panel_wb_common_lcm.ko`
- Runtime driver: `panel_wb_common_lcm`
- Storage location: `vendor_boot` ramdisk
- Extracted path: `ramdisk00/lib/modules/`
- Not present in the standard mounted vendor, vendor_dlkm, system_dlkm, or ODM module directories.

Dependencies from modules.dep:
- cust_panel.ko
- hwinfo.ko
- cust_gpios.ko
- middle_misc.ko
- charger_class.ko
- mtk_panel_ext.ko
- csci_driver.ko

## 5. Touchscreen

Confirmed through Android input diagnostics:

- Driver/device name: `fts_ts`
- Controller family: Focaltech FTS
- Connection: SPI1
- Device node: `/dev/input/event2`
- Maximum touch points: 10
- Coordinate range: 1200×1920

This agrees with the Focaltech node in the extracted DTBO.

## 6. Partition layout

The device uses dynamic partitions inside `super`.

Identified logical partitions include:
- `system`
- `vendor`
- `system_dlkm`
- `vendor_dlkm`
- `odm_dlkm`
- `system_ext`

The current active slot is `_a`. Many B-side logical partition entries are empty placeholders.

Important persistent and calibration partitions include:
- `nvram`
- `nvdata`
- `nvcfg`
- `persist`
- `protect1`
- `protect2`

These must be preserved.

## 7. Extracted working files

Local investigation directory:

`~/Documents/t813/`

Important files:
- `t813_dump/boot_a.img`
- `t813_dump/init_boot_a.img`
- `t813_dump/vendor_boot_a.img`
- `t813_dump/dtbo_a.img`
- `extracted_dtb/t813.dts`
- `extracted_dtb/dtbo/overlay.dts`

Tools:
- AOSP `unpack_bootimg.py`
- AOSP `mkdtboimg.py`
- `dtc`

## 8. HALs

### Graphics HAL
- VINTF declares `android.hardware.graphics.composer3` AIDL HAL version 3, instance `IComposer/default`.
- Vendor provides `hwcomposer.mt6877.so` and `mapper.mediatek.so`.
- VINTF also contains `manifest_allocator.xml` and `mapper.mediatek.xml`; 
- VINTF declares `android.hardware.graphics.allocator` AIDL HAL version 2, instance `IAllocator/default`.
- VINTF declares native `mapper` HAL version 5.0, instance `mediatek`.
- VINTF declares AIDL `android.hardware.memtrack`, instance `IMemtrack/default`.

### Media HAL
- VINTF declares AIDL `android.hardware.media.c2` version 1, instance `IComponentStore/default`.

### Audio HAL
- VINTF declares `android.hardware.audio.core` AIDL HAL version 2.
- Declared module instances: `default`, `r_submix`, `bluetooth`.
- Declared configuration instance: `IConfig/default`.
- VINTF declares `android.hardware.audio.effect` AIDL HAL version 2, instance `IFactory/default`.

### Bluetooth HAL
- VINTF declares `android.hardware.bluetooth.audio` AIDL HAL version 4, instance `IBluetoothAudioProviderFactory/default`.
- VINTF declares AIDL `android.hardware.bluetooth`, instance `IBluetoothHci/default`.

### Wi-Fi HAL
- VINTF declares `android.hardware.wifi` AIDL HAL version 2, instance `IWifi/default`.
- VINTF declares `android.hardware.wifi.supplicant` AIDL version 3, instance `ISupplicant/default`.
- VINTF declares `android.hardware.wifi.hostapd` AIDL version 2, instance `IHostapd/default`.

===================================================================




## Future work

### Immediate investigation
- [ ] Finish mapping display GPIOs and backlight.
- [x] Inspect core Graphics, Media, Audio, Bluetooth and Wi-Fi HAL declarations.
- [ ] Map remaining vendor HALs and proprietary libraries.
- [ ] Extract and document vendor init configuration.

### Kernel
- [ ] Investigate kernel source availability.
- [ ] Identify kernel configuration.
- [ ] Resolve kernel/module vermagic discrepancy.
- [ ] Determine whether existing kernel sources can support the port.

### LineageOS bring-up
- [ ] Create device tree.
- [ ] Prepare vendor tree and proprietary blobs.
- [ ] Prepare fstab and init configuration.
- [ ] Investigate SELinux requirements.
- [ ] Build and validate a native LineageOS image.

## Safety rules

- Preserve original stock firmware images.
- Do not overwrite calibration or modem-related partitions.
- Do not experiment with preloader or bootloader partitions.
- Do not assume another MT6877 device's DTB or firmware is compatible.
- Keep original partition dumps backed up separately.

## Current assessment

The platform, boot architecture, major partition layout, GPU, touchscreen and much of the display hardware have been identified.

The project is still in the research and preparation phase. No native LineageOS device tree or complete build has been produced yet.