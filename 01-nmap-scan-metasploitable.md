# Writeup 01 — Network Reconnaissance with Nmap

**Target:** Metasploitable 2  
**IP:** 192.168.100.5  
**Date:** July 2026  
**Author:** Rúben Silva  

---

## Objective

Perform passive network reconnaissance on the Metasploitable 2 target using Nmap to enumerate open ports, running services, and software versions. Document all findings and identify potential attack vectors for further exploitation.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port scanning and service enumeration |
| Kali Linux | Attack machine |

---

## 1. Environment

| Component | Details |
|---|---|
| Attack machine | Kali Linux (192.168.100.x) |
| Target machine | Metasploitable 2 (192.168.100.5) |
| Network | Isolated LAN — vmbr1 (192.168.100.0/24) |
| Hypervisor | Proxmox VE 8.4.0 |
| Firewall | pfSense — Metasploitable blocked from internet |

---

## 2. Methodology

Reconnaissance was conducted in two phases:

**Phase 1 — Quick scan:** Basic port discovery to identify open TCP ports.

**Phase 2 — Service scan:** Version detection (`-sV`) to enumerate services and software versions running on each open port.

---

## 3. Scan Execution

### Phase 1 — Port Discovery

```bash
nmap 192.168.100.5
```

### Phase 2 — Service & Version Detection

```bash
nmap -sV 192.168.100.5 -oN 01-nmap-scan-metasploitable.txt
```

**Flags used:**
- `-sV` — probe open ports to determine service and version info
- `-oN` — save output to file in normal format

**Scan duration:** 52.97 seconds  
**Result:** 1 host up, 977 closed TCP ports, **23 open ports identified**

---

## 4. Results

### 4.1 Open Ports & Services

| Port | State | Service | Version |
|---|---|---|---|
| 21/tcp | open | ftp | vsftpd 2.3.4 |
| 22/tcp | open | ssh | OpenSSH 4.7p1 |
| 23/tcp | open | telnet | Linux telnetd |
| 25/tcp | open | smtp | Postfix smtpd |
| 53/tcp | open | domain | ISC BIND 9.4.2 |
| 80/tcp | open | http | Apache httpd 2.2.8 |
| 111/tcp | open | rpcbind | 2 (RPC #100000) |
| 139/tcp | open | netbios-ssn | Samba smbd 3.x |
| 445/tcp | open | microsoft-ds | Samba smbd 3.x |
| 512/tcp | open | exec | netkit-rsh rexecd |
| 513/tcp | open | login | OpenBSD or Solaris rlogind |
| 514/tcp | open | shell | Netkit rshd |
| 1099/tcp | open | java-rmi | GNU Classpath grmiregistry |
| 1524/tcp | open | ingreslock | Metasploitable root shell |
| 2049/tcp | open | nfs | 2-4 (RPC #100003) |
| 2121/tcp | open | ftp | ProFTPD 1.3.1 |
| 3306/tcp | open | mysql | MySQL 5.0.51a-3ubuntu5 |
| 5432/tcp | open | postgresql | PostgreSQL DB 8.3.0-8.3.7 |
| 5900/tcp | open | vnc | VNC (protocol 3.3) |
| 6000/tcp | open | X11 | access denied |
| 6667/tcp | open | irc | UnrealIRCd |
| 8009/tcp | open | ajp13 | Apache Jserv (Protocol v1.3) |
| 8180/tcp | open | http | Apache Tomcat/Coyote JSP engine 1.1 |

### 4.2 Host Information

```
MAC Address: BC:24:11:8B:44:D1 (Proxmox Server Solutions GmbH)
Service Info: Hosts: metasploitable.localdomain, irc.Metasploitable.LAN
OS: Linux
CPE: cpe:/o:linux:linux_kernel
```

---

## 5. Analysis — High Value Targets

### 🔴 Critical — Known Exploitable Services

**vsftpd 2.3.4 (Port 21)**
This specific version contains a deliberate backdoor introduced via a compromised source code package. Connecting to port 21 and sending a username containing `:)` triggers a bind shell on port 6200, granting root access without authentication.
- CVE: CVE-2011-2523
- Exploit available in Metasploit: `exploit/unix/ftp/vsftpd_234_backdoor`

**UnrealIRCd (Port 6667)**
The version running on this host contains a backdoor in the DEBUG3 command that allows remote code execution as the user running the IRC daemon.
- CVE: CVE-2010-2075
- Exploit available in Metasploit: `exploit/unix/irc/unreal_ircd_3281_backdoor`

**Metasploitable Root Shell (Port 1524)**
Port 1524 is a pre-configured backdoor root shell — connecting directly grants an interactive root shell with no authentication required.

### 🟠 High — Weak or Unauthenticated Services

**Telnet (Port 23)**
Telnet transmits all data including credentials in plaintext. Any network observer can capture usernames and passwords in transit. This protocol should never be used over any network.

**VNC (Port 5900)**
VNC protocol 3.3 with no authentication — direct graphical access to the desktop without credentials.

**MySQL (Port 3306)**
MySQL 5.0.51a accessible on the network. Default installations of this version often have no root password set.

**NFS (Port 2049)**
Network File System exposed — may allow mounting remote filesystems without authentication depending on export configuration.

**rexec / rlogin / rsh (Ports 512-514)**
Legacy remote access services with weak or no authentication. These services trust connections based on IP address rather than cryptographic credentials.

### 🟡 Medium — Information Exposure

**Apache HTTP (Port 80)**
Hosts DVWA and other vulnerable web applications. Version 2.2.8 is outdated and may have known vulnerabilities.

**Apache Tomcat (Port 8180)**
Default Tomcat installation — likely accessible with default credentials (`tomcat:tomcat`).

**PostgreSQL (Port 5432)**
Database service exposed on the network — may be accessible with default credentials.

**DNS (Port 53)**
ISC BIND 9.4.2 — outdated version, may be vulnerable to zone transfer attacks.

---

## 6. Attack Surface Summary

```
Metasploitable 2 — 192.168.100.5
│
├── CRITICAL (immediate root access)
│   ├── Port 21  — vsftpd 2.3.4 backdoor (CVE-2011-2523)
│   ├── Port 1524 — pre-configured root shell
│   └── Port 6667 — UnrealIRCd backdoor (CVE-2010-2075)
│
├── HIGH (significant risk)
│   ├── Port 23  — Telnet (plaintext credentials)
│   ├── Port 512-514 — rexec/rlogin/rsh (legacy, weak auth)
│   ├── Port 5900 — VNC (no authentication)
│   └── Port 3306 — MySQL (possible no-auth access)
│
└── MEDIUM (requires further testing)
    ├── Port 80   — Apache + DVWA (web vulnerabilities)
    ├── Port 8180 — Tomcat (default credentials)
    ├── Port 2049 — NFS (possible unauthenticated mount)
    └── Port 53   — BIND 9.4.2 (zone transfer)
```

---

## 7. Severity Assessment

| # | Finding | Severity | Notes |
|---|---|---|---|
| F1 | vsftpd 2.3.4 backdoor | 🔴 Critical | Direct root shell via CVE-2011-2523 |
| F2 | Pre-configured root shell on port 1524 | 🔴 Critical | No auth required |
| F3 | UnrealIRCd backdoor | 🔴 Critical | RCE via CVE-2010-2075 |
| F4 | VNC without authentication | 🔴 Critical | Direct graphical access |
| F5 | Telnet in use | 🟠 High | Credentials transmitted in plaintext |
| F6 | Legacy rexec/rlogin/rsh services | 🟠 High | No cryptographic authentication |
| F7 | MySQL exposed on network | 🟠 High | Possible no-password root access |
| F8 | Outdated software versions across all services | 🟠 High | Multiple known CVEs |
| F9 | NFS exposed | 🟡 Medium | Possible unauthenticated filesystem mount |
| F10 | Apache Tomcat default credentials | 🟡 Medium | Default tomcat:tomcat may be active |

---

## 8. Recommendations

**R1 — Disable all unnecessary services:**
A system should only run services required for its function. Telnet, rexec, rlogin, rsh, VNC, and the IRC server should be disabled immediately.

**R2 — Replace Telnet with SSH:**
All remote access should use SSH with key-based authentication. Telnet must never be used on any network.

**R3 — Update all software:**
vsftpd 2.3.4, UnrealIRCd, Apache 2.2.8, BIND 9.4.2 and all other services are severely outdated. Patch to current supported versions.

**R4 — Restrict database access:**
MySQL and PostgreSQL should not be accessible from the network. Bind database services to localhost only (`bind-address = 127.0.0.1`) and enforce strong authentication.

**R5 — Configure NFS exports securely:**
Restrict NFS exports to specific trusted IP addresses and require authentication.

**R6 — Implement network segmentation:**
Services should be segmented by function. Database servers should not be directly accessible from the same network as web servers or end users.

---

## 9. Next Steps

Based on this reconnaissance, the following exploitation exercises are planned:

- [ ] Exploit vsftpd 2.3.4 backdoor via Metasploit
- [ ] Connect to root shell on port 1524
- [ ] Exploit UnrealIRCd backdoor
- [ ] Capture Telnet credentials with Wireshark
- [ ] Access MySQL without password
- [ ] Test NFS for unauthenticated mount

---

## 10. References

- [CVE-2011-2523 — vsftpd 2.3.4 Backdoor](https://nvd.nist.gov/vuln/detail/CVE-2011-2523)
- [CVE-2010-2075 — UnrealIRCd Backdoor](https://nvd.nist.gov/vuln/detail/CVE-2010-2075)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [OWASP Attack Surface Analysis](https://owasp.org/www-community/Attack_Surface_Analysis_Cheat_Sheet)

---

> **Legal disclaimer:** This reconnaissance was performed exclusively on an intentionally vulnerable virtual machine (Metasploitable 2) within a private isolated lab network. The target has no internet access and no real systems were affected.
