# Step-by-Step Guide: Positivo T810 (Unisoc T310 / UMS312) GSI + Root

> **Target:** `Positivo T810` (`T810.OPEN.URBR2023101701`, slot `_b`)  
> **Result:** **LineageOS 19.1 (`a64_bvS`, Android 12.1)** with native **`phh-su` root** (`uid=0(root)`) and `armeabi-v7a` FOSS apps.

---

## 0. Prerequisites (Windows Host)

- **Modern Google Android SDK Platform-Tools** (`adb.exe`, `fastboot.exe`, `mke2fs.exe`, `make_f2fs.exe`).
  - Do **not** use legacy `fastboot.exe` (e.g., 2018 builds); it lacks `reboot fastboot`, `create-logical-partition`, `delete-logical-partition`, and `format:f2fs`.
- **Unisoc SCI/Download USB Driver** (`SPRD U2S Diag`) + **Google USB Driver** (`Android Bootloader Interface` / `ADB`).
- **`spd_dump`** + official signed loaders (`fdl1-sign.bin` @ `0x5500`, `fdl2-sign.bin` @ `0x9efffe00`) extracted from the official Positivo `.pac` firmware.
- **Python 3** + AOSP **`avbtool.py`** + **7-Zip** (`7z`).

---

## 1. Full Stock Backup (Mandatory)

Dump all stock partitions via `spd_dump` before writing anything, especially:
`boot_a/b`, `vbmeta_a/b`, `super`, `misc`, `miscdata`, `prodnv`, `persist`, `l_fixnv1_*`, `l_fixnv2_*`, `l_runtimenv*`.

> [!CAUTION]
> Never erase or overwrite `prodnv`, `miscdata`, `persist`, `l_fixnv*`, or `l_runtimenv*` without a verified SHA-256 backup.

---

## 2. Unlock the Bootloader (Official Unisoc Protocol)

1. In stock Android, enable **Developer Options → OEM Unlocking** and **USB Debugging**.
2. Reboot to bootloader (`adb reboot bootloader`).
3. Request identifier token:
   ```powershell
   fastboot oem get_identifier_token
   ```
4. Sign the token with the Unisoc RSA-4096 unlock key (or standard Unisoc unlock tool) to produce `signature.bin`, then run:
   ```powershell
   fastboot stage signature.bin
   fastboot flashing unlock_bootloader
   ```
5. Press **Volume -** on the tablet when prompted.
6. Expect `Info:Unlock bootloader success!` (boot screen will now show `INFO: LOCK FLAG IS : UNLOCK!!!` / `WARNING: LOCK FLAG IS : UNLOCK, SKIP VERIFY!!!`).

---

## 3. Prepare Images on PC

### 3.1 GSI Image (`a64_bvS`)
Download **AndyYan LineageOS 19.1** (`lineage-19.1-*-UNOFFICIAL-a64_bvS.img.gz`) and extract the `.img`.
- `a64` = `arm32_binder64` (required by `armeabi-v7a` userspace + 64-bit binder).
- `bvS` = Vanilla (no GApps) + built-in `phh-su` root.

### 3.2 `vbmeta` with `Flags: 3` & All 13 Descriptors Preserved
Generate a signed `vbmeta` image preserving every chain partition descriptor from stock `vbmeta_b`:

```powershell
python avbtool.py make_vbmeta_image `
  --include_descriptors_from_image stock_vbmeta_b.img `
  --flags 3 `
  --algorithm SHA256_RSA4096 `
  --key aosp_testkey_rsa4096.pem `
  --padding_size 4096 `
  --output vbmeta_flags3_20k.img
```

### 3.3 Empty 8 MB `ext4` Image for `product_b`
`fstab` requires `product` at first-stage mount. Create an 8 MB empty `ext4` image:

```powershell
mke2fs.exe -t ext4 -L product -b 4096 -O '^64bit,^metadata_csum' empty_product_8m.img 2048
```

### 3.4 `misc_bcb_1k.bin` (Optional Fallback to Enter `fastbootd` Without ADB)
If you cannot enable USB Debugging in Android, create a 1024-byte BCB with `boot-recovery\n--fastboot` at offset `0x0` (`command="boot-recovery"`, `recovery="recovery\n--fastboot\n"`) and write it via `spd_dump`:
`wof misc 0 misc_bcb_1k.bin reset`.

---

## 4. Enter `fastbootd` (`is-userspace: yes`)

Use any of:
- From Android: `adb reboot fastboot`
- From U-Boot bootloader: `fastboot reboot fastboot`
- From BootROM (`spd_dump`): `wof misc 0 misc_bcb_1k.bin reset`

Verify you are in userspace `fastbootd`:
```powershell
fastboot getvar is-userspace
# Must output: is-userspace: yes
```

---

## 5. Flash Sequence in `fastbootd`

> [!WARNING]
> Only flash `vbmeta_flags3_20k.img` while `is-userspace: yes`. Never flash a custom `vbmeta` via `spd_dump` (`fdl2`) or U-Boot `fastboot` (`is-userspace: no`).

```powershell
# 1. Flash vbmeta_b (Flags: 3, 13 descriptors preserved)
fastboot flash vbmeta_b vbmeta_flags3_20k.img

# 2. Delete product_b to free space inside dynamic partition 'super'
fastboot delete-logical-partition product_b

# 3. Flash LineageOS 19.1 a64_bvS to system_b
fastboot flash system_b lineage-19.1-20250606-UNOFFICIAL-a64_bvS.img

# 4. Recreate product_b (8 MB) and flash empty ext4
fastboot create-logical-partition product_b 8388608
fastboot flash product_b empty_product_8m.img

# 5. Keep / restore stock boot_b (ensures clean FBE + kernel 4.14.193)
fastboot flash boot_b stock_boot_b.img

# 6. Format userdata as F2FS and erase metadata
fastboot format:f2fs userdata
fastboot erase metadata

# 7. Reboot to Android
fastboot reboot
```

First boot takes ~30–60 seconds.

---

## 6. Post-Boot Verification & Root Check

Complete initial setup, enable **Developer Options → USB Debugging** (and **Rooted debugging** if desired), then run:

```powershell
adb shell "getprop ro.lineage.version; getprop ro.product.cpu.abi; getprop gsm.version.baseband; which su; su -c id"
```

Expected output:
- `ro.lineage.version`: `19.1-20250606-UNOFFICIAL-a64_bvS`
- `ro.product.cpu.abi`: `armeabi-v7a`
- `gsm.version.baseband`: `4G_MODEM_20A_W21.15.3|sharkl5_5m_modem`
- `/system/xbin/su` → `uid=0(root) gid=0(root) ... context=u:r:phhsu_daemon:s0`

### Recommended `armeabi-v7a` / Universal FOSS Apps
Because the T810 has 2 GB RAM and `armeabi-v7a` userspace, always pick **`armeabi-v7a` / `arm32v7`** or **`universal`** APKs (never `arm64-only`):
- **Stores:** F-Droid, Aurora Store (anonymous Google Play downloads)
- **YouTube (no Google account needed):** NewPipe, LibreTube
- **System-wide Adblock (via root `su`):** AdAway (enable *Root-based ad blocking*)
- **Browser:** Cromite (`arm_ChromePublic.apk`)
- **Keyboard:** HeliBoard
- **File Manager / LAN Transfer:** Material Files, LocalSend (`android-arm32v7.apk`)
- **Media:** Official VideoLAN `VLC-Android-*-armeabi-v7a.apk`
