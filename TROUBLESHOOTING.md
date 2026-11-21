# Troubleshooting & Known Errors (Positivo T810 / Unisoc UMS312)

Every error below was observed on real hardware (`positivo/T810`, `ums312`) along with its verified root cause and fix.

---

## 1. `spd_dump` or U-Boot `fastboot` hangs forever when flashing `vbmeta_b`
- **Symptoms:**
  - `spd_dump` (`write_part vbmeta_b` or `wof vbmeta_b`) reaches 100% and hangs with `usb_send failed (0 / 8)` / `ErrorCode: 0x00000079` and black screen.
  - U-Boot `FASTBOOT_MODE` (`is-userspace: no`) hangs on `flash:vbmeta_b`.
- **Root Cause (verified in `fdl2-sign.bin` and `uboot_b`):**
  - Both `fdl2` (`sprd_dl_vboot_verify`) and U-Boot (`sprd_fb_secure_vboot_verify`) check offset `0xffe00` of `vbmeta` for Unisoc's `DHTB` dual-backup hash header and OEM RSA signature. If missing/invalid, they print `error:check dual backup hash failed for vbmeta.img.` and enter an infinite loop (`b .`).
- **Fix:**
  1. Unplug USB, hold **Power for 15s** to exit the wedged session.
  2. Never flash a custom `vbmeta` via `spd_dump` or U-Boot `fastboot`.
  3. If stuck in a bootloop, flash **stock `vbmeta_b`** via `spd_dump` (it has the genuine `DHTB @ 0xffe00`), boot into **`fastbootd` (`is-userspace: yes`)**, and flash `vbmeta_flags3_20k.img` there.

---

## 2. `fastboot flash` in U-Boot (`is-userspace: no`) fails with `winerr=121`
- **Symptom:** Modern Google `fastboot.exe` hangs or returns semaphore timeout (`winerr=121`) when flashing in U-Boot `FASTBOOT_MODE`.
- **Root Cause:** Modern `fastboot` sends `fetch:super:...` / dynamic partition queries that Unisoc's U-Boot `fastboot` implementation does not handle.
- **Fix:** Always run `fastboot reboot fastboot` to enter **`fastbootd` (`is-userspace: yes`)** first.

---

## 3. Tapping Build Number 7x prompts `"Digite a senha: (confirmar)"` (even with no PIN/password set)
- **Symptom:** After bootloader unlock wipe, stock Android 11 asks for a password when trying to enable Developer Options.
- **Fix:** Bypass ADB completely by entering **`fastbootd`** via **`spd_dump` + `misc_bcb_1k.bin`** (`wof misc 0 misc_bcb_1k.bin reset`).

---

## 4. `fastboot flash system_b` fails with `Not enough space to resize partition`
- **Root Cause:** Stock `product_b` consumes ~1.5 GB inside `super` (`group_unisoc_b` max `4,294,967,296` bytes).
- **Fix:** Delete `product_b`, flash `system_b`, and **recreate `product_b` as 8 MB**:
  ```powershell
  fastboot delete-logical-partition product_b
  fastboot flash system_b system.img
  fastboot create-logical-partition product_b 8388608
  fastboot flash product_b empty_product_8m.img
  ```

---

## 5. Bootloop after deleting `product_b` (or using 0-descriptor `vbmeta_disabled.img`)
- **Causes:**
  1. `first_stage_ramdisk/fstab.ums312_2h10_go_32b` mounts `product` with `wait,logical,first_stage_mount,slotselect`. Deleting `product_b` without recreating it prevents Android from booting.
  2. A 256-byte `vbmeta_disabled.img` with `0` descriptors breaks `uboot_b`'s `vboot` verification flow (even `uboot_log` stops incrementing and `fastbootd` fails to boot).
- **Fix:** Always recreate `product_b` (`8 MB` ext4) and use `vbmeta_flags3_20k.img` (`--include_descriptors_from_image stock_vbmeta_b --flags 3`).

---

## 6. `fastboot -w` fails with `Erasing 'cache' FAILED`
- **Root Cause:** In `fastbootd`, `cache` is exposed as `raw` and `erase cache` returns an error.
- **Fix:** Replace `fastboot -w` with:
  ```powershell
  fastboot format:f2fs userdata
  fastboot erase metadata
  ```

---

## 7. Bootloop on `LineageOS 20` (Android 13 `a64_bvN`)
- **Symptom:** With stock `boot_b`, `vbmeta_flags3_20k.img`, `empty_product_8m.img`, and formatted `userdata`, `LineageOS 20` (`a64_bvN`) still bootloops on the Positivo splash screen.
- **Cause:** Android 13 GSI incompatibility with `4.14.193` / Positivo Android 11 `system_ext` & vendor.
- **Fix:** Use **LineageOS 19.1 (`a64_bvS`, Android 12.1)** instead — boots cleanly in ~30 seconds.

---

## 8. `Android Recovery: Can't load Android system` with `[init_user0_failed]`
- **Symptom:** After flashing a `Magisk v31.1`-patched `boot_b` (or switching between `bvN` and `bvS`), Android drops to Recovery with:
  `*******NEED TO WIPE USERDATA*******`
  `REASON IS: [init_user0_failed]`
- **Root Cause:** `userdata` uses File-Based Encryption (`ro.crypto.type=file`). Changing `boot` (`Magisk v31.1` with `KEEPVERITY=false` / `PREINITDEVICE=cache`) or switching SELinux/user context (`bvN` ↔ `bvS`) breaks `/data/user/0` initialization.
  - Note: `Magisk v31.1` failed with `[init_user0_failed]` / bootloop **even after** `fastboot format:f2fs userdata`.
- **Fix:**
  - Keep **stock `boot_b`** and use **`LineageOS 19.1 a64_bvS`** (native `phh-su` root).
  - If switching from `bvN` to `bvS` and `[init_user0_failed]` appears, select **`Factory data reset`** on that Recovery screen once (with `bvS` and stock `boot_b` already flashed) so `/data` is initialized cleanly by `bvS`.

---

## 9. Tablet stuck in bootloop or Recovery won't catch BootROM (`COM42`)
- **Symptom:** Holding `Volume -` + plugging USB misses the BootROM window during active bootloops, or `spd_dump` exits on a transient 0.5s `CreateFile` error.
- **Fix:**
  1. Run `spd_dump --wait 300` inside a PowerShell retry loop so transient pulses don't kill the listener.
  2. Unplug USB, hold **Volume - + Power** together for ~12–15s until the screen goes black.
  3. The **exact millisecond** the screen goes black, release **Power only** (keep **Volume -** held down) and plug USB in while the screen is still black, holding **Volume -** for ~4 more seconds.

---

## 10. `adb install` fails with `[INSTALL_FAILED_NO_MATCHING_ABIS]` (e.g. F-Droid VLC)
- **Root Cause:** The tablet runs **32-bit `armeabi-v7a`** userspace (`ro.product.cpu.abilist64` is empty). Some APKs on F-Droid only ship `arm64-v8a` native libraries.
- **Fix:** Download the `armeabi-v7a` / `arm32v7` or `universal` APK build (e.g., official VideoLAN `VLC-Android-*-armeabi-v7a.apk`).

---

## 11. `pm grant com.google.android.gms android.permission.FAKE_PACKAGE_SIGNATURE` fails (`Unknown permission`)
- **Root Cause:** AndyYan `lineage-19.1` vanilla (`bvN`/`bvS`) does not include built-in signature spoofing.
- **Workaround:** Use **NewPipe** / **LibreTube** for YouTube and **Aurora Store** (anonymous mode) for Play Store apps; neither requires signature spoofing.
