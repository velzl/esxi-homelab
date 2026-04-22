# Lab Network Configuration

## Overview

This writeup documents the network setup used to give the ESXi homelab server internet access and connect a dedicated Kali Linux attack machine — all without a dedicated router, using only a laptop, a managed switch, and static routing.

---

## Goals

- Give the ESXi host internet access for updates and package downloads
- Access the ESXi web UI from a laptop
- Connect a Kali Linux attack machine that can reach all lab targets
- Keep all lab traffic isolated from the broader home network

---

## Final Network Topology

```
Home WiFi (192.168.1.x)
        |
        | WiFi
        |
┌───────────────────┐
│   Main Laptop     │  192.168.1.x (WiFi)
│   Windows 11      │  192.168.137.1 (Ethernet / ICS gateway)
│   ICS Enabled     │  IPEnableRouter = 1
└────────┬──────────┘
         │ Ethernet
         │
┌────────────────────┐
│  Dell PowerConnect │  Managed switch, all ports VLAN 1
│  3548 (48-port)    │
└──┬─────────────────┘
   │
   ├── Port 3: ESXi Host (192.168.137.50 static)
   │             └── Windows Server 2016 VMs
   │             └── Photon OS VM
   │
   └── (future) Port N: Dedicated wired attack machine

┌───────────────────┐
│  Attack Machine   │  Connected via home WiFi
│  Kali Linux       │  Static route: 192.168.137.0/24 via <main laptop WiFi IP>
└───────────────────┘
```

---

## Component Roles

| Device | Role | IP |
|--------|------|----|
| Main Laptop | Internet gateway / ICS host | 192.168.137.1 (eth), 192.168.1.x (WiFi) |
| Dell PowerConnect 3548 | Layer 2 switch | N/A |
| ESXi Host | Hypervisor / lab server | 192.168.137.50 |
| Kali Attack Machine | Offensive security platform | 192.168.1.x (WiFi) |

---

## Configuration Steps

### 1. Enable Internet Connection Sharing (ICS) on Main Laptop

ICS allows the laptop to share its WiFi connection over its Ethernet port, acting as a NAT gateway for the lab network.

1. Opened `ncpa.cpl` (Network Connections)
2. Right-clicked the **WiFi adapter** → Properties → **Sharing** tab
3. Checked **Allow other network users to connect through this computer's Internet connection**
4. Selected the **Ethernet adapter** in the dropdown
5. Clicked OK

Windows automatically assigns `192.168.137.1` to the Ethernet adapter and starts a lightweight DHCP server on that interface.

### 2. Configure ESXi Management Network with Static IP

The ESXi host needed a predictable IP rather than relying on DHCP.

1. Pressed **F2** at the ESXi DCUI screen
2. Navigated to **Configure Management Network → IPv4 Configuration**
3. Selected **Set static IPv4 address** and entered:
   - IP Address: `192.168.137.50`
   - Subnet Mask: `255.255.255.0`
   - Default Gateway: `192.168.137.1`
4. Pressed Enter, then Escape
5. Pressed **Y** to apply and restart the management network

ESXi web UI became accessible at `http://192.168.137.50`.

### 3. Connect Switch and Verify

Plugged both the main laptop and ESXi host into the Dell PowerConnect 3548. Verified connectivity by checking the switch's MAC address table:

```
console> show bridge address-table
```

Confirmed both MAC addresses appeared on their respective ports. Also verified port and VLAN status:

```
console> show vlan
console> show interfaces status
console> show dot1x
```

All ports were on VLAN 1 with dot1x showing **Force Authorized** — no authentication blocking traffic.

> **Troubleshooting note:** Initially the ESXi host's MAC address was not appearing in the bridge table. The cause was the Ethernet cable being plugged into the wrong NIC port on the server — ESXi's management network was bound to a different physical port than the one in use. Moving the cable to the correct port resolved the issue immediately.

### 4. Enable IP Forwarding on Main Laptop

For the wireless Kali machine to reach the `192.168.137.x` subnet, the main laptop needed to forward packets between its WiFi and Ethernet interfaces.

Opened CMD as Administrator and ran:

```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters /v IPEnableRouter /t REG_DWORD /d 1 /f
```

Rebooted the laptop to apply the change, then re-enabled ICS.

To disable IP forwarding when not in use:

```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters /v IPEnableRouter /t REG_DWORD /d 0 /f
```

### 5. Add Static Route on Kali Attack Machine

The Kali machine connected to home WiFi had no knowledge of the `192.168.137.x` lab subnet. Added a persistent static route pointing to the main laptop's WiFi IP as the next hop:

```bash
sudo ip route add 192.168.137.0/24 via <main laptop WiFi IP>
```

Verified the route was added:

```bash
ip route show
```

Tested connectivity:

```bash
ping 192.168.137.50
nmap -sV 192.168.137.50
```

Both succeeded — the Kali machine could now reach the ESXi host and all VMs on the lab network.

---

## Security Considerations

- The `192.168.137.x` subnet is not advertised or reachable from the internet
- IP forwarding on the main laptop means other devices on the home WiFi could theoretically reach the lab subnet if they knew the route — acceptable for a home lab environment
- IP forwarding can be disabled when the lab is not in use
- All offensive testing stays within the isolated `192.168.137.x` network

---

## Troubleshooting Notes

| Issue | Cause | Fix |
|-------|-------|-----|
| ESXi unreachable after adding switch | Cable plugged into wrong NIC on server | Moved cable to NIC bound to ESXi management network |
| Ethernet showing "Unidentified Network" | ICS dropped when physical connection changed | Disabled and re-enabled ICS on WiFi adapter |
| IP forwarding not working after registry edit | Requires reboot to take effect on Windows 11 | Rebooted laptop |
| Kali unable to ping lab subnet | No route to 192.168.137.0/24 | Added static route via `ip route add` |

---

## Lessons Learned

- Windows ICS is a quick and effective way to share internet to a lab network without a dedicated router
- Managed switches require verification of port/VLAN config even when everything looks default — always check with `show` commands
- IP forwarding on Windows requires a reboot — the registry key alone is not enough
- A wired connection for the attack machine would eliminate the need for static routes entirely and is the cleaner long-term solution

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Windows ICS | Internet sharing / NAT gateway |
| ncpa.cpl | Windows network adapter configuration |
| ESXi DCUI | Server-side network configuration |
| Dell PowerConnect CLI | Switch verification and troubleshooting |
| ip route | Static route management on Kali Linux |
| nmap | Connectivity and service verification |
