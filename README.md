# Linux Forensic Timeline Generation Guide

This guide provides step-by-step procedures for building both **Filesystem Timelines** (MACB metadata via The Sleuth Kit) and **Super Timelines** (deep system events via Plaso / `log2timeline`) for Linux disk images on a SIFT Workstation.

It accounts for both standard partition layouts (e.g., direct `ext4`/`xfs` partitions) and complex setups utilizing **Logical Volume Management (LVM)** (e.g., RHEL, CentOS, Ubuntu LVM defaults).

---

## Prerequisites

Ensure the required forensic toolkits are installed on your workstation:

```bash
sudo apt update && sudo apt install -y sleuthkit plaso libewf-tools lvm2 qemu-utils
```

---

## Phase 1: Expose the Image Block Structure

Before generating timelines, expose the raw disk blocks based on your image format.

### Option A: Expert Witness Format (`.E01`)
```bash
sudo mkdir -p /mnt/ewf
sudo ewfmount /path/to/image.E01 /mnt/ewf
sudo losetup -fP /mnt/ewf/ewf1
```

### Option B: Virtual Appliance Disks (`.qcow2`, `.vmdk`, `.vhdx`)
```bash
sudo modprobe nbd max_part=16
sudo qemu-nbd --connect=/dev/nbd0 --read-only /path/to/image.vmdk
```

> **Note:** Identify the loop or NBD device created using `losetup -a` or `lsblk`. For the remainder of this guide, `/dev/loop0` or `/dev/nbd0` represents your base disk device.

---

## Phase 2: Identify the Filesystem & Partition Layout

Determine whether the disk uses direct partitions or LVM wrappers:

```bash
sudo mmls /dev/loop0
```

Look at the partition descriptions:
* **Non-LVM System:** Shows direct Linux filesystems (e.g., `Linux (0x83)`, `Linux native`, `EFI System`).
* **LVM System:** Shows an explicit LVM container partition (e.g., `Linux Logical Volume Manager (0x8e)` or GPT LVM UUIDs).

---

## Phase 3: Workflow Selection

Choose the workflow corresponding to your layout identified in Phase 2:

* **Workflow 1:** [Standard Non-LVM Linux Systems](#workflow-1-standard-non-lvm-linux-systems)
* **Workflow 2:** [LVM-Based Linux Systems](#workflow-2-lvm-based-linux-systems)

---

## Workflow 1: Standard Non-LVM Linux Systems

If the target disk uses standard partitions without LVM, you can calculate the sector offset directly.

### 1. Filesystem Timeline (TSK / `fls` + `mactime`)

#### Step A: Find the Start Sector
Run `mmls` and identify the start sector of the primary Linux filesystem partition (e.g., sector `2048`).

#### Step B: Generate the Bodyfile
Use `fls` with the `-o` (offset in sectors) flag:

```bash
sudo fls -r -m "/" -o 2048 /dev/loop0 > non_lvm_bodyfile.txt
```

#### Step C: Process into a MACB Timeline
Convert the raw bodyfile into a human-readable CSV timeline using `mactime`:

```bash
mactime -b non_lvm_bodyfile.txt -d -z UTC > filesystem_timeline.csv
```

---

### 2. Super Timeline (Plaso / `log2timeline`)

Plaso automatically parses disk partitions and builds a deep event timeline from file metadata, system logs (`/var/log`), journald, shell histories, and application logs.

```bash
# Step A: Extract events to a storage file
log2timeline.py --storage-file non_lvm_events.plaso /dev/loop0

# Step B: Export storage file to CSV
psort.py -o l2tcsv -w super_timeline.csv non_lvm_events.plaso
```

---

## Workflow 2: LVM-Based Linux Systems

When a Linux system utilizes LVM, standard sector offset calculations via `fls -o` will fail because the filesystem resides inside a virtual Logical Volume rather than a physical partition boundary.

### 1. Scan and Activate LVM Volumes

Scan the loop/NBD block device for underlying Volume Groups:

```bash
# 1. Scan physical and volume groups
sudo pvscan
sudo vgscan

# 2. Activate all detected Volume Groups
sudo vgchange -ay

# 3. List active Logical Volume paths
sudo lvs
```

*(This exposes mapped block devices under `/dev/mapper/<VolumeGroupName>-<LogicalVolumeName>`, e.g., `/dev/mapper/vg_gaia-lv_current` or `/dev/mapper/ubuntu--vg-ubuntu--lv`).*

---

### 2. Filesystem Timeline (TSK / `fls` + `mactime`)

Because `fls` cannot calculate LVM offsets internally, point `fls` **directly at the mapped logical volume block device**:

#### Step A: Generate the Bodyfile from the Mapped Volume
```bash
sudo fls -r -m "/" /dev/mapper/vg_gaia-lv_current > lvm_bodyfile.txt
```

#### Step B: Process into a MACB Timeline
```bash
mactime -b lvm_bodyfile.txt -d -z UTC > lvm_filesystem_timeline.csv
```

---

### 3. Super Timeline (Plaso / `log2timeline`)

Plaso includes native support for `libvslvm` and can automatically parse LVM volume structures directly from the raw disk image file without requiring manual `vgchange` volume mapping.

#### Option A: Target the Base Block Device Directly (Recommended)
```bash
# Plaso automatically detects and parses LVM volumes inside the image
log2timeline.py --storage-file lvm_events.plaso /dev/loop0
psort.py -o l2tcsv -w lvm_super_timeline.csv lvm_events.plaso
```

#### Option B: Target a Specific Activated Logical Volume
If you only want to process a specific logical volume (e.g., `/var/log` volume only):

```bash
log2timeline.py --storage-file lv_log_events.plaso /dev/mapper/vg_gaia-lv_log
psort.py -o l2tcsv -w lv_log_super_timeline.csv lv_log_events.plaso
```

---

## Filtering Timelines by Date Range

When analyzing super timelines or filesystem timelines, filter results using `psort.py` or `mactime` to focus on specific incident windows:

### Filter `mactime` (Filesystem Timeline)
```bash
# Show events between October 1, 2025 and October 15, 2025
mactime -b lvm_bodyfile.txt -d -z UTC 2025-10-01..2025-10-15 > filtered_timeline.csv
```

### Filter `psort.py` (Plaso Super Timeline)
```bash
# Filter events for a specific timeframe during export
psort.py -o l2tcsv \
  --slice "2025-10-01T00:00:00 to 2025-10-15T23:59:59" \
  -w filtered_super_timeline.csv lvm_events.plaso
```

---

## Teardown & Cleanup

Always unmount and deactivate block devices in reverse order to preserve image integrity:

```bash
# 1. Step out of any mounted directories
cd ~

# 2. Deactivate Volume Groups (LVM systems)
sudo vgchange -an

# 3. Detach loop or NBD devices
sudo losetup -d /dev/loop0
# OR
sudo qemu-nbd --disconnect /dev/nbd0

# 4. Unmount EWF containers (if applicable)
sudo umount /mnt/ewf
```
