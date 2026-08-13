# 🎯 Master Notes: Protocols, Authentication & Nmap (TryHackMe Jr Pentester)

> **Personal cheat-sheet** covering: Protocols & Servers → Live Host Discovery → Basic Port Scans → Advanced Port Scans → Post-Port Scans (NSE, Service/OS Detection).
> Goal: re-read this once and the *whole mental model* comes back instantly.

---

## 📖 Table of Contents
1. [Authentication & Password Attacks](#1-authentication--password-attacks)
2. [Cleartext vs Encrypted Protocols](#2-cleartext-vs-encrypted-protocols)
3. [Subnetting, ARP & ICMP (Host Discovery Foundations)](#3-subnetting-arp--icmp-host-discovery-foundations)
4. [Nmap Host Discovery Techniques](#4-nmap-host-discovery-techniques)
5. [TCP SYN Scan vs TCP ACK Scan (Basics)](#5-tcp-syn-scan-vs-tcp-ack-scan-basics)
6. [Privileged vs Unprivileged Scanning](#6-privileged-vs-unprivileged-scanning)
7. [Advanced Inverse Scans (Null / FIN / Xmas / Maimon)](#7-advanced-inverse-scans-null--fin--xmas--maimon)
8. [ACK, Window & Custom Scans](#8-ack-window--custom-scans)
9. [Resolving `open|filtered` Ambiguity](#9-resolving-openfiltered-ambiguity)
10. [Spoofing, Decoys, Fragmentation & Zombie Scans](#10-spoofing-decoys-fragmentation--zombie-scans)
11. [NSE — Nmap Scripting Engine](#11-nse--nmap-scripting-engine)
12. [Service, OS Detection & Output Formats](#12-service-os-detection--output-formats)

---

## 1. Authentication & Password Attacks

**Three authentication factors** (memorize as *Know / Have / Are*):
- Something you **know** → password, PIN
- Something you **have** → phone, security key, smart card
- Something you **are** → fingerprint, face

**Hydra** = online brute-forcer against *live services* (FTP, SSH, IMAP, POP3, SMTP, HTTP...).

```bash
hydra -l username -P wordlist.txt <target> <service>
hydra -l mark -P /usr/share/wordlists/rockyou.txt 10.49.188.203 ftp
hydra -L users.txt -P passwords.txt 10.49.188.203 ssh
```

| Flag | Meaning |
|---|---|
| `-l` / `-L` | single username / username file |
| `-p` / `-P` | single password / password file |
| `-s PORT` | non-default port |
| `-t n` | parallel threads |
| `-f` | stop on first hit |
| `-V` / `-d` | verbose / debug |

**Remember:** Hydra attacks *online* live logins. **Hashcat / John the Ripper** attack *offline* stolen hashes — totally different phase (post-breach cracking, not live guessing).

Other tools: **Medusa** (Hydra alternative), **Ncrack** (Nmap project's speed-focused cracker), **CrackMapExec/NetExec** (AD/SMB/WinRM spraying), **Burp Suite Intruder** (web login forms).

**Defenses to remember as a checklist:**
Long passwords + no complexity rules → Lockout → Rate limiting → CAPTCHA (behavioral) → MFA → Passwordless (passkeys) → Breach-password blocking → Behavioral analysis (impossible travel) → IP/geo controls.

---

## 2. Cleartext vs Encrypted Protocols

**Core idea:** anything without "S" or SSH is sniffable in Wireshark in plaintext.

| Cleartext ❌ | Port | Encrypted ✅ | Port |
|---|---|---|---|
| FTP | 21 | FTPS | 990 |
| HTTP | 80 | HTTPS | 443 |
| Telnet | 23 | SSH (replaces Telnet) | 22 |
| IMAP | 143 | IMAPS | 993 |
| POP3 | 110 | POP3S | 995 |
| SMTP | 25 | SMTPS | 465 |
| — | — | SMTP Submission (STARTTLS *upgrade*, not implicit) | 587 |

**Example to remember:** SFTP and SSH both ride on port **22** — same encrypted channel, different purpose (file transfer vs shell).

**Defensive checklist:** Force TLS 1.2/1.3 → SSH key-auth only (disable passwords) → lockouts/rate-limit/MFA → HSTS headers → network segmentation.

---

## 3. Subnetting, ARP & ICMP (Host Discovery Foundations)

### Bitwise AND — "Am I local or do I need the gateway?"
A host ANDs **its own IP** and the **destination IP** with the subnet mask, then compares the two Network IDs:
- **Same Network ID** → deliver directly via Layer 2 (ARP + MAC address).
- **Different Network ID** → forward to the **Default Gateway** for routing.

**Worked example:** Host `192.168.1.50/24` (Network ID `192.168.1.0`)
- To `192.168.1.100` → ANDed = `192.168.1.0` → **matches** → local delivery via ARP.
- To `8.8.8.8` → ANDed = `8.8.8.0` → **no match** → sent to gateway.

**Remember it as:** *"Same street, knock on the door directly (ARP). Different street, hand it to the mail carrier (gateway/router)."*

### ARP — turns IP into MAC on the LAN
```
Host A: "Who has 192.168.1.20? Tell 192.168.1.10" → BROADCAST (FF:FF:FF:FF:FF:FF)
Host B: "192.168.1.20 is at BB:BB:BB:BB:BB:BB"    → UNICAST reply
```
Steps: Check ARP cache → (miss) broadcast ARP Request → target replies unicast → cache updated → data sent.

```bash
ip neighbor show          # view ARP table (Linux)
sudo ip neighbor flush all  # clear cache (Linux)
```

**Security angle:** ARP has **no authentication** → **ARP Spoofing/Poisoning** lets an attacker claim to *be* the gateway → MITM (sniffing, SSL stripping) via Ettercap/Bettercap. **Mitigation:** static ARP, Dynamic ARP Inspection (DAI), 802.1X.

### ICMP — the network's "diagnostic voice", not a data carrier
| Type | Name | Used by |
|---|---|---|
| 8 / 0 | Echo Request / Reply | `ping` |
| 3 | Destination Unreachable | firewall/routing errors |
| 11 | Time Exceeded (TTL=0) | `traceroute` |
| 5 | Redirect | router telling host to use a better gateway |

**How traceroute works (remember via TTL countdown):** send packet with `TTL=1` → first router decrements to 0, drops it, replies **Type 11** → attacker learns hop 1's IP → resend with `TTL=2` → repeat until destination reached.

**Attack angle:** Ping sweeps (recon), ICMP flood (DoS), **ICMP tunneling** (covert C2/exfil channel — ICMP payload can carry arbitrary data), Ping of Death (oversized malformed packet).

### CIDR & Range Math (don't mix these two!)
- **CIDR block (`/prefix`)** → power-of-two math: `2^(32-prefix)` total IPs.
  - `/29` → `32-29=3` host bits → `2^3=8` addresses → `8-2=6` usable (minus network + broadcast IDs).
  - Block increments in the last octet by the block size (e.g., 0,8,16,24… for `/29`).
- **Dash/range notation** (e.g., `10.10.0-255.101-125`) → NOT a CIDR block → multiply valid values **per octet**:
  `1 × 1 × 256 × 25 = 6,400` total IPs.

**Rule to remember forever:** *CIDR = binary boundary math (2^H). Range notation = simple multiplication across octet slices.* Never apply `2^H` to a dash range.

---

## 4. Nmap Host Discovery Techniques

**Default behavior:** Nmap pings first, then only port-scans hosts that respond. `-sn` = discovery only, **no port scan** (saves time).

| Scan | Command | Notes |
|---|---|---|
| ARP | `sudo nmap -PR -sn <target>` | Fastest/most accurate, **local subnet only** |
| ICMP Echo | `sudo nmap -PE -sn <target>` | Type 8/0; often firewall-blocked |
| ICMP Timestamp | `sudo nmap -PP -sn <target>` | Type 13/14; fallback when Echo blocked |
| ICMP Address Mask | `sudo nmap -PM -sn <target>` | Type 17/18; often blocked, legacy |
| TCP SYN Ping | `sudo nmap -PS22,80,443 -sn <target>` | SYN to specific ports |
| TCP ACK Ping | `sudo nmap -PA22,80,443 -sn <target>` | Bypasses stateless firewalls |
| UDP Ping | `sudo nmap -PU53,161,162 -sn <target>` | Triggers ICMP Port Unreachable |

**Privilege changes strategy automatically:**
- **Root, local network** → raw **ARP**.
- **Root, remote network** → ICMP echo + TCP ACK/80 + TCP SYN/443 + ICMP timestamp (multi-probe).
- **Standard user, remote** → plain TCP handshake to ports 80/443 (no raw sockets available).

`-n` = skip reverse DNS (faster, quieter). `-R` = force reverse DNS.

**Mnemonic:** *"Local = ARP knock on the door. Remote & privileged = throw multiple probe types. Remote & unprivileged = just try to connect like a normal app."*

---

## 5. TCP SYN Scan vs TCP ACK Scan (Basics)

**Golden rule:** SYN scan finds **open ports**. ACK scan finds **firewall rules**. They answer *different questions*.

### SYN Scan (`-sS`) — "Half-Open" scan
```
SYN ─> Open port replies SYN/ACK   (then Nmap sends RST — never completes handshake)
SYN ─> Closed port replies RST
```
Stealthy: handshake never fully completes → most application logs never see it.

### ACK Scan (`-sA`) — "Unsolicited Response" scan
```
ACK ─> No firewall  → RST (regardless of open/closed — doesn't tell you open state!)
ACK ─> Firewall present → dropped, no reply
```
Cannot tell open vs closed — only **filtered vs unfiltered**.

| | SYN Scan | ACK Scan |
|---|---|---|
| Goal | Find open ports | Map firewall rules |
| Open port reply | SYN/ACK | RST |
| Closed port reply | RST | RST |
| Result labels | open / closed | unfiltered / filtered |

---

## 6. Privileged vs Unprivileged Scanning

**Core distinction = raw socket access.**

| | Privileged (`sudo`) | Unprivileged (standard user) |
|---|---|---|
| Packet crafting | Raw sockets (custom headers) | OS `connect()` syscall only |
| Default TCP scan | `-sS` (SYN, half-open) | `-sT` (full Connect scan) |
| Handshake | Incomplete (sends RST after SYN/ACK) | Completes full 3-way handshake |
| Logging | Rarely logged (no full connection) | Logged by target app (real session opens) |
| Local discovery | Raw ARP/ICMP | TCP connect probes to 80/443 |
| Advanced flags (`-sA -sF -sN -sX`) | ✅ supported | ❌ `Requires root privileges` error |

**Example to remember:** running plain `nmap 10.10.10.10` *silently* changes behavior based on privilege — no explicit flag needed, Nmap auto-selects `-sS` under sudo or `-sT` without it.

```bash
sudo nmap -sS 10.10.10.10   # privileged: fast, stealthy, half-open
nmap -sT 10.10.10.10        # unprivileged: slower, full handshake, logged
```

---

## 7. Advanced Inverse Scans (Null / FIN / Xmas / Maimon)

**Family concept — all rely on RFC 793:** an unexpected packet *without SYN/RST/ACK* should get **RST if closed**, and be **silently dropped if open**. So "no response" = ambiguous → Nmap reports `open|filtered`.

| Scan | Flag | Header bits set | Closed reply | Open reply |
|---|---|---|---|---|
| **Null** | `-sN` | none | RST | *(silence)* |
| **FIN** | `-sF` | FIN | RST | *(silence)* |
| **Xmas** | `-sX` | FIN+PSH+URG ("lit up like a tree") | RST | *(silence)* |
| **Maimon** | `-sM` | FIN+ACK | RST | *(silence, BSD-only quirk)* |

```bash
sudo nmap -sN -p 80,443,8080 -n <target>   # Null
sudo nmap -sF -p 22,80,443 -n <target>     # FIN
sudo nmap -sX -p 22,80,443 -n <target>     # Xmas
sudo nmap -sM -p 22,80,443 -n <target>     # Maimon (BSD stacks only)
```

**⚠️ The "Windows Gotcha" — memorize this hard:**
Windows/Cisco/BSD-variants **don't follow RFC 793** for these scans → they send `RST` for **every** port → **100% falsely shows "closed."**
→ These scans only work reliably against **Linux/Unix/BSD**.
→ **Bonus use:** if *every* port comes back closed on a Null/Xmas scan, that's actually a strong hint the target is **Windows**.

**Maimon specifically:** exploits a historical bug in BSD-derived stacks (FreeBSD, OpenBSD, SunOS, AIX...) where open ports silently drop `FIN/ACK` instead of the RFC-mandated RST. Modern systems patched this, so Maimon is mostly a legacy/fingerprinting tool now.

**Why use ambiguous scans at all?** (1) Instant OS fingerprint (all-closed = Windows), (2) test if firewall is stateless (SYN-only) vs stateful, (3) stealth — no full session, no app logs.

---

## 8. ACK, Window & Custom Scans

| Scan | Flag | What it tests | Open indicator | Closed indicator |
|---|---|---|---|---|
| **ACK** | `-sA` | Firewall state only | `unfiltered` (RST either way — can't tell open/closed) | same `unfiltered` |
| **Window** | `-sW` | Same ACK probe, but reads **TCP Window size** in the RST | Window `> 0` → open | Window `= 0` → closed |
| **Custom** | `--scanflags` | Any arbitrary flag combo, e.g. `--scanflags SYNFIN` | Depends on base engine (`-sS` or `-sF` logic) | — |

**Remember the twist:** ACK and Window send the *exact same packet* — Window scan just looks *deeper* (at the buffer-size field) to squeeze out open/closed info that plain ACK can't give. Only works on OS stacks with this particular quirk (older FreeBSD, AIX, OpenBSD, z/OS).

```bash
sudo nmap -sA 10.49.134.130
sudo nmap -sW 10.49.134.130
sudo nmap --scanflags URGACKPSHRSTSYNFIN 10.49.134.130
```

---

## 9. Resolving `open|filtered` Ambiguity

When Null/FIN/Xmas/Maimon give you `open|filtered`, it literally means: *"nothing came back — could be an open port following RFC 793, or a firewall silently eating the packet."* Disambiguate step by step:

1. **`-sV`** (version detection) — forces an actual app-layer handshake (HTTP GET, SSH banner). If it answers → reclassified straight to `open`.
2. **`-sA`** (ACK scan) cross-reference:

   | Inverse scan result | ACK scan result | Conclusion |
   |---|---|---|
   | open\|filtered | `unfiltered` | **OPEN** (ACK reached host & got RST → no firewall blocking) |
   | open\|filtered | `filtered` | **FILTERED** (firewall ate both probes) |

3. **`-sS`** (standard SYN) if stealth no longer matters — SYN/ACK = open, silence/ICMP = filtered.
4. **NSE default scripts** (`-sC`) — force Layer 7 responses.

**Mental shortcut:** *Inverse scan tells you what RFC 793 says the host would do. ACK scan tells you whether a firewall is even letting packets through at all. Combine both to triangulate the truth.*

---

## 10. Spoofing, Decoys, Fragmentation & Zombie Scans

### IP Spoofing (`-S`) — Layer 3, one-way
Forges the source IP in the packet header. **Problem:** replies go to the *spoofed* IP, not you — so you can't read results directly (unless you control that host, sniff the wire, or do an Idle scan).
```bash
sudo nmap -S 10.0.0.99 -e eth0 -Pn 10.0.0.1
```

### Decoy Scan (`-D`) — blend in with noise
Sends your real probe *alongside* fake decoy-sourced probes simultaneously — the target's logs show many "attackers" at once, hiding you.
```bash
sudo nmap -D 10.0.0.15,10.0.0.16,ME,10.0.0.17 10.0.0.1
sudo nmap -D RND:10 10.0.0.1     # 10 random decoys
```

### MAC Spoofing (`--spoof-mac`) — Layer 2, local only
Forges hardware address; only works on the **same LAN** (MAC headers get rewritten at every router hop). Useful for bypassing NAC / vendor filtering.
```bash
sudo nmap --spoof-mac Apple 192.168.1.1
sudo nmap --spoof-mac 0 192.168.1.1     # fully random MAC
```

**Layer comparison to lock in:**
| | MAC Spoof | IP Spoof (`-S`) | Decoy (`-D`) |
|---|---|---|---|
| Layer | 2 | 3 | 3 |
| Scope | LAN only | Cross-network | Cross-network |
| Get replies back? | ✅ Yes | ❌ No | ✅ Yes (real IP included) |

### Fragmentation (`-f` / `-ff`)
Splits packet headers across multiple small IP fragments to slip past pattern-matching firewalls/IDS.
- `-f` → 8-byte chunks (a 24-byte TCP header → 3 fragments).
- `-ff` → 16-byte chunks (→ 2 fragments).
- `--mtu <n>` must be a **multiple of 8**.
```bash
sudo nmap -sS -p80 -f <target>
```
Reassembly relies on IP header's **Identification** (matches fragments to same packet) and **Fragment Offset** (ordering) fields.

### Idle/Zombie Scan (`-sI <zombie_IP>`)
Uses a quiet third-party "zombie" host's predictable **IP ID increment** to scan a target **without ever sending a packet from your own IP** — the ultimate attribution-hiding technique.

### Diagnostics you'll always want on hand
```
--reason   # WHY nmap classified a port that way (syn-ack, rst, no-response)
-v / -vv   # verbosity
-d / -dd   # debug output
```

**Defenses to remember:** BCP38/Reverse Path Forwarding blocks IP spoofing at the ISP; Dynamic ARP Inspection + port security blocks MAC spoofing; stateful firewalls drop unmatched spoofed-IP replies.

---

## 11. NSE — Nmap Scripting Engine

Scripts are written in **Lua**, categorized by intent/risk:

| Category | Purpose | Risk |
|---|---|---|
| `default` | Curated safe set, runs with `-sC`/`-A` | Safe |
| `safe` | Minimal-footprint diagnostics | Very low |
| `version` | Deeper version fingerprinting | Low |
| `discovery` | Map accessible infra (DB tables, DNS zones) | Low |
| `auth` | Weak-credential / auth-bypass checks | Medium |
| `vuln` | Checks for unpatched vulns (no exploit) | Medium |
| `malware` | Detect backdoors/rootkits already present | Medium |
| `broadcast` | LAN-wide asset discovery via broadcasts | Low (LAN) |
| `external` | Sends target info to 3rd parties (Whois, VirusTotal) | Low (but leaks target info!) |
| `brute` | High-speed password spraying | High |
| `intrusive` | Umbrella for anything loud/risky | High |
| `dos` | Crash-testing | High |
| `fuzzer` | Malformed payload injection | High |
| `exploit` | Actively runs PoC exploits | High |

**Logical selectors — very useful, remember the syntax:**
```bash
--script "safe or discovery"          # combine categories
--script "default and not intrusive"  # default minus risky stuff
--script-help http-robots.txt         # read docs before running
```

**Practical patterns:**
```bash
sudo nmap -sV -sC -p22 <target>                 # default scripts + version on SSH
sudo nmap -sV --script vuln -p80,443 <target>   # vuln-hunt a web server
sudo nmap -p21 --script brute <target>          # intrusive password audit on FTP
```

**Rule to remember:** `-sC` is literally shorthand for `--script=default`. `-A` = `-sV + -O + -sC + --traceroute` all in one (great for quick recon, bad for stealth).

---

## 12. Service, OS Detection & Output Formats

| Flag | Purpose |
|---|---|
| `-sV` | Service/version detection (probes to ID exact software version) |
| `--version-light` | Faster, lower-intensity version probing |
| `--version-all` | Exhaustive version probing (slow, thorough) |
| `-O` | OS fingerprinting via TCP/IP stack quirks |
| `--traceroute` | Map hops to target |
| `-A` | Aggressive: `-sV -O -sC --traceroute` combined |

**Why `-sV` matters most:** it's the difference between seeing `80/tcp open http` and seeing `Apache httpd 2.4.41 (Ubuntu)` — the version is what lets you match known CVEs.

### Output formats — save everything, always
| Flag | Format | Use case |
|---|---|---|
| `-oN file` | Normal | Human-readable, matches terminal output |
| `-oG file` | Grepable | One host per line — great for `grep`/`awk`/`cut` |
| `-oX file` | XML | Import into Metasploit, Zenmap, dashboards |
| `-oA basename` | All three at once | `.nmap`, `.gnmap`, `.xml` in one command |

```bash
sudo nmap -A 10.10.244.5
sudo nmap -sV --script=http-robots.txt -p80 -oA initial_recon 10.10.244.5
```

**Habit to build forever:** always tack `-oA <project_name>` onto real engagements — future-you (or a report) will need the raw XML/grepable data, not just what scrolled past in the terminal.

---

## 🧠 One-Page Recall Cheat Sheet

```
DISCOVERY:      -sn (host discovery only) | -PR (ARP) | -PE/-PP/-PM (ICMP) | -PS/-PA/-PU (TCP/UDP ping)
BASIC SCANS:     -sS (SYN, needs root)     | -sT (Connect, no root needed) | -sA (ACK, firewall map only)
INVERSE SCANS:   -sN (Null) -sF (FIN) -sX (Xmas) -sM (Maimon)  → open|filtered on Linux, all-closed = Windows tell
OTHER SCANS:     -sW (Window, reads RST size) | --scanflags (fully custom)
EVASION:         -S (spoof IP) -D (decoys) --spoof-mac -f/-ff (fragment) -sI (zombie/idle)
ENUMERATION:     -sV (version) -O (OS) -sC/--script (NSE) -A (aggressive = all of the above)
OUTPUT:          -oN (normal) -oG (grepable) -oX (XML) -oA (all three)
DIAGNOSTICS:     --reason -v/-vv -d/-dd
```

---
*Compiled from TryHackMe Jr Pentester rooms: Protocols & Servers 2, Nmap Live Host Discovery, Nmap Basic Port Scans, Nmap Advanced Port Scans, Nmap Post Port Scans.*
