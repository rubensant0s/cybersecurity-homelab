# 🛡️ Cybersecurity Homelab

> Laboratório de cibersegurança construído sobre Proxmox VE com firewall pfSense, redes isoladas e acesso remoto seguro via Tailscale.

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
| Type | Laptop repurposed as server |
| Hypervisor | Proxmox VE 8.4.0 (bare-metal) |
| Remote Access | Tailscale VPN (`100.82.62.60`) |

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
- Proxmox management accessible remotely via Tailscale

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
- **fail2ban** — active on Proxmox host, blocks brute-force SSH attempts
- **Tailscale VPN** — encrypted remote access without exposing ports
- **Network isolation** — internal LAN (vmbr1) segregated from WAN (vmbr0)
- **Lid close disabled** — server runs continuously with laptop lid closed

---

## 📁 Repository Structure

```
cybersecurity-homelab/
│
├── setup/
│   ├── proxmox-config.md        # Proxmox VE installation and config
│   ├── pfsense-config.md        # pfSense firewall setup
│   ├── network-setup.md         # Bridge and VLAN configuration
│   └── screenshots/             # Setup screenshots
│
├── writeups/
│   ├── 01-nmap-scan-metasploitable.md
│   └── 02-dvwa-web-exploitation.md
│
├── firewall-rules/
│   └── pfsense-lan-rules.md     # Documented firewall rules
│
└── README.md
```

---

## 📝 Exercises & Writeups

| # | Title | Area | Tools | Date |
|---|---|---|---|---|
| 01 | Nmap reconnaissance on Metasploitable | Pentest | Nmap | Jul 2026 |
| 02 | DVWA web exploitation | Web Security | Burp Suite, browser | Jul 2026 |

> New writeups added regularly.

---

## 🔧 Build Log — Issues & Solutions

Real problems encountered and solved during the build:

| # | Problem | Solution |
|---|---|---|
| 1 | pfSense installed with WAN only — no LAN | Added second network adapter to pfSense VM |
| 2 | Both network adapters on vmbr0 | Changed net1 to vmbr1 in Proxmox hardware settings |
| 3 | vmbr1 created but not active | Applied network configuration in Proxmox → Network → Apply Configuration |
| 4 | pfSense VM failed to start: `bridge 'vmbr1' does not exist` | Applied pending network changes — bridge was created but not applied to OS |
| 5 | Tailscale installed but `command not found` | Proxmox enterprise repos returned 401 — disabled enterprise repos, added free repo, reinstalled |
| 6 | Metasploitable `.vmdk` couldn't be uploaded via Proxmox web UI | Transferred via SCP from Mac, imported with `qm importdisk` |
| 7 | Metasploitable disk not bootable after import | Changed disk bus from VirtIO to SATA — required for Metasploitable 2 compatibility |
| 8 | Metasploitable firewall block rule not working | Rule was below "Default allow LAN" — moved block rule above allow rule |
| 9 | fail2ban failed to start: `no log file for sshd jail` | Created `/etc/fail2ban/jail.local` with correct log path |
| 10 | Server suspending when laptop lid closed | Set `HandleLidSwitch=ignore` in `/etc/systemd/logind.conf` |

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
- [x] First Nmap scan and service enumeration
- [ ] Complete 5 pentest writeups on Metasploitable
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
- [TryHackMe](https://tryhackme.com) — Guided online labs
- [HackTheBox](https://hackthebox.com) — Practice machines
- [TCM Security (YouTube)](https://www.youtube.com/@TCMSecurityAcademy) — Practical tutorials
- [VulnHub](https://www.vulnhub.com) — Downloadable vulnerable VMs

---

*Actively maintained — last updated: July 2026*
