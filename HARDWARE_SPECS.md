# Hardware, Partition & Bootloader Technical Reference (Positivo T810)

Collected from live `adb`, `fastbootd`, and `uboot_b` / `fdl2-sign.bin` inspection on `positivo/T810` (`T810.OPEN.URBR2023101701`).

---

## Hardware & Kernel Facts

- **Model / Build:** `positivo/T810/T810:11/RP1A.201005.001/URBR2023101701:user/release-keys`
- **SoC:** Unisoc Tiger T310 (`ums312`), board `ums312_2h10_go_32b`
- **Kernel:** `4.14.193` (`armv7l`, `CONFIG_ARM=y`)
  - `CONFIG_ANDROID_BINDERFS=y`, `# CONFIG_ANDROID_BINDER_IPC_32BIT is not set` (64-bit binder → requires `a64` GSI, **not** `arm`)
  - `CONFIG_PSI=y`, `CONFIG_MEMCG=y`, `CONFIG_CGROUP_FREEZER=y`, `CONFIG_BPF_JIT=y`
  - `CONFIG_F2FS_FS=y`, `CONFIG_PSTORE=y`, `CONFIG_PSTORE_RAM=y`
- **ABIs:** `ro.product.cpu.abi=armeabi-v7a`, `ro.zygote=zygote32`, `ro.vndk.version=30`

---

## Partition & Dynamic Layout (`super`)

- **A/B Slots:** Active slot `_b` (`slot-count: 2`)
- **Boot Image:** `boot_b` (`67,108,864` bytes / 64 MB), header v2 (`page_size=2048`), gzipped ramdisk + DTB; `vendor_boot_b` is all zeros on stock.
- **Key `first_stage_ramdisk/fstab.ums312_2h10_go_32b` entries:**
  ```text
  system     /system     ext4 ro,barrier=1 wait,avb=vbmeta_system,logical,first_stage_mount,...,slotselect
  system_ext /system_ext ext4 ro,barrier=1 wait,logical,first_stage_mount,avb=vbmeta_system_ext,slotselect
  vendor     /vendor     ext4 ro,barrier=1 wait,avb=vbmeta_vendor,logical,first_stage_mount,slotselect
  product    /product    ext4 ro,barrier=1 wait,avb=vbmeta_product,logical,first_stage_mount,slotselect
  /dev/block/by-name/metadata /metadata ext4 nodev,noatime,nosuid,errors=panic wait,formattable,first_stage_mount,check
  ```
- **Dynamic Group (`group_unisoc_b`):** max `4,294,967,296` bytes (~4.0 GiB).
  - Stock `product_b`: ~1.50 GB (deleted and recreated as 8 MB `empty_product_8m.img` to fit GSI `system_b`).
  - `vendor_b`: ~277 MB (`0x10889000`).
  - `system_ext_b`: ~261 MB (`0xF969000`).

---

## Bootloader (`uboot_b`) & `fdl2` Security Behavior

- **BootROM / Download USB ID:** `VID_1782&PID_4D00` (`SPRD U2S Diag`)
- **U-Boot Fastboot USB ID:** `VID_18D1&PID_4EE0` (`is-userspace: no`)
- **`fastbootd` USB ID:** `VID_1782&PID_4EE0` (`is-userspace: yes`)
- **Stock `vbmeta_b` (`1,048,576` bytes):**
  - Signed with AOSP `testkey` (`Public key sha1: 2597c218aae470a130f61162feaae70afd97f011`, `SHA256_RSA4096`).
  - Contains **13 Chain Partition descriptors**: `boot`, `dtbo`, `socko`, `odmko`, `vbmeta_system`, `vbmeta_system_ext`, `vbmeta_vendor`, `vbmeta_product`, `l_modem`, `l_ldsp`, `l_gdsp`, `l_agdsp`, `pm_sys`.
  - Contains Unisoc `DHTB` dual-backup hash header at offset **`0xffe00`**.
- **Why `vbmeta_b` can only be customized in `fastbootd`:**
  - `fdl2-sign.bin` (`sprd_dl_vboot_verify`) and `uboot_b` (`sprd_fb_secure_vboot_verify`) both validate `DHTB @ 0xffe00` on `vbmeta` and execute an infinite loop (`b .`) on mismatch.
  - `fastbootd` runs in Linux userspace and writes `vbmeta_b` directly to the block device without triggering `uboot_b`'s download check.
