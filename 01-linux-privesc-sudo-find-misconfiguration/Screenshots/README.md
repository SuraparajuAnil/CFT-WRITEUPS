# Linux Privilege Escalation via Misconfigured Sudo Permissions

**Category:** Linux / Privilege Escalation
**Environment:** Authorized, isolated training-lab virtual machine (VirtualBox)
**Target OS:** Ubuntu 14.04.5 LTS
**Status:** Completed — user and root access obtained

> Machine name, and lab IP address have been redacted per the terms under which this write-up is published. Credential values and flag values are redacted; the write-up describes the method used to obtain them rather than the literal values.

---

## Summary

This write-up documents a black-box assessment of a Linux target in an isolated training-lab network. The engagement involved network reconnaissance, enumeration of exposed services, a credential attack against SSH, and privilege escalation via a misconfigured `sudo` permission on the `find` binary, resulting in full root access.

## Skills Demonstrated

- Network scanning and service enumeration with Nmap
- Web service reconnaissance
- Dictionary-based credential attacks with Hydra
- Linux privilege escalation analysis using `sudo -l`
- Exploiting sudo misconfigurations via GTFOBins
- Root-cause remediation reasoning

---

## 1. Reconnaissance

A full-range Nmap scan was run against the lab subnet to identify the live target and its exposed services.

![image](https://github.com/SuraparajuAnil/CFT-WRITEUPS/blob/main/01-linux-privesc-sudo-find-misconfiguration/Screenshots/nmap-scan.png)

**Findings:**
- Two services exposed: SSH (22) and HTTP (80).
- Host responded to ICMP and had no firewall filtering evident.

## 2. Web Enumeration

The HTTP service on port 80 was accessed via browser. The page displayed a taunting message along with an encoded hint (NATO phonetic alphabet spelling out a username):

![image](https://github.com/SuraparajuAnil/CFT-WRITEUPS/blob/main/01-linux-privesc-sudo-find-misconfiguration/Screenshots/web-enumeration.png)

Decoding the NATO phonetic string (`BRAVO-ECHO-LIMA-LIMA` → **B-E-L-L**) revealed a candidate SSH username.

## 3. Exploitation — SSH Credential Attack

With a candidate username identified, a dictionary-based password attack was performed against the SSH service using Hydra and the `rockyou.txt` wordlist.

![image](https://github.com/SuraparajuAnil/CFT-WRITEUPS/blob/main/01-linux-privesc-sudo-find-misconfiguration/Screenshots/ssh-credential-attack.png)

A valid credential pair was recovered, and SSH access was obtained.

![image](https://github.com/SuraparajuAnil/CFT-WRITEUPS/blob/main/01-linux-privesc-sudo-find-misconfiguration/Screenshots/ssh-user-flag.png)

**Result:** Initial foothold (low-privileged shell) obtained; user-level flag captured.

## 4. Privilege Escalation

With low-privileged access established, sudo permissions for the current user were enumerated:

![image](https://github.com/SuraparajuAnil/CFT-WRITEUPS/blob/main/01-linux-privesc-sudo-find-misconfiguration/Screenshots/prev-esc.png)

The user was permitted to run `/usr/bin/find` as root with no password required — a well-known privilege escalation vector documented on [GTFOBins](https://gtfobins.github.io/gtfobins/find/).

`find` supports the `-exec` flag, which can be abused to spawn an arbitrary command (in this case, a root shell):

![image](https://github.com/SuraparajuAnil/CFT-WRITEUPS/blob/main/01-linux-privesc-sudo-find-misconfiguration/Screenshots/root-flag.png)

**Result:** Full root access obtained; root-level flag captured.

---

## Impact

An attacker with low-privileged shell access (obtained here via a weak SSH credential) could escalate to full root/administrative control of the host due to an overly permissive `sudo` rule. This represents a **complete compromise** of the affected system, with implications for confidentiality, integrity, and availability of any data or services hosted on it.

## Root Cause

- A weak, dictionary-guessable password on a user account exposed via SSH.
- An unrestricted `sudo` entitlement granted to a general-purpose utility (`find`) capable of arbitrary command execution.

## Remediation Recommendations

| Issue | Recommendation |
|---|---|
| Weak SSH credential | Enforce strong password policies and/or move to key-based SSH authentication. Consider rate-limiting or fail2ban to slow credential attacks. |
| Unrestricted `sudo` access to `find` | Remove blanket sudo access to `find`. If elevated access to specific file operations is genuinely required, replace it with a narrowly scoped wrapper script that only permits the specific intended operation, or use filesystem permissions instead of sudo. |
| General hardening | Regularly audit `sudo -l` output for all users against [GTFOBins](https://gtfobins.github.io/) to identify similar privilege escalation paths before they can be exploited. |

## Lessons Learned

- Simple OSINT-style clues (e.g., encoded hints on a web page) can be a legitimate part of enumeration and shouldn't be dismissed.
- `sudo -l` should always be one of the first checks performed after gaining any foothold on a Linux host.
- Even well-known, "safe-looking" binaries like `find` can provide a trivial path to root when granted unrestricted sudo rights — GTFOBins is an essential reference during privilege escalation.

---

*This assessment was carried out in an authorized, isolated training-lab environment. No production systems or third-party assets were involved.*
