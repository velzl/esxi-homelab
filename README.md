# VMware ESXi Offensive Security Homelab

A self-built enterprise homelab deployed on an HP ProLiant DL360p Gen8 server, configured from scratch for offensive security research, penetration testing practice, and portfolio development.

---

## Hardware

| Component | Details |
|-----------|---------|
| Server | HP ProLiant DL360p Gen8 |
| CPUs | 2x Intel Xeon E5-2670 (16 cores / 32 threads) |
| RAM | 48GB DDR3 ECC |
| Storage | 8x 300GB SAS (RAID configuration, ~1.1TB usable) |
| Switch | Dell PowerConnect 3548 (managed, 48-port) |
| Hypervisor | VMware ESXi 6.5.0 Build 9298722 |

---

## Network Topology

```
Home WiFi
    |
    | (WiFi)
    |
Main Laptop (Windows 11)
    | ICS / IP Forwarding enabled
    | (Ethernet - 192.168.137.1)
    |
Dell PowerConnect 3548 (Switch)
    |
    |-------- ESXi Host (192.168.137.50)
    |              |
    |              |--- Windows Server 2016 VMs
    |              |--- Photon OS VM
    |              |--- (future targets)
    |
Attack Machine (Kali Linux)
    | (WiFi/Ethernet + static route via main laptop)
    | ip route add 192.168.137.0/24 via <main laptop WiFi IP>
    | (For an ethernet connection (used via switch) use sudo ip addr add 192.168.137.100/24 [static ip for attack machine on the /24 subnet] and the previous ip route command 
```

---

## What I Built

### 1. ESXi Deployment & Recovery
The server was acquired secondhand with ESXi 6.5 already installed but no known root credentials. Rather than wiping the drives (which would have destroyed existing VMs and Windows Server licenses), I performed a manual password reset:

- Booted SystemRescue Linux from USB
- Identified ESXi partition layout using `fdisk -l`
- Located `state.tgz` on the ESXi boot partition (sda5)
- Extracted `state.tgz` → `local.tgz` → `etc/shadow`
- Cleared password hashes for root, admin, and vpxuser using `sed`
- Repacked the archive chain and wrote it back to the partition
- Rebooted into ESXi with blank root password

All existing VMs and data were preserved throughout.

### 2. Network Configuration
Configured a segmented lab network without a dedicated router:

- Enabled Windows Internet Connection Sharing (ICS) on the main laptop to share WiFi over Ethernet
- Configured ESXi management network with a static IP (`192.168.137.50`)
- Set up a Dell PowerConnect 3548 managed switch, verified VLAN and port configuration via CLI
- Enabled IP forwarding on the Windows host (`IPEnableRouter`) and added static routes on the Kali attack machine to reach the `192.168.137.0/24` subnet wirelessly
- Used a wired connection for the kali attack machine wired in via switch setting up the static ip of ('192.168.137.100') to ensure the attack machine can access the server

### 3. Windows Server VM Access Recovery
Multiple Windows Server 2016 VMs existed with unknown Administrator passwords:

- Uploaded Hiren's BootCD PE ISO directly to the ESXi datastore via the web UI
- Mounted the ISO as a virtual CD drive on each VM
- Modified VM BIOS boot order to prioritize CD-ROM
- Used NTPWEdit within Hiren's to reset Administrator credentials
- Restored normal boot order post-recovery

### 4. Lab Network for Offensive Security Practice
- Kali Linux deployed as dedicated attack machine
- Static routing configured to reach all lab targets wirelessly
- ESXi web UI accessible from both main laptop and attack machine
- VMs snapshotted before any offensive testing to allow rollback

---

## Skills Demonstrated

- VMware ESXi administration and troubleshooting
- Linux filesystem navigation and system recovery
- Managed switch configuration (VLANs, port status, dot1x)
- Windows networking (ICS, IP forwarding, static routes)
- Offline credential recovery (Linux shadow file manipulation, NTPWEdit)
- Enterprise hardware deployment and management
- Lab documentation and network diagramming

---

## Planned Next Steps

- [ ] Deploy Kali Linux VM as internal attack machine
- [ ] Set up Active Directory on Windows Server 2016 for AD attack practice
- [ ] Deploy pfSense VM to replace laptop-based routing
- [ ] Add Metasploitable / intentionally vulnerable VMs as targets
- [ ] Configure Wazuh SIEM for log monitoring and detection practice
- [ ] Document penetration testing exercises with full methodology writeups
- [ ] Set up VLANs on the managed switch for network segmentation practice

---

## Writeups

- [ESXi Password Reset via Linux Live Boot](writeups/esxi-password-reset.md)
- [Windows Server VM Credential Recovery](writeups/windows-vm-password-reset.md)
- [Lab Network Configuration](writeups/network-config.md)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| SystemRescue | Linux live environment for ESXi recovery |
| Rufus | Bootable USB creation |
| Hiren's BootCD PE | Windows VM credential recovery |
| NTPWEdit | Offline Windows password reset |
| VMware ESXi Host Client | Hypervisor web UI |
| Dell PowerConnect CLI | Switch configuration |
| Kali Linux | Offensive security attack platform |
| Nmap | Network reconnaissance |
