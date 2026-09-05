---
title: "HTB: BlockSynergy (Linux Insane) - Partial Notes"
date: 2026-09-04
tags: ["ctf", "htb", "linux", "blockchain", "ssrf", "command-injection", "path-traversal", "race-condition"]
description: "Recon methodology and general attack-surface notes for HackTheBox BlockSynergy. Full writeup after retirement."
ShowToc: true
TocOpen: false
draft: false
---

> **Note:** BlockSynergy is an active HackTheBox machine. This post covers only
> reconnaissance and general attack-surface observations. The full writeup goes up
> once the machine retires.

---

## Machine Profile

BlockSynergy is a Linux box built around a blockchain web app. From the outside
there was almost nothing to hit, just SSH and one web port, so the whole thing came
down to a single question: how do you reach the parts of the app it keeps to
itself? Everything worth finding ran on localhost, and most of my time went into
convincing the server to fetch it for me.

```text
OS:         Ubuntu 24.04
Difficulty: Insane
Themes:     Blockchain web app, Flask/Werkzeug, internal services
```

---

## Reconnaissance

I opened with a full TCP scan:

```bash
nmap -p- -sCV --open -oA scan/blocksynergy target_address
```

Only two ports came back, and that was the entire external surface:

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 8080 | HTTP | Werkzeug httpd 3.1.3, Python 3.12.3 (Flask) |

Two things stood out. First, it's a Werkzeug dev server, and those rarely run
alone, so I assumed there were other Flask apps bound to localhost that a scan
would never show. Second, with only 22 and 8080 reachable, anything else on the box
had to be internal.

I added the host:

```bash
echo "target_address blocksynergy.htb" | sudo tee -a /etc/hosts
```

The app was a blockchain dashboard. I registered, got a wallet, and found a
self-documenting API index listing its transaction and node endpoints. It handed me
a map of its own surface, so that is where I spent my time.

---

## General Attack Categories

The techniques the box touches, with the reading I'd point you at instead of a
walkthrough:

- Blockchain reward / coinbase transaction trust: [Bitcoin developer guide: transactions](https://developer.bitcoin.org/reference/transactions.html)
- SSRF and loopback filter bypasses: [PortSwigger: SSRF](https://portswigger.net/web-security/ssrf), [PayloadsAllTheThings: SSRF](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/README.md)
- OS command injection: [OWASP: Command Injection](https://owasp.org/www-community/attacks/Command_Injection), [PayloadsAllTheThings: Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md)
- Path traversal file writes: [CWE-22: Path Traversal](https://cwe.mitre.org/data/definitions/22.html)
- TOCTOU race conditions: [CWE-367: Time-of-check Time-of-use](https://cwe.mitre.org/data/definitions/367.html)

---

## Tools

`nmap` `curl` `python3` `penelope` / `nc`
`xxd` `bash` `grep`
