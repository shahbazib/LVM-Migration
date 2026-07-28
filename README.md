# 🚀 Safe Linux System & LVM Migration Guide

> A comprehensive, fail-safe guide to migrating a live RHEL/CentOS/Rocky Linux system (including **Zabbix** & **MySQL/MariaDB**) to a new, larger LVM disk without encountering `grub rescue`, file corruption, or SELinux permission errors.

---

## 📌 Overview

When expanding storage for critical infrastructure like Zabbix monitoring servers, simple disk expansion isn't always optimal. This step-by-step procedure covers migrating an entire operating system and database layout to a brand-new LVM-partitioned virtual disk on VMware/KVM.

### Key Highlights

* 🛡️ **Zero Data Corruption:** Database services are safely halted before migration.
* 🔐 **Preserved Security:** File permissions, ownership, ACLs, and SELinux contexts remain 100% intact.
* ⚙️ **Clean Booting:** Prevents `grub rescue` and UUID mismatches via clean `chroot` rebuilding.

---

## 📋 Prerequisites

* **OS:** RHEL 7/8/9, CentOS 7/8, Rocky Linux, or AlmaLinux
* **Privileges:** `root` access
* **Scenario:** Moving from existing disk (`/dev/sda`) to a new raw disk (`/dev/sdb`)

---

## 🛠️ Step-by-Step Migration Process

### Step 0: Scan & Detect New SCSI Disk

Force the Linux kernel to detect the new disk added via VMware without rebooting:

```bash
for host in /sys/class/scsi_host/host*/scan; do echo "- - -" > $host; done
lsblk

```

*Verify that your new raw disk (e.g., `/dev/sdb`) appears in the output.*

---

### Step 1: Disk Partitioning (`fdisk`)

Create a dedicated 2GB partition for `/boot` (non-LVM) and assign the rest to LVM:

```bash
(echo o; echo n; echo p; echo 1; echo ; echo +2G; echo n; echo p; echo 2; echo ; echo ; echo t; echo 2; echo 8e; echo w) | fdisk /dev/sdb

```

---

### Step 2: LVM Creation & File System Formatting

```bash
# 1. Format boot partition
mkfs.ext4 -F /dev/sdb1

# 2. Create PV and VG
pvcreate /dev/sdb2
vgcreate vg_zabbix /dev/sdb2

# 3. Create Logical Volumes (Adjust sizes as needed)
lvcreate -L 20G -n lv_root vg_zabbix
lvcreate -L 2G -n lv_tmp vg_zabbix
lvcreate -L 200G -n lv_mysql vg_zabbix

# 4. Format LVM volumes
mkfs.xfs -f /dev/vg_zabbix/lv_root
mkfs.xfs -f /dev/vg_zabbix/lv_tmp
mkfs.xfs -f /dev/vg_zabbix/lv_mysql

```

---

### Step 2.5: Stop Critical Services (Data Integrity)

Stop active services to freeze database state and avoid copying corrupted table files:

```bash
systemctl stop zabbix-server zabbix-agent
systemctl stop mariadb 2>/dev/null || systemctl stop mysqld 2>/dev/null
systemctl stop httpd 2>/dev/null || systemctl stop nginx 2>/dev/null

# Verify status (must be inactive/dead)
systemctl status zabbix-server mysqld | grep "Active:"

```

---

### Step 3: Mount New File Systems & Copy Data

```bash
# Prepare mount points
mkdir -p /mnt/new_sys
mount /dev/vg_zabbix/lv_root /mnt/new_sys

mkdir -p /mnt/new_sys/{boot,tmp,var/lib/mysql}

mount /dev/sdb1 /mnt/new_sys/boot
mount /dev/vg_zabbix/lv_tmp /mnt/new_sys/tmp
mount /dev/vg_zabbix/lv_mysql /mnt/new_sys/var/lib/mysql

# Copy files while retaining ALL attributes, permissions, and links
cp -a -x / /mnt/new_sys/
cp -a -r /boot/* /mnt/new_sys/boot/
cp -a -r /tmp/* /mnt/new_sys/tmp/
cp -a -r -p /var/lib/mysql/* /mnt/new_sys/var/lib/mysql/

# Restore SELinux contexts & enforce database ownership
restorecon -R /mnt/new_sys
chown -R mysql:mysql /mnt/new_sys/var/lib/mysql

```

---

### Step 4: Configure `/etc/fstab`

Edit `/mnt/new_sys/etc/fstab` on the target disk:

```bash
nano /mnt/new_sys/etc/fstab

```

Comment out all old `UUID=...` entries and insert device mappings:

```text
/dev/vg_zabbix/lv_root    /                xfs     defaults    0 0
/dev/sda1                 /boot            ext4    defaults    1 2
/dev/vg_zabbix/lv_tmp     /tmp             xfs     defaults    0 0
/dev/vg_zabbix/lv_mysql   /var/lib/mysql   xfs     defaults    0 0

```

---

### Step 5: Fix GRUB Bootloader via `chroot`

*This step prevents the dreaded `grub rescue>` screen caused by legacy UUID references.*

```bash
# Mount system pseudo-filesystems
mount --bind /dev /mnt/new_sys/dev
mount --bind /proc /mnt/new_sys/proc
mount --bind /sys /mnt/new_sys/sys

# Chroot into the new environment
chroot /mnt/new_sys

# Install GRUB MBR on the new disk
grub2-install /dev/sdb

# Regenerate grub.cfg from scratch
grub2-mkconfig -o /boot/grub2/grub.cfg

# Exit chroot
exit

```

---

### Step 6: Verification & Cleanup

Before unmounting, ensure target configurations are valid:

```bash
# 1. Verify fstab
cat /mnt/new_sys/etc/fstab

# 2. Check GRUB config file size (should be ~4-10 KB)
ls -lh /mnt/new_sys/boot/grub2/grub.cfg

# 3. Check mounted DB volume size
df -h /mnt/new_sys/var/lib/mysql

```

#### Finalize and Power Off:

```bash
umount -R /mnt/new_sys
poweroff

```

---

## 🎯 Finalizing in VMware / Hypervisor

1. Open Hypervisor VM Settings.
2. **Disconnect / Remove** the old original virtual disk (`sda`).
3. Set the new migrated disk as the **Primary Boot Disk**.
4. **Power On** the Virtual Machine.

---

## 📝 License

Distributed under the MIT License. Feel free to modify and use in enterprise environments!
