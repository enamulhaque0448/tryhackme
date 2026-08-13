<img width="1200" height="558" alt="image" src="https://github.com/user-attachments/assets/adb996d5-fdfd-452f-947f-1d030649ccfa" />
<img width="1453" height="563" alt="image" src="https://github.com/user-attachments/assets/3494e994-a921-41bc-965d-d4f11748fecb" />
<img width="1829" height="499" alt="image" src="https://github.com/user-attachments/assets/d9c39ad5-c5d8-4924-9fdc-8760e9c8a3ac" />
# Nmap Scripting Engine (NSE) Categories & Architectural Overview

The **Nmap Scripting Engine (NSE)** is a powerful tool subsystem that allows users to write and automate scripts to perform advanced network tasks. These scripts are written in the **Lua** programming language and are categorized based on their behavior, safety, and operational intent.

---

## 🗂️ Core NSE Script Categories

| Script Category | Operational Purpose & Description | Risk Profile |
| :--- | :--- | :--- |
| **`auth`** | Executes scripts related to authentication systems, checking for weak credentials or bypassing login requirements on the target. | Medium |
| **`broadcast`** | Discovers local network assets dynamically by sending unrequested broadcast messages across the local layer. | Low (LAN Only) |
| **`brute`** | Performs high-speed brute-force password auditing campaigns against accessible network login portals. | High (Intrusive) |
| **`default`** | Runs a carefully selected subset of essential scripts. This group is triggered automatically when passing the **`-sC`** or **`-A`** flags. | Safe / Low |
| **`discovery`** | Queries live services to map accessible infrastructure data, including active database tables, directory listings, and DNS zones. | Low |
| **`dos`** | Tests for application instability by probing for flaws that could crash or lock up the target system (Denial of Service). | High (Disruptive) |
| **`exploit`** | Actively attempts to breach a service by running known proof-of-concept exploits against discovered software flaws. | High (Intrusive) |
| **`external`** | Forwards target details to third-party databases (e.g., Whois servers, Geoplugin, or VirusTotal) for extended intelligence gathering. | Low (Reveals Target) |
| **`fuzzer`** | Sends randomized, unexpectedly structured, or malformed payloads to software fields to discover hidden software bugs or crashes. | High (Disruptive) |
| **`intrusive`** | A broad umbrella category for heavy, high-volume scripts (like exploitation or brute-forcing) likely to crash the target or trigger alerts. | High (Intrusive) |
| **`malware`** | Audits the remote system for known security compromises, searching for active backdoors, worms, or malicious rootkit installations. | Medium |
| **`safe`** | General diagnostic scripts explicitly engineered with minimal resource footprints to avoid system disruption or application crashes. | Very Low (Safe) |
| **`version`** | Extends default version matching by actively analyzing responses to identify exact application software variations. | Low |
| **`vuln`** | Scans target services to cross-check whether they contain unpatched software vulnerabilities, without actively executing full exploits. | Medium |

---

## 🏗️ Core NSE Execution Concepts

To use the Nmap Scripting Engine effectively, you must understand how Nmap invokes and handles these Lua automation elements:

### 1. Script Arguments (`--script-args`)
Many advanced scripts require external configuration context, such as a custom username dictionary or a specific network parameter. You pass these into the engine using key-value pairs:
```bash
nmap -p 445 --script http-vhosts --script-args http-vhosts.domain=example.com <TARGET>
```

### 2. Boolean Logical Selectors
You do not have to load single categories at a time. The NSE engine supports logical operators (`and`, `or`, `not`) to let you design precise auditing sweeps:
*   **Run safe and discovery scripts together:** `--script "safe or discovery"`
*   **Run all default scripts EXCEPT intrusive ones:** `--script "default and not intrusive"`

### 3. Real-Time Script Help (`--script-help`)
Before executing an unknown script, you can query its documentation, parameters, and description metadata directly from your local terminal database:
```bash
nmap --script-help http-robots.txt
```

---

## 🖥️ Practical Command Examples

*   **Audit an SSH Server with Default Scripts:**
    ```bash
    sudo nmap -sV -sC -p22 <TARGET>
    ```
*   **Scan for Active Vulnerabilities on an Web Server:**
    ```bash
    sudo nmap -sV --script vuln -p80,443 <TARGET>
    ```
*   **Run an Intrusive Password Audit against an FTP Service:**
    ```bash
    sudo nmap -p21 --script brute <TARGET>
    ```
<img width="1685" height="641" alt="image" src="https://github.com/user-attachments/assets/6bcad5b9-beb0-44cc-b4fa-b3814b57ea96" />
<img width="1834" height="630" alt="image" src="https://github.com/user-attachments/assets/e019c1ed-0080-402a-a918-ffd68b58e910" />

# Nmap Service, OS, and Script Scanning Guide

This reference guide covers service detection, operating system fingerprinting, Nmap Scripting Engine (NSE) automation, and output formatting flags.

---

## 🔍 Service & OS Enumeration Matrix

| Nmap Option | Operational Purpose & Importance | Real-World Utility & Impact |
| :--- | :--- | :--- |
| **`-sV`** | **Service Version Detection.** Probes open ports to identify the exact application name and version number. | Highly critical. Prevents false assumptions about port uses and identifies vulnerable software versions. |
| **`--version-light`** | **Lightweight Version Sweep.** Limits probes to the most likely matching profiles (Intensity Level 2). | Maximizes speed on large networks where full version sweeps take too long. |
| **`--version-all`** | **Aggressive Version Sweep.** Forces Nmap to test every single available version probe (Intensity Level 9). | Invaluable for identifying heavily modified, uncommon, or obscured background network services. |
| **`-O`** | **OS Fingerprinting.** Analyzes subtle TCP/IP stack response patterns to guess the target's operating system. | Helps select appropriate exploits (e.g., distinguishing a Linux target from a Windows server). |
| **`--traceroute`** | **Network Route Tracing.** Maps the exact sequence of network routers (hops) between the tester and target. | Crucial for mapping network topology and discovering intermediate hardware firewalls. |
| **`-A`** | **Aggressive Scan Mode.** Combines `-sV`, `-O`, `-sC`, and `--traceroute` into a single directive. | Best for quick, comprehensive assessments on a single target IP when stealth is not a priority. |

### 🛠️ Scripting Automation (`NSE`)

*   **`--script=SCRIPTS`**: Automatically executes custom user scripts or comma-separated categories (e.g., `--script=vuln,safe`).
*   **`-sC` / `--script=default`**: Fires a pre-configured array of safe, high-utility diagnostic scripts designed to grab essential surface data.

---

## 💾 Scan Output Management (Reporting)

Nmap offers specialized logging options to ensure results can be parsed easily by humans, terminal commands, or tracking software.

*   **`-oN <FILE>` (Normal):** Saves output exactly as it appears on your terminal screen. Essential for human-read review.
*   **`-oG <FILE>` (Grepable):** Compresses host data onto a single flat line. Perfect for quick command-line filtering using `grep`, `awk`, or `cut`.
*   **`-oX <FILE>` (XML Format):** Structures raw scan details into clean XML syntax. Used to import data into security dashboards like Metasploit, Zenmap, or custom reporting parsers.
*   **`-oA <BASENAME>` (All Formats):** Instantly outputs all three variations (`.nmap`, `.gnmap`, and `.xml`) simultaneously under a single project name.

---

## 🖥️ Command Implementation & Example Outputs

### Example 1: The Aggressive Audit (`-A`)
**Command:**
```bash
sudo nmap -A 10.10.244.5
```

**Simulated Output:**
```text
Starting Nmap 7.94 ( https://nmap.org ) at 2026-08-13 14:10 UTC
Nmap scan report for target.internal (10.10.244.5)
Host is up (0.012s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|_  256 ed:a3:cf:41:22:ef:31:0c (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Welcome to Corporate Portal
|_http-server-header: Apache/2.4.41 (Ubuntu)

Network Distance: 2 hops

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   4.10 ms  10.10.0.1
2   11.80 ms 10.10.244.5

OS details: Linux 5.4 - 5.11
```
*   **Importance:** This output cleanly maps the exact version numbers (`OpenSSH 8.2p1`, `Apache 2.4.41`), pulls standard public data via automated scripts (`http-title`), maps the route infrastructure via traceroute, and narrows down the target kernel to Linux.

### Example 2: Target Script Enumeration & Multi-Format Saving (`-oA`)
**Command:**
```bash
sudo nmap -sV --script=http-robots.txt -p80 -oA initial_recon 10.10.244.5
```

**Simulated Output:**
```text
Starting Nmap 7.94 ( https://nmap.org ) at 2026-08-13 14:15 UTC
Nmap scan report for target.internal (10.10.244.5)
Host is up (0.0091s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41
| http-robots.txt: 2 disallowed entries found 
|_/backup/ /dev/private/

Output saved to: initial_recon.nmap, initial_recon.gnmap, initial_recon.xml
Nmap done: 1 IP address (1 host up) scanned in 4.12 seconds
```
*   **Importance:** The script successfully discovered hidden web directories (`/backup/` and `/dev/private/`) listed in the server's tracking file. The `-oA` suffix automatically created three report files in your local directory for backup logging.






