# 📖 Complete Linux Disk Management, LVM Architecture & Migration Guide

A comprehensive, production-grade guide covering LVM concepts, live disk expansion, safe shrinking, snapshots, and step-by-step system/database migration for Enterprise Linux systems (**RHEL, CentOS, Rocky Linux, AlmaLinux, Ubuntu, Debian**).

![Bash](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnubash&logoColor=white)
![LVM](https://img.shields.io/badge/Storage-LVM2-blue)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 🎯 Table of Contents
1. [Prerequisites & Safety Checklist](#0-prerequisites--safety-checklist)
2. [Understanding LVM Architecture](#1-understanding-lvm-architecture)
3. [Scenario A: Extending an Existing Disk Live](#2-scenario-a-extending-an-existing-disk-live)
4. [Scenario B: Complete 0-to-100 System & DB Migration](#3-scenario-b-complete-0-to-100-system--db-migration)
5. [Scenario C: Shrinking a Logical Volume Safely](#4-scenario-c-shrinking-a-logical-volume-safely)
6. [Scenario D: Adding a New Disk to an Existing VG](#5-scenario-d-adding-a-new-disk-to-an-existing-vg)
7. [LVM Snapshots (Backup Before Risky Operations)](#6-lvm-snapshots-backup-before-risky-operations)
8. [Monitoring & Alerting](#7-monitoring--alerting)
9. [Troubleshooting Common Failures](#8-troubleshooting-common-failures)
10. [Cheat Sheet](#9-cheat-sheet)
11. [FAQ](#10-faq)
12. [License](#-license)

---

## 0. Prerequisites & Safety Checklist

> ⚠️ **Read this before touching production disks.** LVM operations are generally safe, but partition table edits, GRUB reinstalls, and shrink operations are **destructive if done wrong**.

- [ ] Take a full backup (or at least a snapshot — see [Section 6](#6-lvm-snapshots-backup-before-risky-operations)) before any migration or shrink.
- [ ] Confirm you have out-of-band access to the server (iDRAC/iLO/hypervisor console, or cloud serial console) in case GRUB or networking breaks.
- [ ] Verify free space with `vgs` / `pvs` before running `lvextend` or `lvcreate`.
- [ ] Know your filesystem type (`lsblk -f` or `df -Th`) — **XFS cannot be shrunk**, only EXT4 can.
- [ ] Schedule a maintenance window; stopping databases/services causes downtime.
- [ ] Confirm the target/new disk device name (`/dev/sdb`, `/dev/nvme1n1`, etc.) with `lsblk` — **never** guess a device name, wiping the wrong disk is irreversible.
- [ ] Required packages installed: `lvm2`, `parted` or `fdisk`, `xfsprogs` (for XFS), `e2fsprogs` (for EXT4), `grub2-tools` (RHEL family) or `grub-pc`/`grub-efi` (Debian/Ubuntu).

---

## 1. Understanding LVM Architecture

Logical Volume Manager (LVM) abstracts physical storage into flexible volumes. Understanding its three core layers is essential:

```text
+------------------------------------------------------------------+
|                    Physical Storage (/dev/sdb)                   |
+------------------------------------------------------------------+
          │                                        │
          ▼                                        ▼
 +------------------+                   +--------------------+
 | /dev/sdb1 (/boot)|                   | /dev/sdb2 (LVM PV) |
 +------------------+                   +--------------------+
                                                   │
                                                   ▼
                                        +--------------------+
                                        | Volume Group (VG)  |
                                        |   (vg_zabbix)      |
                                        +--------------------+
                                           │   │          │
                     ┌─────────────────────┘   │          └─────────────────────┐
                     ▼                         ▼                                ▼
          +--------------------+    +--------------------+            +--------------------+
          | Logical Vol (LV)   |    | Logical Vol (LV)   |            | Logical Vol (LV)   |
          |     lv_root        |    |      lv_tmp        |            |     lv_mysql       |
          +--------------------+    +--------------------+            +--------------------+
                     │                         │                                │
                     ▼                         ▼                                ▼
               Ext4 / XFS                 Ext4 / XFS                       Ext4 / XFS
```

| Layer | Description | Example |
|---|---|---|
| **Physical Volume (PV)** | Raw disk/partition initialized for LVM use | `/dev/sdb2` |
| **Volume Group (VG)** | Pool of storage aggregated from one or more PVs | `vg_zabbix` |
| **Logical Volume (LV)** | Virtual partition carved from a VG, holds a filesystem | `lv_root`, `lv_mysql` |
| **Physical Extent (PE)** | Smallest allocatable chunk of a PV (default 4MB) | — |
| **Logical Extent (LE)** | Maps 1:1 to PEs, makes up an LV | — |

**Why use LVM instead of raw partitions?**
- Online resizing of volumes without unmounting (in most cases).
- Ability to span a single logical volume across multiple physical disks.
- Native snapshot support for consistent backups.
- Easier disk replacement/migration without reinstalling the OS.

---

## 2. Scenario A: Extending an Existing Disk Live

Use this method if you expanded an existing virtual disk (e.g., increased disk size in VMware/Hyper-V/AWS EBS) and need to expand your partitions **without rebooting**.

### Step 1: Rescan the Disk in Linux

Force the kernel to recognize the new disk size:

```bash
echo 1 > /sys/class/block/sda/device/rescan
```

If the disk isn't detected, rescan the whole SCSI bus instead:

```bash
for host in /sys/class/scsi_host/host*/scan; do echo "- - -" > $host; done
```

Confirm the kernel sees the new size:

```bash
lsblk
cat /sys/class/block/sda/size
```

### Step 2: Grow the Partition (if using a partition, not a raw PV)

If your PV sits on a partition (e.g. `/dev/sda2`) rather than a whole disk, grow the partition first with `growpart` or `parted`:

```bash
# Using cloud-utils-growpart
growpart /dev/sda 2

# OR using parted interactively
parted /dev/sda resizepart 2 100%
```

### Step 3: Resize the Physical Volume (PV)

Update the LVM metadata to recognize the extra space:

```bash
pvresize /dev/sda2
```

Verify the new size:

```bash
pvs
pvdisplay /dev/sda2
```

### Step 4: Extend the Logical Volume & File System

Use the `-r` (`--resizefs`) flag to automatically expand the underlying filesystem (**XFS** or **EXT4**) online.

**Add a specific amount of space (e.g., +50GB):**
```bash
lvextend -L +50G /dev/vg_zabbix/lv_mysql -r
```

**Allocate 100% of remaining free space in VG:**
```bash
lvextend -l +100%FREE /dev/vg_zabbix/lv_root -r
```

**If `-r` isn't available (older lvm2 versions), resize manually:**
```bash
lvextend -L +50G /dev/vg_zabbix/lv_mysql
xfs_growfs /var/lib/mysql          # for XFS
resize2fs /dev/vg_zabbix/lv_mysql  # for EXT4
```

### Step 5: Verify

```bash
df -Th /var/lib/mysql
lvs -o lv_name,lv_size,vg_name
```

---

## 3. Scenario B: Complete 0-to-100 System & DB Migration

Use this method when migrating a live operating system (including databases like MySQL/MariaDB and Zabbix) to a newly added raw disk without data corruption or GRUB boot issues.

### Step 0: Detect the New Disk

Scan the SCSI bus to register the newly attached disk without restarting:

```bash
for host in /sys/class/scsi_host/host*/scan; do echo "- - -" > $host; done
lsblk
```

*(Verify your new target disk, e.g., `/dev/sdb`, appears — double-check size and device name.)*

### Step 1: Partition the Target Disk

Create a 2GB standard partition for `/boot` and dedicate the remaining space to LVM:

```bash
(echo o; echo n; echo p; echo 1; echo ; echo +2G; echo n; echo p; echo 2; echo ; echo ; echo t; echo 2; echo 8e; echo w) | fdisk /dev/sdb
```

> 💡 On UEFI systems, add a small (512MB–1GB) `EFI System Partition` (type `ef00` in `gdisk`) before the `/boot` partition.

### Step 2: Initialize LVM & Format File Systems

```bash
# 1. Format boot partition
mkfs.ext4 -F /dev/sdb1

# 2. Create PV and VG
pvcreate /dev/sdb2
vgcreate vg_zabbix /dev/sdb2

# 3. Create Logical Volumes
lvcreate -L 20G  -n lv_root  vg_zabbix
lvcreate -L 2G   -n lv_tmp   vg_zabbix
lvcreate -L 200G -n lv_mysql vg_zabbix

# 4. Format LVM volumes with XFS
mkfs.xfs -f /dev/vg_zabbix/lv_root
mkfs.xfs -f /dev/vg_zabbix/lv_tmp
mkfs.xfs -f /dev/vg_zabbix/lv_mysql
```

### Step 3: Stop Services for a Consistent Snapshot

Prevent active writes to guarantee database integrity during copy:

```bash
systemctl stop zabbix-server zabbix-agent
systemctl stop mariadb 2>/dev/null || systemctl stop mysqld 2>/dev/null
systemctl stop httpd 2>/dev/null || systemctl stop nginx 2>/dev/null

# Confirm services are inactive
systemctl status zabbix-server mysqld | grep "Active:"
```

### Step 4: Mount Structures & Copy Data Safely

Copy files while maintaining permissions, ACLs, and ownership:

```bash
# Prepare mount points
mkdir -p /mnt/new_sys
mount /dev/vg_zabbix/lv_root /mnt/new_sys

mkdir -p /mnt/new_sys/{boot,tmp,var/lib/mysql}

mount /dev/sdb1 /mnt/new_sys/boot
mount /dev/vg_zabbix/lv_tmp /mnt/new_sys/tmp
mount /dev/vg_zabbix/lv_mysql /mnt/new_sys/var/lib/mysql

# Copy operating system files accurately
# -a preserves permissions/timestamps/links, -x stays on one filesystem
rsync -aAXx --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /mnt/new_sys/

cp -a /boot/.               /mnt/new_sys/boot/
cp -a /tmp/.                /mnt/new_sys/tmp/
cp -a -p /var/lib/mysql/.   /mnt/new_sys/var/lib/mysql/

# Restore SELinux labels & fix DB ownership
restorecon -R /mnt/new_sys
chown -R mysql:mysql /mnt/new_sys/var/lib/mysql
```

> 📌 `rsync -aAXx` is preferred over plain `cp -a` for whole-system copies — it preserves ACLs/xattrs and is resumable if interrupted.

### Step 5: Update `/etc/fstab` Mapping

Edit `/mnt/new_sys/etc/fstab` to point to logical volume devices instead of hardcoded legacy UUIDs:

```bash
nano /mnt/new_sys/etc/fstab
```

*Comment out all old `UUID=...` entries and append:*

```text
/dev/vg_zabbix/lv_root    /                xfs     defaults    0 0
/dev/sda1                 /boot            ext4    defaults    1 2
/dev/vg_zabbix/lv_tmp     /tmp             xfs     defaults    0 0
/dev/vg_zabbix/lv_mysql   /var/lib/mysql   xfs     defaults    0 0
```

> ⚠️ Note: `/dev/sda1` above assumes the **old** boot disk becomes `/dev/sda` after the original disk is detached and the new disk takes its place — adjust device names to match your actual post-migration layout. Using LVM device paths (`/dev/vg_name/lv_name`) instead of UUIDs avoids this entire class of problem for LVM-backed mounts.

### Step 6: Rebuild GRUB in `chroot` (Prevents `grub rescue`)

Entering `chroot` regenerates `grub.cfg` without referencing old disk UUIDs:

```bash
# Bind pseudo-filesystems
mount --bind /dev  /mnt/new_sys/dev
mount --bind /proc /mnt/new_sys/proc
mount --bind /sys  /mnt/new_sys/sys

# Chroot into new system
chroot /mnt/new_sys

# Install GRUB onto MBR of new disk (BIOS/Legacy)
grub2-install /dev/sdb
# Debian/Ubuntu equivalent:
# grub-install /dev/sdb

# Rebuild clean grub configuration
grub2-mkconfig -o /boot/grub2/grub.cfg
# Debian/Ubuntu equivalent:
# update-grub

# Rebuild initramfs so it can find the new LVM root at boot
dracut -f --regenerate-all
# Debian/Ubuntu equivalent:
# update-initramfs -u -k all

# Exit chroot
exit
```

### Step 7: Final Verification & Shutdown

```bash
# Verify GRUB file size (should be non-zero, ~4-10KB)
ls -lh /mnt/new_sys/boot/grub2/grub.cfg

# Verify mounted database volume size
df -h /mnt/new_sys/var/lib/mysql

# Unbind pseudo-filesystems and unmount cleanly
umount /mnt/new_sys/dev /mnt/new_sys/proc /mnt/new_sys/sys
umount -R /mnt/new_sys
poweroff
```

1. Detach/remove the old disk (`/dev/sda`) in VMware/hypervisor settings **only after confirming a successful boot from the new disk**, or keep it detached-but-present as a rollback option for one boot cycle.
2. Power on the virtual machine and verify all services start correctly:
   ```bash
   systemctl status zabbix-server mariadb httpd
   ```

---

## 4. Scenario C: Shrinking a Logical Volume Safely

> ⚠️ **XFS cannot be shrunk.** This procedure only works for **EXT4**. If your LV is XFS, you must back up the data, recreate a smaller LV, format it, and restore data instead.

```bash
# 1. Unmount the volume first (cannot shrink a mounted EXT4 filesystem)
umount /dev/vg_zabbix/lv_tmp

# 2. Check filesystem integrity before resizing
e2fsck -f /dev/vg_zabbix/lv_tmp

# 3. Shrink the filesystem FIRST, to a size smaller than the target LV size
resize2fs /dev/vg_zabbix/lv_tmp 1G

# 4. THEN shrink the logical volume to match (or slightly larger, never smaller than the fs)
lvreduce -L 1G /dev/vg_zabbix/lv_tmp

# 5. Remount and verify
mount /dev/vg_zabbix/lv_tmp /tmp
df -h /tmp
```

**Order matters:** shrink the filesystem before the LV. Doing it in reverse truncates the filesystem's data blocks and causes irreversible data loss.

---

## 5. Scenario D: Adding a New Disk to an Existing VG

Use this when you need more capacity but don't want to resize an existing disk — instead you attach an additional disk and extend the VG across it.

```bash
# 1. Detect the new disk
lsblk

# 2. Partition it (or use the whole disk as a PV directly)
pvcreate /dev/sdc

# 3. Extend the existing Volume Group with the new PV
vgextend vg_zabbix /dev/sdc

# 4. Confirm the VG now has more free space
vgs
vgdisplay vg_zabbix

# 5. Extend an LV using the newly added space
lvextend -l +100%FREE /dev/vg_zabbix/lv_mysql -r
```

> 💡 An LV can now be striped/spread across multiple PVs. Use `pvs -o pv_name,vg_name,pv_free` to see which physical disk has free extents.

---

## 6. LVM Snapshots (Backup Before Risky Operations)

Snapshots let you capture a consistent point-in-time copy of an LV — essential before migrations, upgrades, or shrink operations.

```bash
# 1. Create a snapshot (ensure the VG has enough free space, ~10-20% of origin size)
lvcreate -L 10G -s -n lv_mysql_snap /dev/vg_zabbix/lv_mysql

# 2. Mount the snapshot read-only to inspect/back it up
mkdir -p /mnt/snap
mount -o ro /dev/vg_zabbix/lv_mysql_snap /mnt/snap

# 3. Back up data from the snapshot (doesn't affect live DB writes)
tar -czf /backup/mysql_snapshot_$(date +%F).tar.gz -C /mnt/snap .

# 4. Roll back to the snapshot if something goes wrong (origin LV must be inactive)
umount /dev/vg_zabbix/lv_mysql
lvconvert --merge /dev/vg_zabbix/lv_mysql_snap

# 5. Remove the snapshot once no longer needed
umount /mnt/snap
lvremove /dev/vg_zabbix/lv_mysql_snap
```

> ⚠️ Snapshots fill up as the origin volume changes — monitor with `lvs` (`Data%` column) and remove or extend them before they hit 100%, or they'll be dropped automatically and become unusable.

---

## 7. Monitoring & Alerting

Keep an eye on VG/LV utilization to avoid unplanned outages:

```bash
# Quick capacity check across everything
vgs -o vg_name,vg_size,vg_free
lvs -o lv_name,vg_name,lv_size,data_percent
df -Th

# Set up a simple cron-based alert (example, adapt to your monitoring stack)
*/15 * * * * root df -h /var/lib/mysql | awk 'NR==2{gsub("%","",$5); if ($5+0 > 85) print "WARN: /var/lib/mysql at "$5"%"}' | logger -t disk-alert
```

For Zabbix-monitored hosts, use the built-in `vfs.fs.size` and `vfs.fs.dependent.size` item keys with a discovery rule on mounted filesystems, and set trigger thresholds (e.g., 80% warning / 90% critical).

---

## 8. Troubleshooting Common Failures

| Symptom | Likely Cause | Fix |
|---|---|---|
| `grub rescue>` prompt after migration | GRUB installed to wrong disk, or `grub.cfg` references old UUID | Boot rescue media, chroot in, re-run `grub2-install` + `grub2-mkconfig` |
| `pvresize` shows no size change | Underlying partition wasn't grown first | Run `growpart`/`parted resizepart` before `pvresize` |
| `lvextend: Insufficient free space` | VG has no free extents left | Run `vgextend` with an additional PV, or free space with `lvreduce` elsewhere |
| Filesystem still shows old size after `lvextend` | Forgot the `-r` flag or ran it on wrong path | Manually run `xfs_growfs <mountpoint>` (XFS) or `resize2fs <lv-path>` (EXT4) |
| `xfs_growfs` "not mounted" error | XFS growfs targets the **mount point**, not the device | Use `xfs_growfs /mountpoint`, never `xfs_growfs /dev/...` |
| VM won't boot after removing old disk | Old disk was still referenced somewhere in fstab/grub | Re-check `fstab` and confirm `grub2-install` targeted the new disk's boot sector |
| Snapshot LV disappears/errors | Snapshot exceeded its allocated COW space | Always allocate snapshots generously (15–20% of origin) and monitor `data_percent` |
| Database corruption after copy | Files copied while DB was still running | Always stop DB services (or use proper hot-backup tools like `mysqldump`/`xtrabackup`) before copying data files directly |

---

## 9. Cheat Sheet

| Command | Description |
| --- | --- |
| `pvs` / `pvdisplay` | Display Physical Volume status and free extents |
| `vgs` / `vgdisplay` | Display Volume Group status and unallocated pool capacity |
| `lvs` / `lvdisplay` | Display Logical Volume configurations and sizes |
| `lsblk -f` | Tree view of disks, partitions, filesystems, and UUIDs |
| `df -Th` | Disk usage with human-readable values and filesystem types |
| `pvcreate <dev>` | Initialize a device/partition as a Physical Volume |
| `vgcreate <vg> <dev>` | Create a new Volume Group from one or more PVs |
| `vgextend <vg> <dev>` | Add a new PV to an existing Volume Group |
| `lvcreate -L <size> -n <lv> <vg>` | Create a new Logical Volume of a fixed size |
| `lvextend -L +<size> <lv> -r` | Grow an LV and its filesystem together (online) |
| `lvreduce -L <size> <lv>` | Shrink an LV (EXT4 only, filesystem must be shrunk first) |
| `xfs_growfs /mountpoint` | Manually expand an XFS filesystem |
| `resize2fs /dev/VG/LV` | Manually expand/shrink an EXT4 filesystem |
| `lvcreate -s -n <name> <lv>` | Create a snapshot of an existing LV |
| `lvconvert --merge <snap>` | Roll back an LV to a snapshot |

---

## 10. FAQ

**Q: Can I shrink an XFS filesystem?**
A: No. XFS has no shrink support. You must back up, recreate a smaller LV, format, and restore.

**Q: Do I need to unmount the filesystem to extend it?**
A: No — both XFS (`xfs_growfs`) and EXT4 (`resize2fs`) support **online growth** while mounted. Shrinking, however, requires unmounting (EXT4 only).

**Q: What happens if my VG runs out of free space mid-`lvextend`?**
A: The command fails safely without applying partial changes — add a PV via `vgextend` or free space elsewhere, then retry.

**Q: Is `cp -a` safe for copying a live root filesystem?**
A: It's usable but `rsync -aAXx` is generally preferred for whole-system migrations since it preserves ACLs/xattrs, handles interruptions better, and gives progress output.

**Q: My new disk shows up as `/dev/sdb` now but was `/dev/sda` before migration — will that break anything?**
A: Not if you use LVM device paths or **filesystem UUIDs/labels** in `fstab` and GRUB rather than raw device names like `/dev/sda1`. Re-run `blkid` after migration to confirm UUIDs and update `fstab`/`grub.cfg` accordingly if needed.

---

## 📝 License

Distributed under the MIT License. Feel free to modify and use in enterprise environments.

---

### 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you'd like to change. Please make sure to test any command changes in a non-production environment before submitting.
