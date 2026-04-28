# Fix: Generic IMEI on LineageOS 23 for OnePlus 15 (Infiniti)

## Problem

The unofficial LineageOS 23.2 ROM reports a **generic Qualcomm IMEI** instead of the device's real IMEI. Some carriers reject devices with a generic IMEI, breaking cellular connectivity.

## Root Cause

The ROM uses a **prebuilt kernel** extracted from stock OxygenOS (`android_device_oneplus_infiniti-kernel`). This prebuilt kernel's Image and DTB (Device Tree Blob) were built for OxygenOS's userspace. When running under LineageOS (different init sequences, SELinux policies, system properties), the modem subsystem doesn't properly initialize its IPC channels — causing the modem to fall back to a default/generic IMEI instead of reading the real one from NV storage.

**All modem-critical kernel modules ARE present** in the prebuilt (qrtr, mhi, rmnet, etc.) — the issue is at the kernel Image/DTB level, not missing modules.

## Fix

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

**Module list files** (extracted from prebuilt kernel's `modules.load`):
- `modules.system_dlkm.list.msm.canoe` (82 modules)
- `modules.vendor_dlkm.list.msm.canoe` (435 modules)
- `modules.vendor_dlkm.list.oplus.canoe` (161 modules)
- `modules.vendor_boot.list.msm.canoe` (120 modules)
- `modules.vendor_boot.list.oplus.canoe` (18 modules)
- `modules.recovery.list.msm.canoe` (316 modules)
- `modules.recovery.list.oplus.canoe` (149 modules)

**Kernel config fragment** (generated from `configs/canoe_perf.bzl`):
- `vendor/canoe_GKI.config` (468 CONFIG_ options)

**OPlus config fragment** (needs to be created — extract oplus-specific configs from `canoe_perf.bzl`):
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

```bash
# Check real IMEI (should NOT be generic Qualcomm)
adb shell service call iphonesubinfo 1

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

## Notes

- The `modules.vendor_blocklist.msm.canoe` already exists in the OSS kernel source at `modules-lists/` — it just needs to be copied to the root or the BoardConfig path updated.
- The `gki_defconfig` comes from the `android_kernel_common_oneplus_sm8850` repo, which provides the GKI base config.
- Switching to OSS kernel should also fix DT2W (Double Tap to Wake) and touch gestures, which are broken with the prebuilt kernel.
- Vendor blobs are from `CPH2745_16.0.5.700(EX01)`. Kernel source branch is `sm8850_b_16.0.0_oneplus_15`. If there's an ABI mismatch, vendor blobs may need to be re-extracted from a firmware version matching the kernel source.
