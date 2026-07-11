# 🛡️ Cybersecurity Homelab

> Cybersecurity lab built on Proxmox VE with pfSense firewall, isolated networks and secure remote access via Tailscale.

---

## 📌 About

This repository documents the build and evolution of my personal cybersecurity homelab. The goal is to develop hands-on skills in penetration testing, network security, and defensive operations in a fully isolated and controlled environment.

**Status:** 🟢 Active  
**Started:** July 2026  
**Level:** Intermediate

---

## 🖥️ Infrastructure

### Hardware
| Component | Specification |
|---|---|
| Type | Laptop repurposed as dedicated server |
| Hypervisor | Proxmox VE 8.4.0 (bare-metal, type-1) |
| Remote Access | Tailscale VPN |

### Network Architecture

```
Internet
    │
  [vmbr0] — WAN bridge
    │
 [pfSense VM] ← Firewall / Router / DHCP
    │
  [vmbr1] — LAN bridge (192.168.100.0/24)
    ├── Kali Linux       → 192.168.100.x (attacker)
    ├── Metasploitable 2 → 192.168.100.5 (target)
    └── Windows 11       → 192.168.100.x (victim/analyst)
```

**Design decisions:**
- pfSense controls all traffic between WAN and LAN
- Metasploitable 2 is firewall-blocked from internet access (pfSense rule)
- All internal VMs communicate only through vmbr1
- Proxmox management accessible remotely via Tailscale VPN

---

## 🧰 Virtual Machines

| VM ID | Name | OS | Role | Network |
|---|---|---|---|---|
| 100 | Kali | Kali Linux 2024.x | Attack machine | vmbr1 |
| 101 | Windows11 | Windows 11 | Victim / Blue Team | vmbr1 |
| 102 | pfSense | pfSense 2.8.1 | Firewall / Router | vmbr0 + vmbr1 |
| 103 | Metasploitable | Metasploitable 2 | Vulnerable target | vmbr1 |

---

## 🔒 Security Controls

- **pfSense firewall** — blocks Metasploitable from internet access
- **fail2ban** — active on Proxmox host, auto-bans brute-force SSH attempts
- **Tailscale VPN** — encrypted remote access without exposing ports
- **Network isolation** — internal LAN (vmbr1) fully segregated from WAN (vmbr0)
- **Lid close disabled** — server runs continuously with laptop lid closed

---

## 📝 Writeups & Exercises

| # | Title | Area | Tools | Key Finding |
|---|---|---|---|---|
| [01](writeups/01-nmap-scan-metasploitable.md) | Network Reconnaissance with Nmap | Recon | Nmap | 23 open ports, 4 critical services identified |
| [02](writeups/02-dvwa-sql-injection.md) | SQL Injection — DVWA | Web Security | Burp Suite, browser | Full user table extracted + passwords cracked |
| [03](writeups/03-vsftpd-backdoor-exploitation.md) | vsftpd 2.3.4 Backdoor (CVE-2011-2523) | Exploitation | Metasploit | Root shell in < 10 seconds |
| [04](writeups/04-telnet-credential-capture.md) | Telnet Credential Capture | Network Security | Wireshark | Credentials captured in plaintext via passive sniffing |
| [05](writeups/05-mysql-unauthenticated-access.md) | MySQL Unauthenticated Access | Database Security | mysql client | Root DB access with no password — 7 databases exposed |

> All writeups include: methodology, screenshots, severity-rated findings table, and remediation recommendations.

---

## 📁 Repository Structure

```
cybersecurity-homelab/
│
├── writeups/
│   ├── screenshots/               # Evidence screenshots for all writeups
│   ├── 01-nmap-scan-metasploitable.md
│   ├── 02-dvwa-sql-injection.md
│   ├── 03-vsftpd-backdoor-exploitation.md
│   ├── 04-telnet-credential-capture.md
│   └── 05-mysql-unauthenticated-access.md
│
└── README.md
```

---

## 🔧 Build Log — Issues & Solutions

Real problems encountered and solved during the build:

| # | Problem | Solution |
|---|---|---|
| 1 | pfSense installed with WAN only — no LAN | Added second network adapter to pfSense VM |
| 2 | Both network adapters on vmbr0 | Changed net1 to vmbr1 in Proxmox hardware settings |
| 3 | vmbr1 created but not active | Applied network config in Proxmox → Network → Apply Configuration |
| 4 | pfSense VM failed to start: `bridge 'vmbr1' does not exist` | Applied pending network changes — bridge was created but not applied to OS |
| 5 | Tailscale: `command not found` after install | Proxmox enterprise repos returned 401 — disabled enterprise repos, added free repo, reinstalled |
| 6 | Metasploitable `.vmdk` couldn't upload via Proxmox web UI | Transferred via SCP from Mac, imported with `qm importdisk` |
| 7 | Metasploitable disk not bootable after import | Changed disk bus from VirtIO to SATA — required for Metasploitable 2 compatibility |
| 8 | Metasploitable firewall block rule not working | Rule was below "Default allow LAN" — moved block rule above allow rule |
| 9 | fail2ban failed to start: `no log file for sshd jail` | Created `/etc/fail2ban/jail.local` with correct log path |
| 10 | Server suspending when laptop lid closed | Set `HandleLidSwitch=ignore` in `/etc/systemd/logind.conf` |
| 11 | MySQL connection error: `TLS/SSL wrong version number` | Modern client vs ancient MySQL 5.0 — bypassed with `--skip-ssl` flag |

---

## 🎯 Roadmap

- [x] Install Proxmox VE 8.4.0
- [x] Configure pfSense with WAN + LAN
- [x] Set up isolated internal network (vmbr1)
- [x] Deploy Kali Linux (attack machine)
- [x] Deploy Metasploitable 2 (vulnerable target)
- [x] Deploy Windows 11 (victim machine)
- [x] Configure remote access via Tailscale
- [x] Block Metasploitable from internet via pfSense rules
- [x] Install and configure fail2ban
- [x] Writeup 01 — Nmap reconnaissance (23 services, 10 findings)
- [x] Writeup 02 — SQL Injection on DVWA (full credential extraction)
- [x] Writeup 03 — vsftpd 2.3.4 backdoor exploitation (root shell)
- [x] Writeup 04 — Telnet credential capture with Wireshark
- [x] Writeup 05 — MySQL unauthenticated access (7 databases)
- [ ] Writeup 06 — UnrealIRCd backdoor (CVE-2010-2075)
- [ ] Writeup 07 — Port 1524 root shell (no exploit)
- [ ] Configure Wazuh SIEM for Blue Team monitoring
- [ ] Set up WireGuard VPN via pfSense
- [ ] Obtain eJPT certification

---

## ⚖️ Legal Disclaimer

> **All exercises and tests documented in this repository were performed exclusively in isolated virtual environments created for educational purposes.**
>
> No activity was performed on third-party systems without authorization. The author is not responsible for misuse of any information published here.

---

## 📚 Resources

- [Proxmox VE Documentation](https://pve.proxmox.com/wiki/Main_Page)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [TryHackMe](https://tryhackme.com)
- [HackTheBox](https://hackthebox.com)
- [TCM Security (YouTube)](https://www.youtube.com/@TCMSecurityAcademy)
- [VulnHub](https://www.vulnhub.com)

---

*Actively maintained — last updated: July 2026*
