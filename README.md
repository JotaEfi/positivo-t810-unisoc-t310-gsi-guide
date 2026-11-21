# Positivo T810 (Unisoc T310 / UMS312) — LineageOS 19.1 GSI + Root Guide

[![Device](https://img.shields.io/badge/Device-Positivo%20T810-blue)](#device-profile--search-keywords)
[![SoC](https://img.shields.io/badge/SoC-Unisoc%20T310%20(ums312)-orange)](#device-profile--search-keywords)
[![GSI](https://img.shields.io/badge/GSI-LineageOS%2019.1%20(a64__bvS)-green)](#tldr--verified-working-recipe)
[![Root](https://img.shields.io/badge/Root-phh--su%20(native)-brightgreen)](#tldr--verified-working-recipe)

A clean, documentation-only guide for unlocking the bootloader, flashing a **Project Treble GSI (LineageOS 19.1 `a64_bvS`)**, and getting **working root (`su`)** on the **Positivo T810** tablet (**Unisoc Tiger T310 / `ums312`**) and similar **Unisoc Android 11 (`VNDK 30`, kernel `4.14`) `arm32_binder64` Dynamic-Partition A/B** devices.

Written for both humans and **AI coding/debugging agents** searching for verified Unisoc `ums312` GSI bring-up, `vbmeta` / `DHTB` pitfalls, `fastbootd` dynamic-partition resizing, and `init_user0_failed` fixes.

---

## Device Profile & Search Keywords

| Property | Verified Value |
| :--- | :--- |
| **Brand / Model** | `positivo` / `T810` (`T810.OPEN.URBR2023101701`, `RP1A.201005.001`) |
| **SoC / Board** | Unisoc Tiger T310 (`ums312`, `ums312_2h10_go_32b`, `sharkl5`) |
| **CPU / RAM** | 1x Cortex-A75 + 3x Cortex-A55, 2 GB RAM (`low_ram=true`) |
| **Kernel** | Linux `4.14.193` (`armv7l`, `CONFIG_ARM=y`, `CONFIG_ANDROID_BINDERFS=y`, 64-bit binder) |
| **Userspace / GSI Arch** | `armeabi-v7a` (`zygote32`), 64-bit binder → **`a64`** (`arm32_binder64`, A/B) |
| **Stock OS / VNDK** | Android 11, VNDK `30`, VINTF target-level `5` |
| **Partitioning** | A/B (`slot_suffix=_b`), Dynamic Partitions (`super` ~4.3 GB), Virtual A/B |
| **Boot / Recovery** | `boot` header v2 (no standalone `recovery`; `fastbootd` inside `boot` ramdisk) |
| **Encryption** | File-Based Encryption (`ro.crypto.state=encrypted`, `ro.crypto.type=file`, F2FS `userdata`) |
| **Bootloader Unlock** | Official Unisoc RSA token (`oem get_identifier_token` → `flashing unlock_bootloader`) |

---

## Repository Contents

- [`GUIDE.md`](GUIDE.md) — Step-by-step installation guide (official unlock → `vbmeta` → `fastbootd` → GSI + `product` fix → root verification & FOSS apps).
- [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md) — Catalog of real errors encountered on Unisoc `ums312` / T810 (exact error strings, U-Boot/FDL2 root causes, and verified fixes).
- [`HARDWARE_SPECS.md`](HARDWARE_SPECS.md) — Low-level partition, `fstab`, AVB/DHTB (`0xffe00`), and bootloader facts.

---

## TL;DR — Verified Working Recipe

1. **Do NOT use `LineageOS 20` (Android 13) or `Magisk v31.1` on `boot_b`:**
   - `LineageOS 20` (`a64_bvN`) bootloops on this `4.14.193` / Android 11 vendor.
   - `Magisk v31.1` (`KEEPVERITY=false`, `PREINITDEVICE=cache`) causes `Android Recovery: [init_user0_failed]` (FBE failure) even on freshly formatted `userdata`.
2. **Use `LineageOS 19.1` `a64_bvS` (Android 12.1, AndyYan) + untouched stock `boot_b`:**
   - `bvS` includes native **`phh-su` (`/system/xbin/su`)** inside `/system`, keeping stock `boot_b` intact and stable.
3. **Preserve all 13 AVB chain descriptors in `vbmeta_b` (`Flags: 3`):**
   - Do **not** flash an empty 256-byte `vbmeta_disabled.img` with 0 descriptors (breaks U-Boot `vboot` even for `fastbootd`).
   - Build `vbmeta_flags3_20k.img` with `--include_descriptors_from_image <stock_vbmeta_b> --flags 3`.
4. **Flash `vbmeta_b` ONLY in `fastbootd` (`is-userspace: yes`), NEVER via `spd_dump` (`fdl2`) or U-Boot `fastboot`:**
   - Both `fdl2` and U-Boot `FASTBOOT_MODE` check for Unisoc's `DHTB` header at offset `0xffe00` on `vbmeta` and enter an infinite loop (`while(1)`) if missing.
5. **Delete `product_b` to fit `system_b`, then recreate an empty 8 MB `ext4` `product_b`:**
   - `first_stage_ramdisk/fstab.ums312_2h10_go_32b` requires `product` (`wait,logical,first_stage_mount,slotselect`).
6. **Wipe with `fastboot format:f2fs userdata` + `fastboot erase metadata`** (do not rely on `fastboot -w`, which aborts on `erase cache`).

---

## Suggested GitHub Topics (for Discoverability)

If you publish this repository on GitHub, add these repository topics so AI agents and developers can find it easily:

`positivo-t810` `unisoc` `t310` `ums312` `treble-gsi` `lineageos-19` `a64-bvs` `fastbootd` `spd-dump` `vbmeta` `avb` `android-gsi` `phh-su` `root` `unbrick`
