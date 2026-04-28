---
title: "HTB: Logging (Windows Medium) - Partial Notes"
date: 2026-04-21
tags: ["ctf", "htb", "windows", "active-directory", "kerberos"]
description: "Recon methodology and general attack surface notes for HackTheBox Logging. Full writeup after retirement."
ShowToc: true
TocOpen: false
draft: false
---

> **Note:** Logging is an active HackTheBox machine. This post covers only
> reconnaissance methodology and general observations. The full writeup will
> be published after the machine is retired.

---

## Machine Profile

A Windows domain controller running a fairly realistic enterprise configuration.
The attack surface involves services you'd expect to find in an actual AD environment
rather than contrived CTF gimmicks, which is what made it interesting.

```text
OS:         Windows Server 2019
Domain:     logging.htb
Difficulty: Medium
```

---

## Reconnaissance

Starting with a full TCP scan to map the full port surface before going narrow:

```bash
nmap -p- -sCV -T5 --open -oA scan/tcp_detailed <IP>
```

The open ports tell you immediately this is a domain controller:

| Port | Service |
|------|---------|
| 53 | DNS |
| 88 | Kerberos |
| 389 / 636 | LDAP / LDAPS |
| 445 | SMB |
| 5985 | WinRM |
| 8530 / 8531 | WSUS |

The WSUS ports (8530/8531) stood out. Worth keeping in the back of your mind.

Added the DC to `/etc/hosts` with its hostname and domain before proceeding,
since many AD tools resolve by name rather than IP.

---

## General Attack Categories

The techniques this box touches on, without getting into specifics:

- Shadow Credentials: [Elad Shamir's original research](https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab)
- ADCS abuse: [SpecterOps "Certified Pre-Owned"](https://posts.specterops.io/certified-pre-owned-d95910965cd2)
- WSUS attacks: [the WSUXploit / wsuks tooling](https://github.com/pimps/wsuxploit)

---

## Tools


`nmap` `netexec` `smbclient` `bloodhound-python`
`impacket` `certipy-ad` `evil-winrm` `msfvenom`
`krbrelayx` `stunnel` `wsuks` `openssl`


---

*Full writeup with the complete attack chain will be posted once the machine
retires.*
