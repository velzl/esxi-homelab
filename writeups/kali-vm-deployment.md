# Kali Linux VM Deployment on ESXi 6.5

## Overview

Deployed a Kali Linux VM directly on the ESXi host as a dedicated internal attack machine. The process required VMDK format conversion and manual VMX configuration due to compatibility issues between the modern Kali VMware image and ESXi 6.5.

---

## Environment

| Item | Details |
|------|---------|
| Hypervisor | VMware ESXi 6.5.0 Build 9298722 |
| Kali Image | Kali Linux 2026.1 VMware AMD64 |
| Transfer Tool | WinSCP |
| VM Config | 4 vCPU, 8GB RAM, 80GB disk |

---

## Issues Encountered

The modern Kali VMware image ships with hardware incompatibilities for ESXi 6.5:

- **Disk type 7** — newer VMDK format not supported by ESXi 6.5
- **VM hardware version 19+** — causes ESXi 6.5 web UI to crash with a `TypeError` when trying to edit VM settings
- **Missing network portgroup** — Kali's default VMX references a portgroup that doesn't exist on the ESXi host
- **Sound device warning** — ESXi servers have no sound hardware, causes a non-fatal warning on boot

All of these were resolved without reinstalling or downloading an older image.

---

## Steps

### 1. Download Kali VMware Image

Downloaded the official Kali Linux VMware image from kali.org. The VMware download comes as a `.7z` archive containing:
- Multiple `.vmdk` files (split disk)
- One `.vmx` configuration file

> Note: The VirtualBox image (`.vdi`) is not compatible with VMware ESXi. Make sure to download the VMware specific image.

### 2. Enable SSH on ESXi Host

The ESXi datastore browser only supports single file uploads. SSH was enabled to allow bulk file transfer via WinSCP:

1. ESXi web UI → **Host → Actions → Services → Enable Secure Shell (SSH)**

### 3. Transfer Files via WinSCP

Connected to the ESXi host via WinSCP:
- Protocol: **SCP**
- Host: `192.168.137.50`
- Username: `vpxuser`

> Note: Root SSH login was restricted on this ESXi host. `vpxuser` had sufficient datastore access for file transfer.

Created a dedicated folder on the datastore and uploaded all files at once by selecting all and dragging into WinSCP:

```
/vmfs/volumes/datastore1/kali-linux-2026.1-vmware-amd64.vmwarevm/
```

### 4. Register the VM

1. ESXi web UI → **Virtual Machines → Create/Register VM**
2. Selected **Register an existing virtual machine**
3. Browsed to the Kali folder and selected the `.vmx` file
4. Completed the wizard — VM appeared in the VM list

### 5. Convert VMDK Format

Attempting to power on the VM produced:

```
Unsupported or invalid disk type 7 for 'scsi0:0'. Ensure that the disk has been imported.
```

ESXi 6.5 does not support disk type 7 (the newer VMDK format). Converted the disk via SSH on the ESXi host:

```bash
vmkfstools -i /vmfs/volumes/datastore1/kali-linux-2026.1-vmware-amd64.vmwarevm/kali-linux-2026.1-vmware-amd64.vmdk \
           /vmfs/volumes/datastore1/kali-linux-2026.1-vmware-amd64.vmwarevm/kali-converted.vmdk \
           -d thin
```

This created a new ESXi 6.5 compatible VMDK (`kali-converted.vmdk`) and a corresponding flat file (`kali-converted-flat.vmdk`, ~80GB).

### 6. Edit VMX Configuration

The ESXi 6.5 web UI crashes when trying to edit VMs with hardware version 19+:

```
TypeError: Cannot read properties of undefined (reading 'numSupportedFloppyDevices')
```

All VM configuration was done by directly editing the `.vmx` file in WinSCP.

**Changed disk pointer to converted VMDK:**
```
scsi0:0.fileName = "kali-converted.vmdk"
```

**Set CPU and RAM:**
```
numvcpus = "4"
cpuid.coresPerSocket = "4"
memsize = "8192"
```

**Fixed network configuration** (original VMX was missing `networkName` and using NAT):
```
ethernet0.networkName = "VM Network"
ethernet0.connectionType = "bridged"
```

**Lowered hardware version** for better ESXi 6.5 compatibility:
```
virtualHW.version = "11"
```

### 7. Power On and Verify

Powered on the VM. A warning appeared about the sound device — clicked **No** to skip reconnecting it on every boot (servers have no sound hardware).

VM booted successfully into Kali Linux. Default credentials:
- Username: `kali`
- Password: `kali`

Verified network connectivity from inside the VM:

```bash
ip a
ping 192.168.137.50
```

VM was reachable on the `192.168.137.x` lab network and could reach the ESXi host and other VMs.

---

## Result

- Kali Linux VM running on ESXi 6.5 as a dedicated internal attack machine
- Reachable at a static IP on the `192.168.137.x` lab subnet
- 4 vCPU, 8GB RAM, 80GB disk — sufficient for all standard offensive security tooling
- All compatibility issues resolved without downloading an older Kali image

---

## Lessons Learned

- Modern Kali VMware images use VMDK disk type 7 which is incompatible with ESXi 6.5 — `vmkfstools` conversion is the fix
- ESXi 6.5 web UI crashes on VMs with hardware version 19+ — direct VMX editing is the workaround
- `vpxuser` can be used for datastore file transfers when root SSH is restricted
- Always verify the `networkName` in the VMX matches an existing portgroup on the ESXi host — missing portgroup silently prevents network connectivity
- Sound device warnings on ESXi are harmless — always click No to avoid reconnection attempts on every boot

---

## Tools Used

| Tool | Purpose |
|------|---------|
| WinSCP | Bulk file transfer to ESXi datastore via SCP |
| vmkfstools | VMDK format conversion |
| VMX editor (WinSCP) | Manual VM configuration |
| ESXi Host Client | VM registration and power management |
