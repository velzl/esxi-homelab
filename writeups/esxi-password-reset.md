# ESXi Root Password Reset via Linux Live Boot

## Overview

The HP ProLiant DL360p Gen8 was acquired with VMware ESXi 6.5 already installed but no known root credentials. The goal was to recover access without wiping the drives, preserving all existing VMs, datastores, and Windows Server licenses.

---

## Environment

| Item | Details |
|------|---------|
| Server | HP ProLiant DL360p Gen8 |
| Hypervisor | VMware ESXi 6.5.0 Build 9298722 |
| Recovery OS | SystemRescue 11.x (bootable USB) |
| USB Flash Tool | Rufus |

---


## How ESXi Stores Passwords

ESXi does not use a standard Linux filesystem at runtime. Passwords are stored in `/etc/shadow` inside a compressed archive chain:

```
sda5 (ESXi boot partition)
  └── state.tgz
        └── local.tgz
              └── etc/shadow
```

The key insight is that `state.tgz` on the boot partition is the persistent config store — modifying it and writing it back is equivalent to changing the live system config.

---

## Steps

### 1. Create Bootable Recovery USB

Downloaded SystemRescue ISO from systemrescue.org and flashed to USB using Rufus with MBR partition scheme.

> Ubuntu was attempted first but failed to boot on server hardware due to GPU driver issues. SystemRescue booted successfully on the first attempt.

### 2. Boot Server from USB

Powered on the server and pressed **F11** at the HP POST screen to access the one-time boot menu. Selected **USB DriveKey** and booted into SystemRescue.

### 3. Identify Partition Layout

```bash
fdisk -l
```

Output showed the following relevant partitions on `/dev/sda`:

| Partition | Size | Type |
|-----------|------|------|
| sda1 | 4M | EFI System |
| sda2 | 4G | Microsoft basic data |
| sda3 | 1.1T | VMware VMFS (datastore) |
| sda5 | 250M | Microsoft basic data (ESXi boot) |
| sda6 | 250M | Microsoft basic data (ESXi boot mirror) |
| sda7 | 110M | VMware Diagnostic |
| sda8 | 286M | Microsoft basic data |
| sda9 | 2.5G | VMware Diagnostic |

`/dev/sdb` was identified as the SystemRescue USB (7.3G, FAT32).

### 4. Mount the ESXi Boot Partition

```bash
mkdir /mnt/esxi5
mount /dev/sda5 /mnt/esxi5
ls /mnt/esxi5
```

`sda5` contained driver/module `.v00` files — the boot modules partition, not the config partition.

```bash
umount /mnt/esxi5
mount /dev/sda6 /mnt/esxi5
ls /mnt/esxi5
```

`sda6` showed `boot.cfg` and `imgdb.tgz` — still not the right partition.

After further investigation, `state.tgz` was located on `sda5`:

```bash
find /mnt/esxi5 -name "state.tgz"
# /mnt/esxi5/state.tgz
```

### 5. Extract the Archive Chain

Copied `state.tgz` to `/tmp` to avoid modifying the mounted partition directly:

```bash
cp /mnt/esxi5/state.tgz /tmp/
cd /tmp
tar xzf state.tgz
```

This extracted `local.tgz`. Then:

```bash
tar xzf local.tgz
ls etc/
```

Output confirmed `shadow` was present alongside `passwd`, `hosts`, `ssh`, and other config files.

### 6. Inspect the Shadow File

```bash
cat etc/shadow
```

The root entry showed a populated SHA-512 hash (`$6$...`), confirming the password was set. Other accounts present: `nobody`, `nfsnobody`, `dcui`, `daemon`, `vpxuser`, `admin`.

### 7. Clear Password Hashes

Used `sed` to replace the password hash field (between the first and second `:`) with an empty string for root, vpxuser, and admin:

```bash
sed -i 's/^root:[^:]*:/root::/' etc/shadow
sed -i 's/^vpxuser:[^:]*:/vpxuser::/' etc/shadow
sed -i 's/^admin:[^:]*:/admin::/' etc/shadow
```

Verified the changes:

```bash
cat etc/shadow
# root::...
# vpxuser::...
# admin::...
```

### 8. Repack and Write Back

Repacked the archive chain in reverse order:

```bash
tar czf local.tgz etc
tar czf state.tgz local.tgz
cp state.tgz /mnt/esxi5/state.tgz
```

### 9. Reboot into ESXi

```bash
reboot
```

Removed the USB during reboot. Server booted back into ESXi. Pressed **F2** on the DCUI screen and logged in as root with a blank password — successful.

---

## Result

- Root access restored to ESXi host
- All existing VMs intact and unaffected
- Windows Server 2016 Datacenter VMs preserved with licenses
- No data loss

---

## Lessons Learned

- ESXi's config persistence mechanism (`state.tgz` → `local.tgz` → `etc/shadow`) is not immediately obvious — required methodical partition-by-partition investigation
- SystemRescue is more reliable than Ubuntu Live for booting on enterprise server hardware
- Always work from `/tmp` when extracting and repacking — avoids issues with modifying files on a mounted partition directly
- The `umount` command (not `unmount`) is the correct syntax on Linux

---

## Tools Used

| Tool | Purpose |
|------|---------|
| SystemRescue | Linux live recovery environment |
| fdisk | Partition identification |
| tar | Archive extraction and repacking |
| sed | In-place shadow file modification |
| mount/umount | Partition mounting |
