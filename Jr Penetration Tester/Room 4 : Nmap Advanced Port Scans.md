# Advanced Nmap Scanning & Evasion Techniques

This module covers advanced port scanning methodologies, custom packet crafting, and firewall evasion/spoofing strategies using Nmap.

---

## 🔍 Advanced Port Scan Types

| Scan Type | Nmap Flag | Core Mechanism & Behavior | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **Null Scan** | `-sN` | Sends a TCP packet with **no flags set**. Open ports ignore it (no response), while closed ports reply with an `RST`. | Bypassing basic stateless firewalls and stealth tracking. |
| **FIN Scan** | `-sF` | Sends a TCP packet with **only the FIN flag** set. Like the Null scan, open ports drop it, and closed ports return an `RST`. | Probing target systems without initiating a standard connection. |
| **Xmas Scan** | `-sX` | Sets the **FIN, PSH, and URG flags** simultaneously (lighting the packet up "like a Christmas tree"). | Probing ports hidden behind strict or stateless firewalls. |
| **Maimon Scan** | `-sM` | Sets both **FIN and ACK flags** together in a single custom probe. | Exploiting specific TCP stack behaviors found in older BSD-derived systems. |
| **ACK Scan** | `-sA` | Sends a packet containing **only the ACK flag**. | Mapping out firewall filtering rules rather than discovering open ports. |
| **Window Scan** | `-sW` | Examines the specific **TCP Window field size** inside returned `RST` packets. | Differentiating open from closed ports on systems with unique TCP architectures. |
| **Custom Scan** | `--scanflags` | Allows the manual configuration of **tailored TCP flag combinations** (e.g., `--scanflags SYNFIN`). | Bypassing customized intrusion detection systems (IDS). |

---

## 🥷 Evasion & Spoofing Techniques

*   **Spoofing IP (`-S <IP>`):** Forges the source IP address in the packet header. Scan traffic appears to originate from a different host, though replies will go to that spoofed host.
*   **Spoofing MAC (`--spoof-mac <MAC>`):** Forges your network card's hardware address. This works exclusively when scanning targets on your same local Layer 2 network segment.
*   **Decoy Scan (`-D <Decoy1,Decoy2,ME>`):** Mixes your real IP address among a list of fake decoy addresses. This obfuscates your true identity in the target's security logs.
*   **Fragmented Packets (`-f` or `-ff`):** Splits the raw IP packets into much smaller fragmented pieces. This splits header strings across multiple packets to blind firewalls and Intrusion Detection Systems (IDS).
*   **Idle / Zombie Scan (`-sI <Zombie_IP>`):** Leverages an inactive third-party host's IP ID increment sequence to map the target. This scans the destination without ever routing a single packet directly from your own IP.

---

## 📊 Verbosity & Diagnostics Reference

To gain real-time insight into why Nmap classifies a port a certain way, append these diagnostics parameters to your execution string:

*   `--reason`: Displays the exact packet response type (e.g., `syn-ack`, `rst`, `no-response`) that led Nmap to determine the port state.
*   `-v` / `-vv`: Increases output verbosity levels, showing discovered ports and state transitions in real time.
*   `-d` / `-dd`: Triggers detailed debugging diagnostics output, essential for troubleshooting failing scans or network timeouts.
## TCP Null Scan, FIN Scan, and Xmas Scan:
<img width="2400" height="1240" alt="image" src="https://github.com/user-attachments/assets/fb827e6b-8372-4140-a2bc-96dfefe50a54" />

<img width="2400" height="1240" alt="image" src="https://github.com/user-attachments/assets/171470b3-f842-44f1-b3de-1e82f388e3e7" />

A **Null Scan (`-sN`)** is a stealthy TCP port scanning technique that crafts a TCP packet with **all control flags turned off (0 / empty)**.

Under normal circumstances, every valid TCP packet contains at least one flag (e.g., `SYN`, `ACK`, `FIN`). A Null scan exploits how operating systems handle illegal or unexpected flag combinations according to the core TCP RFC specification.

---

### Mechanics & RFC 793 Logic

A Null scan sets all 6 standard TCP header flags (`URG`, `ACK`, `PSH`, `RST`, `SYN`, `FIN`) to **0**:

```text
TCP Header Flags: [ 0 0 0 0 0 0 ] (No bits set)

```

Nmap relies on **RFC 793** (the foundational TCP standard), which dictates how a system should react when receiving an unexpected packet without a `SYN`, `RST`, or `ACK` flag:

* **If the target port is CLOSED:** The target operating system is required to reply with a **`RST` (Reset)** packet.
* **If the target port is OPEN:** The target operating system must **ignore and drop** the packet silently (no response).
* **If a FIREWALL is blocking:** The firewall drops the packet silently (no response) or sends an ICMP error.

```text
                      ┌── [Port Closed] ──> Responds with: RST
[Tester] ── (0 Flags) ─>┤
                      └── [Port Open]   ──> Responds with: Nothing (Packet Dropped)

```

---

### Response Matrix & Nmap Classification

Because an **Open** port and a **Filtered** port (firewall block) both produce **no response**, Nmap cannot distinguish between the two and classifies the result as **`open|filtered`**.

| Port State | Target Response | Nmap Inferred Result |
| --- | --- | --- |
| **Closed** | `RST` Packet | `closed` |
| **Open** | *No Response* | `open|filtered` |
| **Filtered (Firewall)** | *No Response* or *ICMP Unreachable* | `open|filtered` |

---

### Primary Advantages in Security Testing

* **Stateless Firewall Evasion:** Simple or legacy stateless firewalls look specifically for incoming `SYN` packets to block unauthorized inbound connections. Null scan packets contain no `SYN` flag, allowing them to pass through unmonitored.
* **Stealth (No Session Logging):** Because no TCP connection is ever initialized or completed, application-level logs (such as Apache, Nginx, or SSH) will not record the probe.
* **Minimal Traffic Footprint:** Requires sending only a single tiny packet per targeted port.

---

### Critical Limitations & The "Windows Gotcha"

1. **Requires Privileged Access:** Like SYN scans, crafting raw TCP packets with custom zero-flag headers requires raw socket permissions (`sudo` / `root`).
2. **RFC Non-Compliance (Windows/Cisco Issue):**
* Operating systems like **Microsoft Windows**, Cisco devices, and some BSD implementations **do not strictly follow RFC 793**.
* Windows responds with an `RST` packet for **every** probe, regardless of whether the target port is open or closed.
* **Result:** Running a Null scan against a Windows target falsely shows **100% of ports as closed**.
* **Best Target OS:** Systems based on Linux, Unix, and BSD that strictly adhere to RFC 793.



---

### Nmap Command Syntax

```bash
# Basic Null scan against a target
sudo nmap -sN 10.10.10.15

# Null scan targeting specific web ports without DNS resolution
sudo nmap -sN -p 80,443,8080 -n 10.10.10.15

```
<img width="1434" height="548" alt="image" src="https://github.com/user-attachments/assets/66ec6f71-6c03-4f70-87e0-bbbe064a23ae" />


---
A **FIN Scan (`-sF`)** is an inverse TCP stealth scanning technique that crafts a TCP packet with **only the FIN (Finish) flag set** in the header.

In standard TCP communication, a `FIN` packet is used to gracefully close an established connection. A FIN scan sends an unsolicited `FIN` packet to a target port without establishing a connection first, exploiting how operating system network stacks process out-of-sequence packets according to the RFC 793 standard.

---

### Mechanics & RFC 793 Logic

In a FIN scan, Nmap crafts a raw TCP packet where only the `FIN` bit is enabled (`1`), and all other control flags (`SYN`, `ACK`, `RST`, `PSH`, `URG`) are set to `0`:

```text
TCP Header Flags: [ 0 0 0 0 0 1 ] (Only FIN bit set)

```

Nmap relies on **RFC 793** rule execution for unexpected packets received without a `SYN`, `RST`, or `ACK` flag:

* **If the target port is CLOSED:** The target operating system kernel **must reply with an `RST` (Reset)** packet.
* **If the target port is OPEN:** The target operating system kernel **must ignore and drop** the packet silently without replying.
* **If a FIREWALL is blocking:** The packet is dropped silently, or an ICMP unreachable error is returned.

```text
                      ┌── [Port Closed] ──> Responds with: RST
[Tester] ── (FIN Set) ─>┤
                      └── [Port Open]   ──> Responds with: Nothing (Packet Dropped)

```

---

### Response Matrix & Nmap Classification

Because an **Open** port and a **Filtered** port (firewall block) both result in silence (no packet returned), Nmap cannot definitively separate the two and classifies the result as **`open|filtered`**.

| Port State | Target Response | Nmap Inferred Result |
| --- | --- | --- |
| **Closed** | `RST` Packet | `closed` |
| **Open** | *No Response* | `open|filtered` |
| **Filtered (Firewall)** | *No Response* or *ICMP Unreachable* | `open|filtered` / `filtered` |

---

### Advantages & Use Cases

* **Stateless Firewall Evasion:** Simple or legacy stateless firewalls filter inbound traffic by looking for the `SYN` flag to block unauthorized connection attempts. Because a `FIN` scan packet lacks a `SYN` bit, it often passes through uninspected.
* **Stealth (No Session Logs):** Since no full TCP connection or handshake is initiated, application-level logs (such as Apache, Nginx, or SSH) will not log a connection attempt.
* **Subtle Probe Profile:** Less likely to trigger basic intrusion detection rules that monitor exclusively for `SYN` floods or standard port scans.

---

### Limitations & The "Windows Gotcha"

1. **Requires Privileged Access:** Crafting raw TCP packets with custom flag combinations requires raw socket permissions (`sudo` / `root`).
2. **RFC Non-Compliance (Windows / Cisco Issue):**
* Microsoft Windows, Cisco devices, printer OSs, and some BSD implementations **do not comply with RFC 793** regarding out-of-sequence `FIN` packets.
* Windows replies with an `RST` packet for **every** probe, regardless of whether the target port is open or closed.
* **Result:** Running a FIN scan against a Windows host will falsely display **100% of ports as closed**.
* **Target OS:** Works reliably on Linux, Unix, and BSD systems that strictly adhere to RFC 793.



---

### Nmap Command Syntax

```bash
# Basic FIN scan against a target
sudo nmap -sF 10.10.10.15

# FIN scan targeting specific ports with fast execution and no DNS resolution
sudo nmap -sF -p 22,80,443 -n 10.10.10.15

```
<img width="1347" height="534" alt="image" src="https://github.com/user-attachments/assets/f7df05cd-2a2d-480d-85d4-a68ffaeb457d" />

---

An **Xmas Scan (`-sX`)**—or **Christmas Tree Scan**—is an inverse TCP stealth scanning technique that crafts a raw TCP packet with the **FIN (Finish)**, **PSH (Push)**, and **URG (Urgent)** flags set simultaneously in the header.

The scan gets its name because the control flags are "lit up like a Christmas tree" in packet analyzers like Wireshark. Like Null (`-sN`) and FIN (`-sF`) scans, an Xmas scan relies on specific rules defined in **RFC 793** to infer port states without initiating a standard TCP handshake.

---

### Mechanics & RFC 793 Logic

In an Xmas scan, Nmap sets three specific bits to `1` in the TCP flag byte:

```text
TCP Header Flags: [ URG: 1 | ACK: 0 | PSH: 1 | RST: 0 | SYN: 0 | FIN: 1 ]

```

Because the packet contains an illegal combination of flags without a `SYN`, `RST`, or `ACK` flag, the target operating system's network stack processes it according to RFC 793 out-of-sequence packet handling:

* **If the target port is CLOSED:** The operating system kernel **must respond with an `RST` (Reset)** packet.
* **If the target port is OPEN:** The operating system kernel **must ignore and drop** the packet silently (no response).
* **If a FIREWALL is blocking:** The packet is dropped silently (no response) or returns an ICMP unreachable error.

```text
                               ┌── [Port Closed] ──> Responds with: RST
[Tester] ── (FIN + PSH + URG) ─>┤
                               └── [Port Open]   ──> Responds with: Nothing (Packet Dropped)

```

---

### Response Matrix & Nmap Classification

Because an **Open** port and a **Filtered** port (firewall drop) both result in silence (no packet returned), Nmap classifies non-responsive ports as **`open|filtered`**.

| Port State | Target Response | Nmap Inferred Result |
| --- | --- | --- |
| **Closed** | `RST` Packet | `closed` |
| **Open** | *No Response* | `open|filtered` |
| **Filtered (Firewall)** | *No Response* or *ICMP Unreachable* | `open|filtered` / `filtered` |

---

### Advantages & Primary Use Cases

* **Stateless Firewall Evasion:** Legacy stateless firewalls inspect incoming packets specifically for the `SYN` bit to block new connection attempts. Because an Xmas scan lacks a `SYN` flag, it frequently slips through stateless rules.
* **No Application Session Logs:** Since no TCP connection or handshake is ever completed, application-level logging services (such as Apache, Nginx, or OpenSSH) will not write a connection record to disk.
* **OS Fingerprinting Utility:** Observing how a target system reacts to an abnormal `FIN+PSH+URG` flag combination helps security tools determine the underlying operating system stack.

---

### Limitations & The "Windows Gotcha"

1. **Requires Privileged Access:** Crafting raw packets with custom flag combinations requires raw socket permissions (`sudo` / `root`).
2. **RFC Non-Compliance (Windows / Cisco Issue):**
* **Microsoft Windows**, Cisco devices, printer firmwares, and some BSD implementations do **not** comply with RFC 793 regarding unexpected flag combinations.
* Windows returns an `RST` packet for **every** port, regardless of whether the target port is open or closed.
* **Result:** Running an Xmas scan against a Windows target falsely indicates that **100% of ports are closed**.
* **Target OS Compatibility:** Reliable primarily against Linux, Unix, and BSD systems that strictly adhere to RFC 793.


3. **IDS/IPS Signatures:** Modern Intrusion Detection Systems (like Snort or Suricata) easily flag Xmas scans because a packet with `FIN+PSH+URG` set without an active session is an obvious anomaly.

---

### Nmap Command Syntax

```bash
# Basic Xmas scan against a target
sudo nmap -sX 10.10.10.15

# Xmas scan targeting specific ports with fast execution and no DNS resolution
sudo nmap -sX -p 22,80,443 -n 10.10.10.15

```
<img width="1361" height="540" alt="image" src="https://github.com/user-attachments/assets/8ecdc3b9-f566-4e32-b771-db24c4c530ba" />

---

### Inverse TCP Scan Family Summary

Null, FIN, and Xmas scans are three variations of the same fundamental concept:

| Scan Type | Flag Syntax | Nmap Flag | Header Flag State | Open Port Behavior |
| --- | --- | --- | --- | --- |
| **Null Scan** | None | `-sN` | `[ 0 0 0 0 0 0 ]` | No response (`open|filtered`) |
| **FIN Scan** | `FIN` | `-sF` | `[ 0 0 0 0 0 1 ]` | No response (`open|filtered`) |
| **Xmas Scan** | `FIN, PSH, URG` | `-sX` | `[ 1 0 1 0 0 1 ]` | No response (`open|filtered`) |

---
### Why Inverse Scans Matter (Despite Ambiguous Output)

While an `open|filtered` result looks like Nmap is just guessing, these scans serve three crucial offensive and diagnostic purposes that standard SYN scans cannot fulfill:

* **Instant OS Identification (Windows vs. Linux):**
* If an Xmas or Null scan returns `closed` (RST) for **every single port**, the target is running **Windows** (which ignores RFC 793).
* If it returns a mix of `closed` and `open|filtered`, the target is running **Linux, Unix, or BSD**.


* **Firewall Rule Verification:** It tests whether a firewall is **stateless** (only checking for `SYN` flags) or **stateful** (tracking overall TCP connection state).
* **Stealth & Evasion:** It verifies if intrusion detection systems (IDS) flag non-standard flag combinations or if packet filters silently drop anomalous traffic.

---

### Step-by-Step Next Steps to Disambiguate `open|filtered`

When Nmap returns `open|filtered`, it means: *"No packet came back. Either an open port dropped it per RFC 793, or a firewall dropped it before it reached the host."*

To eliminate this ambiguity and confirm the real state, follow this workflow:

#### Step 1: Force Banner Interaction with Version Detection (`-sV`)

Run version detection specifically on the `open|filtered` ports. `-sV` forces Nmap to send real application-level payloads (e.g., HTTP GET, SSH banner requests).

* **If the port is open:** The application responds to the payload, forcing Nmap to reclassify the port state from `open|filtered` directly to **`open`**.

```bash
sudo nmap -sV -p 22,80,443 10.10.10.15

```

#### Step 2: Cross-Reference with a TCP ACK Scan (`-sA`)

Run a TCP ACK scan against the same ports to test firewall filter status independently, then cross-reference the two outputs:

```bash
sudo nmap -sA -p 22,80,443 10.10.10.15

```

Apply this logical deduction matrix:

| Inverse Scan (`-sX`/`-sN`/`-sF`) | ACK Scan (`-sA`) | Deductive Conclusion | Why? |
| --- | --- | --- | --- |
| `open|filtered` | **`unfiltered`** | **OPEN** | ACK reached the target OS (returned `RST`), proving no firewall blocked it. Silence on the inverse scan confirms RFC 793 open behavior. |
| `open|filtered` | **`filtered`** | **FILTERED** | ACK was dropped by a firewall. The inverse scan was also dropped by the firewall, not the host. |

#### Step 3: Run a Standard TCP SYN Scan (`-sS`)

If stealth is no longer a constraint, execute a standard privileged SYN scan.

* **If the port is open:** Target replies with `SYN/ACK`.
* **If the port is filtered:** Target gives no reply or returns an ICMP Unreachable.

```bash
sudo nmap -sS -p 22,80,443 10.10.10.15

```

#### Step 4: Execute Targeted NSE Scripts (`-sC` or `--script`)

Target the suspicious ports using Nmap Scripting Engine (NSE) discovery scripts to force Layer 7 application responses:

```bash
sudo nmap -sC -p 22,80,443 10.10.10.15

```

---

**Key Insight:** An `open|filtered` status is resolved by pairing an inverse scan (which tests RFC 793 open-port behavior) with an ACK scan (which tests firewall filtering), allowing you to mathematically isolate open ports behind stateless firewalls.





**Key Insight:** An Xmas scan sets the FIN, PSH, and URG flags to exploit RFC 793 out-of-sequence processing, allowing stateless firewall evasion on Unix/Linux systems by expecting open ports to remain silent while closed ports return an RST.




