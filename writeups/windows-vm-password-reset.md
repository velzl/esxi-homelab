# Windows Server VM Credential Recovery

## Overview

Multiple Windows Server 2016 VMs were running on the ESXi host with unknown Administrator passwords. The goal was to recover access to each VM without reinstalling Windows or losing any data, using only tools available through the ESXi hypervisor.

---

## Environment

| Item | Details |
|------|---------|
| Hypervisor | VMware ESXi 6.5.0 |
| Guest OS | Windows Server 2016 (Datacenter) |
| Recovery Tool | Hiren's BootCD PE |
| Password Reset Utility | NTPWEdit (included in Hiren's) |

---

## Approach

Rather than booting a physical USB drive, the ISO was uploaded directly to the ESXi datastore and mounted as a virtual CD drive — no physical media required. This is the cleanest method when you already have access to the ESXi host.

---

## Steps

### 1. Download Hiren's BootCD PE

Downloaded the Hiren's BootCD PE ISO from hirensbootcd.org. This is a Windows PE based recovery environment that includes NTPWEdit for offline Windows password reset.

### 2. Upload ISO to ESXi Datastore

1. Logged into ESXi Host Client at `http://192.168.137.50`
2. Navigated to **Storage → Datastore Browser**
3. Clicked **Upload** and selected the Hiren's ISO file
4. Waited for the upload to complete

### 3. Mount ISO to Target VM

1. Selected the target VM in the ESXi Host Client
2. Clicked **Edit Settings**
3. Located the **CD/DVD Drive** device
4. Changed the type to **Datastore ISO file**
5. Browsed to and selected the uploaded Hiren's ISO
6. Checked **Connect** to ensure it would be active on next boot

### 4. Modify VM Boot Order

The VM BIOS needed to be configured to boot from CD/DVD before the hard drive:

1. With the VM powered off, clicked **Edit Settings → VM Options → Boot Options**
2. Enabled **Force BIOS setup on next boot**
3. Powered on the VM and opened the console
4. Inside the VM BIOS, navigated to the **Boot** tab
5. Moved **CD-ROM Drive** to the top of the boot order
6. Pressed **F10** to save and exit

### 5. Boot into Hiren's BootCD PE

The VM booted into the Hiren's BootCD PE environment — a full Windows PE desktop loaded from the ISO.

### 6. Reset Administrator Password with NTPWEdit

1. Opened the **Start menu → Security → NTPWEdit**
2. Clicked **Open** — NTPWEdit automatically located the Windows SAM database on the local drive
3. Selected the **Administrator** account from the user list
4. Clicked **Change Password**
5. Entered a new password (note: blank passwords are blocked by Windows Server 2016 password policy — a password meeting complexity requirements must be set)
6. Clicked **OK**
7. Clicked **Save Changes**

> **Note:** Windows Server 2016 enforces password complexity by default. A blank password will fail silently — NTPWEdit will appear to succeed but Windows will reject the empty hash on login. Always set a real password (e.g., `Password1!`) and change it after logging in.

### 7. Restore Normal Boot Order

Before rebooting into Windows:

1. Went back into VM BIOS (or Edit Settings → Boot Options)
2. Moved the hard drive back to the top of the boot order
3. Disconnected the CD/DVD ISO (Edit Settings → CD/DVD → uncheck Connect)

### 8. Log In

Rebooted the VM. At the Windows Server login screen, selected **Administrator** and entered the newly set password — access granted.

Repeated this process for each VM on the host.

---

## Result

- Administrator access restored on all Windows Server 2016 VMs
- No data, applications, or settings affected
- Windows licenses and activation status preserved
- Process completed entirely through the ESXi web UI — no physical USB required

---

## Lessons Learned

- Windows Server 2016 password complexity policy blocks blank passwords even when set offline via NTPWEdit — always set a real password
- Uploading the ISO to the datastore is cleaner and faster than dealing with physical USB passthrough in VMware
- Always restore the boot order after recovery — leaving CD-ROM first causes boot loops on restart
- The ESXi Host Client's built-in console is sufficient for BIOS-level VM interaction, no additional tools needed

---

## Tools Used

| Tool | Purpose |
|------|---------|
| ESXi Host Client | Datastore upload, VM settings, console access |
| Hiren's BootCD PE | Windows PE recovery environment |
| NTPWEdit | Offline Windows SAM password reset |
