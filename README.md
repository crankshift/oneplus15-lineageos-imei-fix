# Fix: Generic IMEI on LineageOS 23 for OnePlus 15 (Infiniti)

## Problem

The unofficial LineageOS 23.2 ROM reports a **generic Qualcomm IMEI** instead of the device's real IMEI. Some carriers reject devices with a generic IMEI, breaking cellular connectivity.

## Root Cause: Oplus PCBA Verification Failure

The IMEI issue is caused by **Oplus PCBA verification failing on custom ROMs** — not by the kernel source. The ROM developer [confirmed](https://xdaforums.com/t/rom-unofficial-lineageos-23-for-oneplus-15.4785437/) this:

> "The IMEI issue isn't related to Qualcomm source releases. It's actually due to Oplus PCBA verification failing on custom ROMs."

OnePlus 13 (SM8750) running official LineageOS 23 does **not** have this issue — telephony, VoLTE, and WiFi Calling all work. Something changed in the SM8850 verification flow.

### How PCBA Verification Works

The verification chain, reconstructed from binary analysis of `libqti-radio-service.so` (loaded by `subsys_daemon`):

```
subsys_daemon starts (triggered by ro.boot.hardware=qcom)
  → loads libqti-radio-service.so
  → ModemConfig::LoadPcba()
    → getPcbaFromReserve1()
       reads raw PCBA data from /dev/block/by-name/oplusreserve1
    → PcbaDeconfuse()
       deobfuscates the PCBA data
    → VerifyPcbaValid(reserve1_pcba_data*)
       RSA signature verification (VerifyDataRawByPublicKey)
    → if valid: extracts PCBA number, modem uses real IMEI from modemst1/modemst2
    → if invalid: "VerifyPcbaValid failed!" → modem falls back to generic Qualcomm IMEI
    → fallback: getPcbaFromNV6855() (reads PCBA from NV item 6855 if reserve1 fails)
```

The `subsys_daemon` service (`odm/bin/hw/subsys_daemon`) implements the `vendor.oplus.hardware.subsys_interface.subsys_radio` AIDL HAL. It runs as two instances:
- `qti-modem-daemon-0` (SIM slot 0) — starts on `property:ro.boot.hardware=qcom`
- `qti-modem-daemon-1` (SIM slot 1) — starts on DSDS config

Both load `libqti-radio-service.so` via `-l /odm/lib64/libqti-radio-service.so`.

### Key Strings from libqti-radio-service.so

```
"enter LoadPcba"
"Load Pcba from reserve1"
"getPcbaFromReserve1 failed, errno: %d. Try to get from NV6855"
"PcbaDeconfuse args error"
"VerifyPcbaValid failed!"
"pcba:  = %s"
"pcba_len = %d"
"pcba data is not same"                    ← reserve1 vs NV6855 mismatch
"ue_imeisv_svn wrong"
"receive cmd of getPcbaNumber"
```

Paths accessed by the library:
- `/dev/block/by-name/oplusreserve1` (primary PCBA data source)
- `/dev/block/bootdevice/by-name/oplusreserve1` (alternative path)
- `/dev/block/by-name/opporeserve1` (legacy OPPO path)
- `/mnt/vendor/oplusreserve/radio/exp_operator_switch.config`
- `/mnt/vendor/oplusreserve/radio/rule_list.config`

### SM8850 vs SM8750: What Changed

Both OnePlus 13 (SM8750) and OnePlus 15 (SM8850) use the same PCBA verification architecture (`subsys_daemon` + `libqti-radio-service.so`). The differences that may explain why OP15 breaks:

| Area | OnePlus 13 (SM8750, working) | OnePlus 15 (SM8850, broken) |
|------|-----|-----|
| `ro.vendor.oplus.radio.serialno` | Not set | Set from `ro.boot.serialno` in `init.oplus.rc:84` |
| `ro.vendor.oplus.radio.carrier_detect_mode` | Not present | `1` (in `odm.prop`) |
| `ro.telephony.default_network` | `33,33` | `26,26` |
| `sys.oplus_ftm_mode` | Not present | `997` |
| `persist.vendor.radio.mdccbsize` | `164` | Not present |
| `persist.vendor.radio.poweron_opt` | `1` (in `system_ext.prop`) | Not present |
| ODM VINTF (Oplus radio HALs) | Identical | Identical |
| `init.oppo.reserve.rc` | Present | Present |
| `subsys_daemon` + deps | Present | Present |
| `soccp_firmware` fstab mount | Not present | Present (new firmware partition) |
| RF license files | SM8750-specific | SM8850-specific (more licenses) |

The new `ro.vendor.oplus.radio.serialno` property in SM8850 is the most significant change — the modem verification flow likely now requires the serial number to validate PCBA data, and if this chain breaks for any reason, the modem falls back to generic IMEI.

### What Needs Investigation (On-Device)

The fix requires answering these questions on a running LineageOS device:

**1. Is `subsys_daemon` running?**
```bash
adb shell ps -A | grep -E "subsys_daemon|qti-modem-daemon"
adb shell getprop init.svc.qti-modem-daemon-0
adb shell getprop init.svc.qti-modem-daemon-1
```
If not running or crashed, check logcat for crash reason — likely a missing dependency or SELinux denial.

**2. Can it read oplusreserve1?**
```bash
adb shell ls -la /dev/block/by-name/oplusreserve1
adb shell ls -la /dev/block/bootdevice/by-name/oplusreserve1
```
The SELinux policy in `hardware/oplus` labels these as `vendor_reserve_partition` and grants `subsystem_daemon` read access. Check for denials:
```bash
adb shell dmesg | grep "avc: denied" | grep -iE "subsystem_daemon\|reserve\|oplusreserve"
```

**3. Is oplusreserve2 mounted with the radio directory?**
```bash
adb shell mount | grep oplusreserve
adb shell ls -la /mnt/vendor/oplusreserve/radio/
```
The `init.oppo.reserve.rc` creates `/mnt/vendor/oplusreserve/radio` (radio:system 0771) during `post-fs-data`. If this directory doesn't exist, the radio config files can't be read.

**4. Are the Oplus radio HALs registered?**
```bash
adb shell lshal | grep -i oplus
adb shell service list | grep -i oplus
```
Expected HALs (from `network_manifest_odm.xml`):
- `vendor.oplus.hardware.appradioaidl` (OplusAppRadio0/1)
- `vendor.oplus.hardware.ims` (OplusImsRadio0/1)
- `vendor.oplus.hardware.radio` (OplusRadio0/1)
- `vendor.oplus.hardware.subsys_interface.subsys_radio` (slot1/slot2)

**5. Is the serial number propagated?**
```bash
adb shell getprop ro.boot.serialno
adb shell getprop ro.vendor.oplus.radio.serialno
```

**6. Are RF licenses installed?**
```bash
adb shell ls -la /mnt/vendor/persist/data/pfm/licenses/
```
`init.network.rc` copies RF tuner and TDD-bypass licenses from `/odm/etc/` and `/vendor/etc/` to `/mnt/vendor/persist/data/pfm/licenses/` on boot. If these aren't present, the modem may not initialize properly.

**7. Is oplus_mdmfeature loaded?**
```bash
adb shell lsmod | grep mdmfeature
adb shell cat /proc/oplusManifest/network_manifest
```

**8. The actual IMEI test:**
```bash
adb shell service call iphonesubinfo 1
```

### Likely Fix Approaches (Ranked)

**1. Ensure `subsys_daemon` runs and completes PCBA verification** — the most likely issue is that this service either doesn't start, crashes, or gets blocked by SELinux on LineageOS. The SELinux policy in `hardware/oplus/sepolicy/qti/vendor/subsystem_daemon.te` grants the necessary permissions (`vendor_reserve_partition:blk_file r_file_perms`, `qipcrtr_socket create_socket_perms_no_ioctl`), but the file_contexts must match the actual device paths for SM8850.

**2. Verify `oplusreserve1` is intact** — the PCBA data lives on the `oplusreserve1` raw block partition. This is NOT wiped during normal ROM flashing (it's a separate partition, not part of super/userdata). But if someone ran a full erase or used MSM tool, the PCBA data may be gone.

**3. Check RF license installation** — `init.network.rc` copies multiple platform-specific license files (`*.pfm`) to persist. SM8850 has new license files not present on SM8750. If these fail to copy (SELinux, missing source files), the modem may not initialize.

**4. Property-level fix** — ensure `ro.vendor.oplus.radio.serialno` is correctly set from the bootloader. If `ro.boot.serialno` is empty or wrong on custom ROMs, inject it from another source.

**5. Stub the PCBA verification (last resort)** — if the RSA verification in `VerifyPcbaValid()` fails for reasons that can't be fixed at the ROM level (e.g., changed public key in newer firmware), a stub in `libqti-radio-service.so` would be needed. This is extremely invasive and should be avoided.

### Key Files and Services

| File | Role |
|------|------|
| `odm/bin/hw/subsys_daemon` | Implements `ISubsysRadio` HAL, loads PCBA verification library |
| `odm/lib64/libqti-radio-service.so` | Contains `ModemConfig::LoadPcba()`, `VerifyPcbaValid()`, PCBA read/verify logic |
| `odm/lib64/libradio-service.so` | Radio service with `writeEncryptedSerialId`, `setImeiSvn`, `setMdmFeature` |
| `odm/lib64/libradioapis.so` | QMI vendor service client: `radioVerifyIdl`, `radioSetMdmFeature` |
| `odm/bin/commcenterd` | Communication center daemon, implements `ICommCenter` HAL |
| `odm/etc/init/subsys_daemon.rc` | Service definitions for `qti-modem-daemon-0/1` |
| `odm/etc/init/init.oppo.reserve.rc` | Creates `/mnt/vendor/oplusreserve/radio` directory |
| `odm/etc/init/init.network.rc` | Copies RF license files to persist, modem diag logging |
| `hardware/oplus/sepolicy/qti/vendor/subsystem_daemon.te` | SELinux policy for subsys_daemon |
| `hardware/oplus/sepolicy/qti/vendor/file_contexts` | Labels oplusreserve partitions as `vendor_reserve_partition` |

---

## OSS Kernel Source (Fixes DT2W, Not IMEI)

Switching to an OSS source-built kernel fixes **DT2W (Double Tap to Wake)** and other kernel-level features, but does **not** fix the IMEI issue. The kernel patches below are still needed as a separate fix.

OnePlus has released GPL kernel sources for SM8850:

- https://github.com/OnePlusOSS/android_kernel_oneplus_sm8850 (branch: `oneplus/sm8850_b_16.0.0_oneplus_15`)
- https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8850
- https://github.com/OnePlusOSS/android_kernel_common_oneplus_sm8850

Building the kernel from source allows proper modem driver initialization under LineageOS. The device tree already has OSS kernel build infrastructure (`BoardConfigCommon.mk` has the full `ifneq ($(USE_PREBUILT_KERNEL), true)` block) — it just needs corrections and missing files.

## What Needs to Change

### 1. Flip the kernel switch (device_infiniti)

In `android_device_oneplus_infiniti/BoardConfig.mk`, change:

```diff
-USE_PREBUILT_KERNEL ?= true
+USE_PREBUILT_KERNEL ?= false
```

### 2. Fix platform naming (device_common)

The existing `BoardConfigCommon.mk` references "pineapple" (Snapdragon 8 Gen 3), but OnePlus 15 is "canoe" (SM8850 / Snapdragon 8 Elite). In `android_device_oneplus_sm8850-common/BoardConfigCommon.mk`:

```diff
 TARGET_KERNEL_CONFIG := \
     gki_defconfig \
-    vendor/pineapple_GKI.config \
-    vendor/oplus/pineapple_GKI.config
+    vendor/canoe_GKI.config \
+    vendor/oplus/canoe_GKI.config
```

And update ALL module list references from `pineapple` to `canoe`:

```diff
-BOARD_SYSTEM_KERNEL_MODULES_LOAD := $(strip $(shell cat $(TARGET_KERNEL_SOURCE)/modules.system_dlkm.list.msm.pineapple))
-BOARD_VENDOR_KERNEL_MODULES_BLOCKLIST_FILE := $(TARGET_KERNEL_SOURCE)/modules.vendor_blocklist.msm.pineapple
-BOARD_VENDOR_KERNEL_MODULES_LOAD := $(strip $(shell cat $(TARGET_KERNEL_SOURCE)/modules.vendor_dlkm.list.msm.pineapple ...))
+BOARD_SYSTEM_KERNEL_MODULES_LOAD := $(strip $(shell cat $(TARGET_KERNEL_SOURCE)/modules.system_dlkm.list.msm.canoe))
+BOARD_VENDOR_KERNEL_MODULES_BLOCKLIST_FILE := $(TARGET_KERNEL_SOURCE)/modules.vendor_blocklist.msm.canoe
+BOARD_VENDOR_KERNEL_MODULES_LOAD := $(strip $(shell cat $(TARGET_KERNEL_SOURCE)/modules.vendor_dlkm.list.msm.canoe ...))
```

Full diff in `patches/0002-device_common-fix-kernel-config-refs.patch`.

### 3. Create missing files in kernel source tree

The `BoardConfigCommon.mk` references module list files and config fragments that **don't exist** in the OnePlusOSS kernel source. These need to be added to `kernel/oneplus/sm8850/` (the `TARGET_KERNEL_SOURCE` path):

**Module list files** — the OSS kernel source only has a combined list (`modules-lists/modules.list.msm.canoe`) and a blocklist, but the LineageOS build system expects per-partition lists. These were derived by cross-referencing the combined list against the prebuilt kernel's partition-specific `modules.load` files:

| File | Modules | Source |
|------|---------|--------|
| `modules.system_dlkm.list.msm.canoe` | 82 | `android_device_oneplus_infiniti-kernel/modules/system_dlkm/modules.load` |
| `modules.vendor_dlkm.list.msm.canoe` | 435 | `android_device_oneplus_infiniti-kernel/modules/vendor_dlkm/modules.load` |
| `modules.vendor_dlkm.list.oplus.canoe` | 161 | OPlus modules filtered from vendor_dlkm `modules.load` |
| `modules.vendor_boot.list.msm.canoe` | 120 | `android_device_oneplus_infiniti-kernel/modules/vendor_ramdisk/modules.load` |
| `modules.vendor_boot.list.oplus.canoe` | 18 | `android_kernel_modules_and_devicetree_oneplus_sm8850/kernel_platform/oplus/config/modules.vendor_boot.list.oplus` |
| `modules.recovery.list.msm.canoe` | 316 | `android_device_oneplus_infiniti-kernel/modules/vendor_ramdisk/modules.load.recovery` |
| `modules.recovery.list.oplus.canoe` | 149 | OPlus modules filtered from recovery `modules.load.recovery` |

**Kernel config fragment** — the OSS kernel uses Bazel `.bzl` format, not traditional Kconfig `.config` files. `canoe_GKI.config` (468 options) was generated by converting the Python dict in `android_kernel_oneplus_sm8850/configs/canoe_perf.bzl` to standard `CONFIG_*=` format:
- `vendor/canoe_GKI.config` (468 CONFIG_ options)

**OPlus config fragment** — extract oplus-specific entries from the generated config:
- `vendor/oplus/canoe_GKI.config`

All these files are provided in `patches/kernel_source_files/`.

### 4. Kernel source tree paths

In the LineageOS build tree, repos must be at:
- `kernel/oneplus/sm8850` -> `android_kernel_oneplus_sm8850` (main kernel)
- `kernel/oneplus/sm8850-modules` -> `android_kernel_modules_and_devicetree_oneplus_sm8850` (external modules)

The `local_manifests` should map these accordingly.

## How to Apply

```bash
# In android_device_oneplus_infiniti:
git apply 0001-device_infiniti-switch-to-oss-kernel.patch

# In android_device_oneplus_sm8850-common:
git apply 0002-device_common-fix-kernel-config-refs.patch

# Copy module list files to kernel source root (kernel/oneplus/sm8850/):
cp patches/kernel_source_files/modules.*.canoe kernel/oneplus/sm8850/

# Copy kernel config fragment:
mkdir -p kernel/oneplus/sm8850/vendor/oplus
cp patches/kernel_source_files/canoe_GKI.config kernel/oneplus/sm8850/vendor/canoe_GKI.config
# For oplus config, extract oplus-specific entries from canoe_GKI.config:
grep "OPLUS\|oplus" patches/kernel_source_files/canoe_GKI.config > kernel/oneplus/sm8850/vendor/oplus/canoe_GKI.config
```

## Verification (After Build & Flash)

### IMEI / PCBA Verification
```bash
# Check real IMEI (should NOT be generic Qualcomm)
adb shell service call iphonesubinfo 1

# Check PCBA verification chain
adb shell getprop init.svc.qti-modem-daemon-0    # should be "running"
adb shell getprop ro.vendor.oplus.radio.serialno  # should be non-empty
adb shell ls /mnt/vendor/oplusreserve/radio/       # should exist
adb shell dmesg | grep "avc: denied" | grep -iE "subsystem_daemon|reserve|radio"
```

### Kernel / Modem
```bash
# Check baseband version (should be non-empty)
adb shell getprop gsm.version.baseband

# Check modem subsystem state (should be ONLINE)
adb shell cat /sys/bus/msm_subsys/devices/subsys*/name
adb shell cat /sys/bus/msm_subsys/devices/subsys*/state

# Check QMI services registered
adb shell qrtr-lookup

# Check kernel logs for clean modem boot
adb shell dmesg | grep -iE "pil|modem|mhi|qrtr"

# Check for SELinux denials (should be none for radio)
adb shell dmesg | grep "avc: denied" | grep -iE "rild|radio|modem|qmi"
```

## Repositories Used

**OSS Kernel Sources (OnePlusOSS):**
- [android_kernel_oneplus_sm8850](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8850) — branch `oneplus/sm8850_b_16.0.0_oneplus_15` — main kernel source, defconfigs, Bazel configs (`configs/canoe_perf.bzl`), combined module lists
- [android_kernel_modules_and_devicetree_oneplus_sm8850](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8850) — external kernel modules (Qualcomm + OPlus), device trees, OPlus module boot list
- [android_kernel_common_oneplus_sm8850](https://github.com/OnePlusOSS/android_kernel_common_oneplus_sm8850) — GKI common kernel, provides `gki_defconfig`

**Device Tree & Vendor (OnePlus-SM8850-Development):**
- [android_device_oneplus_sm8850-common](https://github.com/OnePlus-SM8850-Development/android_device_oneplus_sm8850-common) — branch `lineage-23.2` — shared BoardConfig, kernel build config, module lists
- [android_device_oneplus_infiniti](https://github.com/OnePlus-SM8850-Development/android_device_oneplus_infiniti) — branch `lineage-23.2` — device-specific BoardConfig with `USE_PREBUILT_KERNEL` toggle
- [android_device_oneplus_infiniti-kernel](https://github.com/OnePlus-SM8850-Development/android_device_oneplus_infiniti-kernel) — branch `lineage-23.2` — current prebuilt kernel (used as reference for partition-specific module lists)
- [proprietary_vendor_oneplus_infiniti](https://github.com/OnePlus-SM8850-Development/proprietary_vendor_oneplus_infiniti) — branch `lineage-23.2` — vendor blobs from `CPH2745_16.0.5.700(EX01)`

## OnePlusOSS Kernel vs CLO (CodeLinaro) Kernel

This fix uses **OnePlus's GPL kernel dump** — not Qualcomm's official **CLO (CodeLinaro) platform release** for SM8850/canoe. These are fundamentally different kernel sources with different stability expectations.

### What we're using: OnePlusOSS GPL dump

OnePlus publishes kernel source to comply with GPL. These sources are:

- **Built for OxygenOS**, not AOSP — init sequences, property namespaces, and SELinux contexts assume OxygenOS userspace
- **Bazel/Kleaf build system** — uses `.bzl` config files instead of standard Kconfig. We had to convert `configs/canoe_perf.bzl` to a traditional `CONFIG_*=` format (`canoe_GKI.config`) because the LineageOS build system doesn't speak Bazel
- **Version-pinned to a specific firmware** — kernel branch is `sm8850_b_16.0.0` but current vendor blobs are from firmware `16.0.5.700`. If Qualcomm changed the KMI (Kernel Module Interface) between these versions, vendor `.ko` modules won't load
- **OPlus-heavy** — includes OPlus-specific drivers, configs, and device tree modifications that may conflict with or duplicate what LineageOS provides
- **Not tested against AOSP** — OnePlus tests against their own ROM, not against GKI compliance or AOSP boot flows

### What the ROM devs are waiting for: CLO platform drop

Qualcomm publishes official AOSP-compatible kernel platforms through [CodeLinaro](https://git.codelinaro.org/). For each SoC, CLO provides:

- **AOSP-native kernel** — designed and tested against AOSP init, SELinux, and property conventions. Works with GKI out of the box
- **Stable KMI** — guaranteed kernel module interface compatibility with vendor blobs. No ABI mismatch risk
- **Standard Kconfig** — proper `defconfig` files, no Bazel conversion needed
- **Clean modem/DSP drivers** — PIL, QRTR, QMI, and remoteproc drivers configured for AOSP userspace IPC, not vendor-specific paths
- **Tested device trees** — DTBs with correct memory maps, firmware paths, and peripheral configurations for AOSP boot flow

CLO drops for new SoCs typically lag months behind device launch. SM8850/canoe hasn't been published yet — that's what the devs are waiting for.

### Stability assessment

| | OnePlusOSS (this fix) | CLO (future) |
|---|---|---|
| Build complexity | High — Bazel→Kconfig conversion, missing files | Low — standard AOSP kernel build |
| ABI risk | **Medium-High** — kernel `16.0.0` vs blobs `16.0.5` | None — matched by Qualcomm |
| Modem/IMEI fix | Likely but unverified | Expected to work out of the box |
| OPlus driver conflicts | Possible | None |
| Long-term maintainability | Poor — tied to one firmware snapshot | Good — rebases onto CLO updates |

**Bottom line:** This fix is a best-effort approach using what's available today. It addresses the device tree configuration issues (platform naming, module lists, kernel toggle) which are needed regardless of kernel source. The kernel itself may or may not build and boot cleanly — the OnePlusOSS dump wasn't designed for this use case. Once CLO publishes the canoe platform, the device tree patches from this repo will still apply, but the kernel source files (`canoe_GKI.config`, module lists) should be regenerated from CLO's proper Kconfig.

## Notes

- The `modules.vendor_blocklist.msm.canoe` already exists in the OSS kernel source at `modules-lists/` — it just needs to be copied to the root or the BoardConfig path updated.
- The `gki_defconfig` comes from the `android_kernel_common_oneplus_sm8850` repo, which provides the GKI base config.
- Switching to OSS kernel fixes DT2W (Double Tap to Wake) and touch gestures, which are broken with the prebuilt kernel. It does **not** fix the IMEI issue (see PCBA Verification section above).
- Vendor blobs are from `CPH2745_16.0.5.700(EX01)`. Kernel source branch is `sm8850_b_16.0.0_oneplus_15`. If there's an ABI mismatch, vendor blobs may need to be re-extracted from a firmware version matching the kernel source.
- The PCBA verification happens entirely in userspace (`libqti-radio-service.so`), not in TrustZone or modem firmware — this means it is fixable from the ROM side without modifying the modem image.
