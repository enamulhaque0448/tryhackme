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

**Key Insight:** An `open|filtered` status is resolved by pairing an inverse scan (which tests RFC 793 open-port behavior) with an ACK scan (which tests firewall filtering), allowing you to mathematically isolate open ports behind stateless firewalls. An Xmas scan sets the FIN, PSH, and URG flags to exploit RFC 793 out-of-sequence processing, allowing stateless firewall evasion on Unix/Linux systems by expecting open ports to remain silent while closed ports return an RST.

---
### TCP Maimon Scan:
A **Maimon Scan (`-sM`)** is an inverse TCP stealth scan technique named after its discoverer, **Uriel Maimon** (who published the technique in *Phrack* Magazine in 1996).

It belongs to the same family of inverse stealth scans as **Null**, **FIN**, and **Xmas** scans, but uses a different flag combination: **`FIN/ACK`**.

---

### Mechanics & The BSD Stack Quirk

In a Maimon scan, Nmap crafts a raw TCP packet with both the **FIN (Finish)** and **ACK (Acknowledge)** flags set to `1`:

```text
TCP Header Flags: [ URG: 0 | ACK: 1 | PSH: 0 | RST: 0 | SYN: 0 | FIN: 1 ]

```

#### The Protocol Twist (RFC 793 vs. BSD Implementation):

* **Strict RFC 793 Standard:** According to official TCP specifications, sending an unsolicited `FIN/ACK` packet should trigger an **`RST` (Reset)** response regardless of whether the port is open or closed.
* **The Maimon Glitch:** Uriel Maimon discovered that many **BSD-derived networking stacks** (such as FreeBSD, NetBSD, OpenBSD, SunOS, HP-UX, and AIX) contained an implementation flaw. On these specific systems:
* **Closed Ports:** Send an **`RST`** packet back (per RFC 793).
* **Open Ports:** **Silently drop** the packet (ignoring RFC 793).



```text
                           ┌── [Port Closed] ──> Responds with: RST
[Tester] ── (FIN + ACK) ──>┤
                           └── [Port Open]   ──> Responds with: Nothing (Packet Dropped)

```
<img width="2400" height="1400" alt="image" src="https://github.com/user-attachments/assets/b64a39cc-5e7f-47f9-b8da-c8cf60de5f7f" />

---

### Response Matrix & Nmap Classification

Just like other inverse scans, because an **Open** port and a **Filtered** port (firewall block) both return silence, Nmap classifies non-responsive ports as **`open|filtered`**.

| Port State | Target Response (BSD Stacks) | Nmap Inferred Result |
| --- | --- | --- |
| **Closed** | `RST` Packet | `closed` |
| **Open** | *No Response* | `open|filtered` |
| **Filtered (Firewall)** | *No Response* or *ICMP Unreachable* | `open|filtered` / `filtered` |

---

### Advantages & Use Cases

* **Stateless Firewall Evasion:** Because it carries a `FIN` flag alongside an `ACK`, stateless firewalls expecting a standard connection sequence (`SYN`) often pass the packet through without logging.
* **Specialized OS Fingerprinting:** Because modern Linux and Windows stacks usually return an `RST` for both open and closed ports, a successful Maimon scan immediately indicates the target is running an older BSD-based operating system.
* **No Session Logs:** No 3-way handshake is established, ensuring application logs (like web or SSH daemons) record no connection.

---

### Critical Limitations

1. **Requires Privileged Access:** Crafting raw TCP packets with custom `FIN/ACK` flags requires root/sudo permissions (`sudo nmap -sM`).
2. **Obsolete on Modern Systems:** Most modern Linux, Windows, and updated BSD operating systems fixed this implementation bug. On modern systems, a Maimon scan will usually treat all ports as `closed` because the kernel sends an `RST` regardless of port state.

---

### Nmap Command Syntax

```bash
# Execute a Maimon scan against a target
sudo nmap -sM 10.10.10.15

# Maimon scan targeting specific web/SSH ports without DNS resolution
sudo nmap -sM -p 22,80,443 -n 10.10.10.15

```

---

### Complete Inverse Scan Comparison Matrix

| Scan Type | Nmap Flag | Header Flags Set | Target OS Dependency | Open Port Output |
| --- | --- | --- | --- | --- |
| **Maimon Scan** | `-sM` | `FIN, ACK` (`0x11`) | BSD-derived TCP stacks | `open|filtered` |

---

**Key Insight:** A Maimon scan sets both the `FIN` and `ACK` flags to exploit an OS implementation flaw in BSD-derived networking stacks, using non-responsiveness to infer open ports while serving as a historical fingerprinting vector for legacy Unix systems.
## TCP ACK, Window, and Custom Scan:
### 1. TCP ACK Scan (`-sA`)

* **Core Concept:** A specialized scanning technique designed strictly to **map firewall rulesets** and determine whether a firewall is **stateful** or **stateless**. It cannot determine if a port is open or closed.
* **Mechanism & Response Handling:**
* Sends an unsolicited TCP packet with only the **`ACK`** flag set (`0x10`).
* **Unfiltered Port (Open or Closed):** The target OS receives an `ACK` for an unestablished connection, gets confused, and returns a **`RST` (Reset)** packet. Nmap marks the port as `unfiltered`.
* **Filtered Port:** An inline firewall or ACL drops the packet (no response) or replies with an ICMP unreachable message. Nmap marks the port as `filtered`.


* **Key Features:**
* **Stateless Firewall Evasion:** Passes directly through simple, stateless firewalls that only block incoming `SYN` packets.
* **Firewall Mapping:** Allows security analysts to map out complex firewall access control lists (ACLs) by separating blocked ports from reachable ports.
* **Requires Privileged Access:** Uses raw sockets (`sudo nmap -sA`) to craft custom TCP headers.



---

### 2. TCP Window Scan (`-sW`)

* **Core Concept:** An advanced variant of the ACK scan that inspects the **TCP Window Size field** in returning `RST` packets to differentiate **open** ports from **closed** ports on vulnerable operating systems.
* **Mechanism & Response Handling:**
* Sends the exact same probe as a TCP ACK scan (a raw packet with the `ACK` flag enabled).
* Analyzes the 16-bit **Window Size** header field inside the returned `RST` packet:
* **Positive Window Size ($> 0$):** Port is classified as **`open`** (the target OS allocates memory/buffer space for active listening sockets).
* **Zero Window Size ($= 0$):** Port is classified as **`closed`**.
* **No Response / ICMP Error:** Port is classified as **`filtered`**.




* **Key Features:**
* **Exploits TCP Stack Anomalies:** Relies on implementation quirks present in specific networking stacks (e.g., older FreeBSD, AIX, OpenBSD, Z/OS, and certain embedded router firmwares).
* **Evasion Potential:** Detects open ports using an `ACK` packet rather than a standard `SYN` packet on susceptible targets.
* **OS Limitation:** Unreliable against modern, strictly RFC-compliant OS stacks (like modern Linux or Windows), which return a window size of `0` for all `RST` packets, incorrectly showing all ports as closed.



---

### 3. Custom TCP Scan (`--scanflags`)

* **Core Concept:** A flexible Nmap feature that grants complete control over the 8-bit TCP control flag byte, allowing users to craft raw probes with **any arbitrary combination of flags**.
* **Mechanism & Response Handling:**
* Custom flags are specified via text names (`--scanflags SYNFINPSH`) or direct hexadecimal bitmasks (`--scanflags 0x29`).
* Nmap combines custom flags with a base scan engine (such as `-sS` or `-sF`) to determine how it parses returned packets:
* **Base SYN Engine (`-sS`):** Expects `SYN/ACK` for open ports and `RST` for closed ports.
* **Base FIN/Inverse Engine (`-sF`):** Expects silence for open ports and `RST` for closed ports.




* **Key Features:**
* **IDS/IPS Evasion:** Easily bypasses signature-based Intrusion Detection Systems (like Snort or Suricata) that look for standard scan flag signatures (`SYN`, `FIN`, or `Xmas`).
* **Protocol Stack Research:** Allows security researchers to test how firewalls, middleboxes, and unknown OS stacks react to illegal or unusual flag combinations (e.g., `SYN` + `RST` or `URG` + `FIN`).
* **Full Customization:** Combines any sequence of `URG`, `ACK`, `PSH`, `RST`, `SYN`, and `FIN` flags into a single probe.



---

### Summary Comparison Table

| Scan Type | Nmap Flag | Packet Flags Sent | Key Returned Indicator | Primary Operational Objective |
| --- | --- | --- | --- | --- |
| **TCP ACK Scan** | `-sA` | `ACK` | `RST` vs. *No Response* | Map firewall filtering rules (`filtered` vs. `unfiltered`). |
| **TCP Window Scan** | `-sW` | `ACK` | `RST` Window Size ($>0$ vs. $=0$) | Detect open ports behind stateless firewalls via OS stack quirks. |
| **Custom Scan** | `--scanflags` | Arbitrary (User-defined) | Dictated by Base Engine | Bypass IDS/IPS signature rules and research exotic stack behavior. |

---

**Key Insight:** While an ACK scan maps firewall boundaries and a Window scan exploits stack buffer quirks in `RST` replies to infer open ports, Custom scans (`--scanflags`) bypass signature-based security devices by granting bit-level manipulation over the TCP control byte.
## Spoofing and Decoys:
### 1. IP Address Spoofing (`-S` and `-D`)

IP spoofing involves forging the **Source IP address** in the Layer 3 IP header of custom-crafted raw network packets. In Nmap, IP spoofing is implemented either as **Direct IP Spoofing** or **Decoy Scanning**.

#### A. Direct IP Spoofing (`-S`)

* **Core Mechanism:** Nmap constructs a raw IP packet and overwrites the default source IP with a fake IP address specified by the user.
* **The Asymmetric Return-Path Problem:**
* When the target receives the spoofed packet (e.g., a `SYN`), it sends the response packet (`SYN/ACK` or `RST`) back to the **spoofed IP address**, *not* to your actual machine.
* Because response packets never return to your network interface, Nmap cannot directly read open ports using `-S` unless you control the spoofed host, sniff the link passively, or conduct an **IDLE/Zombie scan (`-sI`)**.



```
[Attacker: 10.0.0.5] ── (SYN, Src: 10.0.0.99) ──> [Target: 10.0.0.1]
                                                          │
[Spoofed Host: 10.0.0.99] <── (SYN/ACK Response) ─────────┘

```

* **Command Syntax:**
```bash
# Spoof source IP as 10.0.0.99 targeting 10.0.0.1
# Note: Requires -e (interface) and -Pn (disable host discovery)
sudo nmap -S 10.0.0.99 -e eth0 -Pn 10.0.0.1

```



#### B. Decoy Scanning (`-D`)

* **Core Mechanism:** Instead of replacing your real IP completely, Nmap sends multiple probe packets simultaneously—some carrying fake (decoy) source IPs and one carrying your actual IP.
* **Security & Recon Purpose:** Obfuscates your real identity in target Intrusion Detection System (IDS) and Firewall logs. Security analysts reviewing log files see scan attempts originating from multiple hosts at once, making attribution extremely difficult.
* **Command Syntax:**
```bash
# Explicitly define decoy IPs alongside your real IP (ME)
sudo nmap -D 10.0.0.15,10.0.0.16,ME,10.0.0.17 10.0.0.1

# Generate 10 random decoy IP addresses automatically
sudo nmap -D RND:10 10.0.0.1

```



---

### 2. MAC Address Spoofing (`--spoof-mac`)

MAC address spoofing modifies the **Layer 2 (Data Link) Ethernet Frame Header** before it is transmitted over the wire.

```
+-------------------------------------------------------------------+
| Ethernet Header (Layer 2)      | IP Header (Layer 3) | TCP Payload |
| Src MAC: [ Spoofed MAC ]       | Src IP: [ Real IP ] | ...         |
+-------------------------------------------------------------------+

```

#### Core Mechanisms & Features

* **Bypassing Network Access Control (NAC):** Network switches and wireless access points often use MAC filtering to grant network access. Spoofing an authorized MAC (such as an approved corporate printer or VoIP phone) bypasses these controls.
* **Evading Vendor Fingerprinting:** Security tools flag suspicious hardware vendors. By spoofing a target-compliant OUI (Organizationally Unique Identifier), your scan traffic resembles legitimate hardware on the wire.
* **Local Subnet Limitation:** MAC address headers are stripped and rewritten at every Layer 3 router hop. Therefore, MAC spoofing only functions when scanning targets within the **same local broadcast domain (LAN)**.

#### Command Syntax Options

* **Spoof a Specific MAC Address:**
```bash
sudo nmap --spoof-mac 00:11:22:33:44:55 192.168.1.1

```


* **Spoof by Vendor Name (Auto-selects Vendor OUI):**
```bash
# Spoofs MAC prefix assigned to Apple, Cisco, HP, Dell, etc.
sudo nmap --spoof-mac Apple 192.168.1.1
sudo nmap --spoof-mac Cisco 192.168.1.1

```


* **Generate a Completely Random MAC:**
```bash
# Using '0' instructs Nmap to generate a fully randomized valid MAC
sudo nmap --spoof-mac 0 192.168.1.1

```



---

### Layer 2 vs. Layer 3 Spoofing Matrix

| Parameter | MAC Address Spoofing (`--spoof-mac`) | IP Address Spoofing (`-S`) | Decoy Scan (`-D`) |
| --- | --- | --- | --- |
| **OSI Layer** | **Layer 2** (Data Link) | **Layer 3** (Network) | **Layer 3** (Network) |
| **Primary Scope** | Local Subnet (LAN) | Across Routers / Internet | Across Routers / Internet |
| **Primary Goal** | Bypass MAC filtering / Hide NIC hardware identity | Hide true origin host | Blend real scan inside decoy log noise |
| **Receives Replies?** | **Yes** (Network switches deliver frames back to local NIC) | **No** (Replies go to the spoofed IP) | **Yes** (Replies to real IP arrive normally) |
| **Privilege Requirement** | Requires `sudo` / `root` | Requires `sudo` / `root` | Requires `sudo` / `root` |

---

### Defensive Controls & Limitations

1. **Reverse Path Forwarding (RPF / BCP 38):** Modern border routers drop IP packets whose source IP does not match the ingress interface route table, stopping cross-internet IP spoofing at the ISP level.
2. **Dynamic ARP Inspection (DAI) & Port Security:** Managed switches cross-reference MAC addresses against DHCP snooping binding tables; unauthorized MAC shifts trigger automatic port shutdown (`err-disable`).
3. **Stateful Firewalls:** Drop incoming response traffic from spoofed IPs if no matching connection entry exists in the state table.

---

**Key Insight:** MAC spoofing operates at Layer 2 to bypass local hardware controls while preserving response traffic, whereas IP spoofing operates at Layer 3 to blind attribution, requiring decoy scans (`-D`) or zombie hosts (`-sI`) to maintain bidirectional visibility.
## Fragmented Packets:
# Firewall Evasion via Packet Fragmentation

This module covers the core concepts of firewalls, Intrusion Detection Systems (IDS), and how packet fragmentation parameters can be used within Nmap to bypass restrictive pattern filters.

---

## 🛡️ Network Defense Mechanisms

*   **Firewall:** A software or hardware barrier that allows or blocks network packets based on predefined rules. Traditional firewalls inspect IP and transport layer headers (ports). Advanced versions analyze the upper-layer payload.
*   **Intrusion Detection System (IDS):** Deeply inspects packet header data and transport layer payloads. It scans for specific behavioral patterns or malicious content signatures, triggering administrative alerts when a rule is matched.

---

## 🗜️ Packet Fragmentation Mechanics

To reduce the likelihood of traditional security devices detecting scanning activities, payloads can be broken down into smaller pieces. This splits predictable header patterns across multiple transmission blocks.

### Nmap Fragmentation Controls

| Nmap Option | Behavior & Data Split | Use Case |
| :--- | :--- | :--- |
| `-f` | Divides the IP payload data into blocks of **8 bytes** or fewer. | Splits a standard 24-byte TCP header across **3 IP fragments**. |
| `-ff` (or `-f -f`) | Divides the IP payload data into blocks of **16 bytes** or fewer. | Splits a standard 24-byte TCP header across **2 IP fragments** (16B + 8B). |
| `--mtu <NUM>` | Manually overrides the Maximum Transmission Unit size. | *Note: The customized input value must always be a **multiple of 8**.* |
| `--data-length <NUM>` | Appends a custom number of random bytes to packets. | Artificially inflates packet sizes to make traffic appear benign. |

### IP Header Reassembly Fields
According to RFC 791, when a destination system receives fragmented network packets, it utilizes two critical fields within the IP header to correctly piece the payload back together:
1.  **Identification (ID):** Ensures all split fragments belonging to the same original packet share a matching ID tracking number.
2.  **Fragment Offset:** Tells the receiving system the exact sequential position where the specific fragment's data belongs.

---

## 🖥️ Command Execution Example

To execute a stealth TCP SYN scan against a specific target port while enforcing 8-byte network packet fragmentation, use:

```bash
sudo nmap -sS -p80 -f <TARGET_IP>
```

*When reviewing this exact traffic pattern inside a network analyzer like **Wireshark**, you will observe multiple sequential fragments sharing the same IP ID before the final transmission layer is fully evaluated.*
<img width="1305" height="456" alt="image" src="https://github.com/user-attachments/assets/74dc9367-9be5-460a-bc4c-fc9ae94047b5" />

# Nmap Advanced Reference Guide: Flags & Command Options

This cheat sheet serves as a quick reference for advanced scanning patterns, spoofing configurations, and verbosity parameters covered within this module.

---

## 🔍 Advanced Port Scanning Matrix

These scan types rely on setting TCP flags in unexpected ways to prompt target ports for an identifiable reply. 

*   **Null, FIN, and Xmas Scans:** Provoke a response exclusively from **closed** ports (open ports ignore them).
*   **Maimon, ACK, and Window Scans:** Provoke a unique response from **both open and closed** ports to map structures.

| Port Scan Type | Example Command Syntax | Core Header Modification |
| :--- | :--- | :--- |
| **TCP Null Scan** | `sudo nmap -sN 10.49.134.130` | Sends a packet with **no flags set** at all. |
| **TCP FIN Scan** | `sudo nmap -sF 10.49.134.130` | Sends a packet with **only the FIN flag** set. |
| **TCP Xmas Scan** | `sudo nmap -sX 10.49.134.130` | Sets **FIN, PSH, and URG** flags together. |
| **TCP Maimon Scan** | `sudo nmap -sM 10.49.134.130` | Sets **FIN and ACK** flags together. |
| **TCP ACK Scan** | `sudo nmap -sA 10.49.134.130` | Sets **only the ACK flag** to map firewall rules. |
| **TCP Window Scan** | `sudo nmap -sW 10.49.134.130` | Checks the **TCP Window field size** inside `RST` replies. |
| **Custom TCP Scan** | `sudo nmap --scanflags URGACKPSHRSTSYNFIN 10.49.134.130` | Allows manual specification of any custom flag layout. |

---

## 🥷 Spoofing, Decoys & Evasion Parameters

| Evasion Type / Option | Flag Argument Example | Operational Purpose |
| :--- | :--- | :--- |
| **Spoofed Source IP** | `sudo nmap -S SPOOFED_IP 10.49.134.130` | Forges the sender source IP address in headers. |
| **Spoofed MAC Address** | `--spoof-mac SPOOFED_MAC` | Forges the sender hardware MAC identity layer. |
| **Decoy Scan** | `nmap -D DECOY_IP,ME 10.49.134.130` | Blends your real IP within a group of fake ones. |
| **Idle (Zombie) Scan** | `sudo nmap -sI ZOMBIE_IP 10.49.134.130` | Scans a target entirely via a quiet third-party host. |
| **8-Byte Fragmentation** | `-f` | Splits the IP data payload into blocks of **8 bytes**. |
| **16-Byte Fragmentation**| `-ff` | Splits the IP data payload into blocks of **16 bytes**. |
| **Custom Source Port** | `--source-port PORT_NUM` | Forces Nmap to send probes from a specific port number. |
| **Data Padding** | `--data-length NUM` | Appends random, non-malicious data bytes to change payload sizes. |

---

## 📊 Diagnostics, Logging & Verbosity Flags

To understand exactly how and why Nmap arrives at its final state classification, append these diagnostic parameters to your execution string:

*   `--reason`: Explicitly explains the exact packet response rule (e.g., `syn-ack`, `rst`, `no-response`) that led Nmap to make its final conclusion.
*   `-v`: Enables standard **verbose** output, printing findings dynamically during runtime execution.
*   `-vv`: Enables **very verbose** output, supplying deeper diagnostic logs and real-time state transitions.
*   `-d`: Activates core engine **debugging** output, helping diagnose script execution crashes or drops.
*   `-dd`: Maximizes the detail levels for structural **debugging** diagnostics.





