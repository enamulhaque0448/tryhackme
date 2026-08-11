#### 01. Subnet's work and ARP Request
---
### What the Bitwise AND Operation Actually Calculates

When a device wants to send data, it does **not** compare the result of the bitwise AND operation to the *subnet mask*.

Instead, it applies the subnet mask to **both** the destination IP address and its own IP address:

1. **Local Network Calculation:** `[Sender IP] AND [Subnet Mask]` = **Sender's Network ID**
2. **Destination Network Calculation:** `[Destination IP] AND [Subnet Mask]` = **Destination's Network ID**

The host then compares the **two resulting Network IDs**:

* **If Sender Network ID == Destination Network ID:**
The target is on the **same local network**. The device delivers the packet directly using **Layer 2 (Ethernet MAC addresses via ARP)** without passing through a router.
* **If Sender Network ID ≠ Destination Network ID:**
The target is on a **different network** (either another internal subnet or the public internet). The device encapsulates the packet and sends it to the **Default Gateway (Layer 3 Router)**.

---

### Step-by-Step Practical Example

Assume **Host A** has IP `192.168.1.50` with subnet mask `255.255.255.0` (`/24`).

Its calculated local Network ID is **`192.168.1.0`**.

#### Scenario 1: Host A sends data to `192.168.1.100`

```text
Destination IP:   192.168.1.100  (11000000.10101000.00000001.01100100)
Subnet Mask:      255.255.255.0  (11111111.11111111.11111111.00000000)
----------------------------------------------------------------------
Result (AND):     192.168.1.0    (11000000.10101000.00000001.00000000)

```

* **Comparison:** Result (`192.168.1.0`) **equals** Host A's Network ID (`192.168.1.0`).
* **Action:** Direct local delivery via ARP/Layer 2 switch (No router needed).

#### Scenario 2: Host A sends data to `8.8.8.8` (Google DNS)

```text
Destination IP:   8.8.8.8        (00001000.00001000.00001000.00001000)
Subnet Mask:      255.255.255.0  (11111111.11111111.11111111.00000000)
----------------------------------------------------------------------
Result (AND):     8.8.8.0        (00001000.00001000.00001000.00000000)

```

* **Comparison:** Result (`8.8.8.0`) **does not equal** Host A's Network ID (`192.168.1.0`).
* **Action:** Forward traffic to the Default Gateway (`192.168.1.1`) for Layer 3 routing.

---

###  Summary 

> **Summary:** The subnet mask separates an IP address into a Network ID and a Host ID. A device performs a bitwise AND operation on the destination IP using its subnet mask. If the resulting **Network ID matches its own Network ID**, the traffic is sent directly on the local network via Layer 2; if the **Network IDs do not match**, the packet is forwarded to the Default Gateway for routing.

**Key Insight:** The bitwise AND operation produces a **Network ID**, not the subnet mask itself; hosts decide local vs. remote delivery by checking if the destination's Network ID matches their own.
**Address Resolution Protocol (ARP)** is a core Layer 2 protocol used to map a known **IP address (Layer 3 Logical Address)** to an unknown **MAC address (Layer 2 Physical Address)** on the same local area network (LAN).

Routers and switches use IP addresses to direct packets across networks, but local Ethernet network interface cards (NICs) require a physical MAC address to deliver data frames across switches and wires. ARP bridges this gap.

---

### Step-by-Step Breakdown: How ARP Works

Imagine **Host A** (`192.168.1.10`) wants to send data to **Host B** (`192.168.1.20`) on the same subnet.

```
       [ Host A ]                                                   [ Host B ]
   IP: 192.168.1.10                                             IP: 192.168.1.20
MAC: AA:AA:AA:AA:AA:AA                                      MAC: BB:BB:BB:BB:BB:BB
           │                                                           │
           │─────── 1. Check Local ARP Cache (Miss) ───────────────────│
           │                                                           │
           │─────── 2. ARP Request (Broadcast: FF:FF:FF:FF:FF:FF) ────►│ (All Hosts Receive)
           │        "Who has 192.168.1.20? Tell 192.168.1.10"         │
           │                                                           │
           │◄────── 3. ARP Reply (Unicast: AA:AA:AA:AA:AA:AA) ─────────│ (Host B Responds)
           │        "192.168.1.20 is at BB:BB:BB:BB:BB:BB"             │
           │                                                           │
           │─────── 4. Update ARP Cache & Send IP Packets ────────────►│

```

#### Step 1: Check the ARP Cache

Before sending anything over the wire, Host A checks its local **ARP Cache** (a temporary in-memory table storing IP-to-MAC mappings).

* **If a match exists:** Host A immediately encapsulates the IP packet into an Ethernet frame using Host B's MAC address and sends it.
* **If no match exists (ARP Miss):** Host A initiates an ARP resolution request.

#### Step 2: Send the ARP Request (Broadcast)

Host A builds an **ARP Request** packet with the following header details:

* **Sender IP:** `192.168.1.10` | **Sender MAC:** `AA:AA:AA:AA:AA:AA`
* **Target IP:** `192.168.1.20` | **Target MAC:** `00:00:00:00:00:00` (Unknown)
* **Destination Ethernet Address:** `FF:FF:FF:FF:FF:FF` (**Layer 2 Broadcast**)

The switch receives this frame and floods it to **every connected device** on the local network segment.

#### Step 3: Receive & Process the ARP Request

Every host on the local network receives the broadcast:

* **Other Hosts (`192.168.1.30`, etc.):** Compare the target IP (`192.168.1.20`) with their own IP. Since it does not match, they drop the frame silently.
* **Host B (`192.168.1.20`):** Recognizes its own IP address. It caches Host A’s IP-to-MAC mapping in its own ARP table for future use.

#### Step 4: Send the ARP Reply (Unicast)

Host B creates an **ARP Reply** packet containing its MAC address:

* **Sender IP:** `192.168.1.20` | **Sender MAC:** `BB:BB:BB:BB:BB:BB`
* **Target IP:** `192.168.1.10` | **Target MAC:** `AA:AA:AA:AA:AA:AA`
* **Destination Ethernet Address:** `AA:AA:AA:AA:AA:AA` (**Layer 2 Unicast**)

This frame is sent directly back to Host A through the switch.

#### Step 5: Update Cache & Transmit Data

Host A receives the ARP Reply, stores `192.168.1.20 -> BB:BB:BB:BB:BB:BB` in its ARP Cache, and begins transmitting the actual data traffic.

---

### Useful CLI Commands

To inspect or clear your ARP table on Linux or Windows:

* **View ARP Table:**
```bash
# Linux
ip neighbor show   # or: arp -a

# Windows
arp -a

```


* **Clear ARP Cache:**
```bash
# Linux
sudo ip neighbor flush all

# Windows (Run as Admin)
netsh interface ip delete arpcache

```



---

### Cybersecurity & Ethical Hacking Link

Because standard ARP lacks authentication mechanisms, it is inherently vulnerable to exploitation on local networks:

* **ARP Spoofing / ARP Poisoning:** An attacker sends forged (unsolicited) ARP Replies to the local network claiming that the Default Gateway's IP belongs to the attacker's MAC address.
* **Man-In-The-Middle (MITM):** By poisoning both the target host and the router, the attacker forces all local network traffic to route through their machine first, enabling packet sniffing (via Wireshark), SSL stripping, or session hijacking using tools like `Ettercap` or `Bettercap`.
* **Mitigation:** Static ARP entries, **Dynamic ARP Inspection (DAI)** on enterprise switches, and 802.1X port security.

---

> **Key Insight:** ARP translates Layer 3 logical IP addresses into Layer 2 physical MAC addresses via broadcast requests and unicast replies, operating entirely trust-based without intrinsic authentication.

---
### 02. Host Discovery via TCP/IP Layer
---
<img width="1200" height="800" alt="image" src="https://github.com/user-attachments/assets/835a7356-5156-4013-a90a-28e2bf701690" />
**Internet Control Message Protocol (ICMP)** is a core supporting protocol in the Internet Protocol suite (IP) operating at the **Network Layer (Layer 3)**.

Unlike TCP or UDP, ICMP is **not used to transport application data** (like web pages or emails). Instead, it is used by network devices (routers, firewalls, and hosts) to send **diagnostic information, operational queries, and error messages** regarding IP packet delivery.

---

### How ICMP Works Under the Hood

ICMP messages are encapsulated directly inside standard **IP packets** (IP Protocol number `1`).

When a router or host encounters an issue delivering an IP packet, or when a network diagnostic tool requests information, an ICMP packet is generated and sent back to the original source IP address.

```
       +-------------------------------------------------------+
       |                  IP Header (Layer 3)                  |
       |  Source IP | Destination IP | Protocol: 1 (ICMP)      |
       +-------------------------------------------------------+
       |                  ICMP Data Payload                    |
       |  Type Field | Code Field | Checksum | Payload Data    |
       +-------------------------------------------------------+

```

---

### Core ICMP Message Header Fields

Every ICMP packet contains three primary control fields:

1. **Type (8 bits):** Defines the general category or purpose of the ICMP message.
2. **Code (8 bits):** Provides specific sub-details or reasons for the message Type.
3. **Checksum (16 bits):** Verifies the integrity of the ICMP header and data.

---

### Common ICMP Types & Real-World Use Cases

| ICMP Type | Name / Purpose | Common Code Values | Real-World Utility |
| :--- | :--- | :--- | :--- |
| **Type 8 / Type 0** | **Echo Request / Echo Reply** | Code `0` | Used directly by the **`ping`** utility to test reachability and round-trip time (RTT). |
| **Type 3** | **Destination Unreachable** | Code `0` (Net Unreachable)<br>Code `1` (Host Unreachable)<br>Code `3` (Port Unreachable)<br>Code `13` (Administratively Prohibited / Firewall) | Generated by routers/firewalls when a packet cannot reach its destination port, host, or network segment. |
| **Type 11** | **Time Exceeded** | Code `0` (TTL Expired in Transit) | Generated when an IP packet's **Time to Live (TTL)** counter drops to `0`. This mechanism forms the foundation of the **`traceroute`** utility. |
| **Type 5** | **Redirect Message** | Code `0` (Redirect Datagram for the Network) | Informs a sender host to update its local routing table to use a more optimal gateway router. |

### How Windows ping Translates ICMP Header Fields

When an ICMP packet does return, Windows parses the binary Type and Code fields and displays human-readable text:

| ICMP Type | ICMP Code | Raw Meaning | What Windows ping Displays |
| :--- | :--- | :--- | :--- |
| **Type 0** | Code 0 | Echo Reply | `Reply from 8.8.8.8: bytes=32 time=30ms TTL=117` |
| **Type 3** | Code 1 | Host Unreachable | `Destination host unreachable.` |
| **Type 3** | Code 13 | Communication Prohibited (Firewall) | `Destination net unreachable.` |
| **Type 11** | Code 0 | TTL Expired in Transit | `TTL expired in transit.` |
| *None* | *None* | No packet returned (Timeout) | `Request timed out.` |


| Scenario | Location | ARP Cache Status | Action Taken |
| :---: | :--- | :--- | :--- |
| **1** | Same Subnet | Not Cached | Broadcasts ARP for Target IP |
| **2** | Same Subnet | Cached | No ARP (Uses Target MAC from cache) |
| **3** | Remote Subnet | Not Cached | Broadcasts ARP for Gateway IP |
| **4** | Remote Subnet | Cached | No ARP (Uses Gateway MAC from cache) |

---

### Step-by-Step Examples: Diagnostic Tools in Action

#### 1. How `ping` Works (Echo Request / Reply)

* **Step 1:** Host A sends an **ICMP Type 8 (Echo Request)** packet to Host B (`ping 192.168.1.1`).
* **Step 2:** Host B receives the request and sends back an **ICMP Type 0 (Echo Reply)** packet to Host A.
* **Step 3:** Host A calculates the round-trip delay time based on the timestamp.

```
[ Host A ] ────── ICMP Type 8 (Echo Request) ──────► [ Host B ]
[ Host A ] ◄────── ICMP Type 0 (Echo Reply) ───────── [ Host B ]

```

#### 2. How `traceroute` Works (TTL Expiration)

* **Step 1:** Host A sends a UDP or ICMP packet with a **`TTL = 1`** to the destination.
* **Step 2:** Hop 1 (First Router) decrements TTL to `0`, drops the packet, and sends an **ICMP Type 11 (Time Exceeded)** back to Host A.
* **Step 3:** Host A logs Router 1's IP address, then sends a new packet with **`TTL = 2`** to discover Hop 2.
* **Step 4:** This process repeats sequentially until the destination host is reached.

---

### Cybersecurity & Ethical Hacking Perspectives

* **Ping Sweeps (Network Reconnaissance):** Attackers and pentesters send ICMP Echo Requests across an entire IP subnet (`nmap -sn 192.168.1.0/24`) to discover active live hosts rapidly.
* **ICMP Flood (Denial of Service - DoS):** Overwhelming a target server with massive volumes of ICMP Echo Requests, causing high CPU load and bandwidth exhaustion.
* **ICMP Tunneling (Covert Channel / Exfiltration):** Since ICMP packets allow custom data payloads, attackers can encapsulate covert data (C2 communications or exfiltrated files) inside ICMP request bodies to bypass basic port-based firewalls.
* **Ping of Death:** Legacy attack vector involving malformed/oversized ICMP packets ($>65,535\text{ bytes}$) designed to crash vulnerable network stack implementations.

---

**Key Insight:** ICMP acts as the diagnostic feedback mechanism for IP routing—it does not transfer application data, but enables hosts and routers to report delivery failures, measure latency, and trace network paths.

---
### 03. Enumerating Target
---
* *To find the binary representation and subnet details for 10.10.12.13/29, we convert each octet of the IP address into an 8-bit binary number and determine the network boundary based on the /29 prefix.*
## 1. Complete Binary Representation

| Component | Octet 1 | Octet 2 | Octet 3 | Octet 4 |
|---|---|---|---|---|
| Decimal IP | 10 | 10 | 12 | 13 |
| Binary IP | 00001010 | 00001010 | 00001100 | 00001101 |
| Subnet Mask (/29) | 11111111 | 11111111 | 11111111 | 11111000 |


* Full Binary String: 00001010.00001010.00001100.00001101
* Subnet Mask (Decimal): 255.255.255.248

------------------------------
## 2. Breaking Down the Math (Your Note)
Your calculation steps resolve the host bits and host counts for this subnet:

* $32 - 29 = 3$: This means you have exactly 3 host bits remaining in the last octet.
* $2^3 = 8$: Total size of the block increment (addresses per subnet).
* $8 - 2 = 6$: There are exactly 6 usable host IP addresses (subtracting the Network ID and Broadcast ID).

------------------------------
## 3. Subnet Range Analysis
Because the block size is 8, subnets in the 4th octet increment by 8s: 0, 8, 16, 24...
The IP .13 falls directly inside the 8 to 15 block.

| Property | Decimal Value | Binary Value (4th Octet Only) |
|---|---|---|
| Network ID | 10.10.12.8 | 00001[000] (First 5 bits locked) |
| First Usable | 10.10.12.9 | 00001[001] |
| Your IP Address | 10.10.12.13 | 00001[101] |
| Last Usable | 10.10.12.14 | 00001[110] |
| Broadcast ID | 10.10.12.15 | 00001[111] (Host bits all 1s) |
> calculating the complete address 10.10.12.9, you must take all 8 bits of the octet to represent the full number, but only the unlocked 3 bits are allowed to change.

## The total number of IP addresses in the range **10.10.0-255.101-125** is **6,400**.

---

### Why Your CIDR Math ($32 - 16 = 16$) Caused Confusion

* **CIDR Subnet Math ($2^H$):** The formula $32 - \text{Prefix} = \text{Host Bits } (H)$ applies when dealing with standard CIDR block masks (like `/16` or `/24`) where host bits fill the entire remaining binary space ($0$ to $255$).
* **Range/Dash Notation:** `10.10.0-255.101-125` is a **custom IP target specification** (commonly used in tools like Nmap or Masscan), not a single contiguous CIDR block. Because the 4th octet is restricted to a slice of 25 numbers (`101` to `125`) instead of all 256 numbers (`0` to `255`), you cannot use $2^H$ bitwise power-of-two math. You must calculate the possible combinations per octet.

---

### Step-by-Step Combination Calculation

To find the total number of IPs, multiply the count of valid numbers in each octet:

* **Octet 1 (`10`):** Exactly **1** value (`10`)
* **Octet 2 (`10`):** Exactly **1** value (`10`)
* **Octet 3 (`0-255`):** All values from 0 through 255 inclusive
$$\text{Count} = 255 - 0 + 1 = 256\text{ values}$$


* **Octet 4 (`101-125`):** All values from 101 through 125 inclusive
$$\text{Count} = 125 - 101 + 1 = 25\text{ values}$$



$$\text{Total IPs} = 1 \times 1 \times 256 \times 25 = 6,400$$

---

### Target Scope Comparison

| Range Type | Notation Example | Total IP Count | Calculation Method |
| --- | --- | --- | --- |
| **Full Subnet (CIDR /16)** | `10.10.0.0/16` | $65,536$ | $2^{32-16} = 2^{16} = 65,536$ |
| **Custom Target Range** | `10.10.0-255.101-125` | $6,400$ | $1 \times 1 \times 256 \times 25 = 6,400$ |

> **Key Insight:** CIDR notation ($2^H$) calculates full contiguous subnets along binary boundaries, while dash/range notation ($N_1 \times N_2 \times N_3 \times N_4$) calculates exact target combinations across custom IP slices.
## 04. Nmap Host Discovery Using ARP

<img width="948" height="356" alt="image" src="https://github.com/user-attachments/assets/be3ab3b0-c774-42be-8549-b8cb32d5c151" />

> **Flag Parameters Used:**
> * `-PR`: Explicitly instructs Nmap to issue local ARP Discovery Requests.
> * `-sn`: Forces Host Discovery Only, disabling subsequent automated port scans to save execution time.

### 1. Default Nmap Behavior

* Nmap always tries to find live hosts first via a ping scan.
* It only runs a detailed port scan on systems that respond to the initial ping sweep.
* **The Optimization Flag:** Running `nmap -sn TARGETS` tells Nmap to discover online hosts without performing any subsequent port scanning.

### 2. Privileged vs. Unprivileged Users

Nmap alters its strategy based on your account permissions and target location:

* **Local Network (Sudo/Root):** Uses ARP requests exclusively.
* **Outside Local Network (Sudo/Root):** Uses a combination of ICMP echo requests, TCP ACK to port 80, TCP SYN to port 443, and ICMP timestamp requests.
* **Outside Local Network (Standard User):** Resorts to a standard TCP 3-way handshake by sending SYN packets directly to ports 80 and 443.

### 3. ARP Scan Constraints (`-PR`)

* ARP scanning only functions if you are on the same local subnet as your target.
* Because network devices require a hardware MAC address to communicate across a link layer, any host that replies to an ARP query is classified as alive.
* The explicit command to isolate an ARP scan without any port enumeration is `nmap -PR -sn TARGETS`.

---

## 05. Nmap ICMP Host Discovery Techniques

### 1. ICMP Echo Scan (`-PE`)

* **Mechanism:** Sends ICMP Type 8 (Echo Request) packets and listens for ICMP Type 0 (Echo Reply) responses.
* **Limitation:** Unreliable on modern networks. Firewalls and default Windows host configurations commonly block ICMP Echo packets.
* **Subnet Rule:** If the target is on your local subnet, an ARP query will automatically happen before the ICMP request goes out.
* **Command Syntax:** `nmap -PE -sn TARGETS`

---

### 2. ICMP Timestamp Scan (`-PP`)

* **Mechanism:** Uses ICMP Type 13 (Timestamp Request) packets and looks for ICMP Type 14 (Timestamp Reply) responses.
* **Utility:** Serves as an alternative backup discovery method when firewalls actively filter out standard ICMP Echo pings.
* **Command Syntax:** `nmap -PP -sn TARGETS`

---

### 3. ICMP Address Mask Scan (`-PM`)

* **Mechanism:** Sends ICMP Type 17 (Address Mask Request) queries and expects ICMP Type 18 (Address Mask Reply) responses.
* **Limitation:** Frequently dropped or entirely blocked by modern transit firewalls and host operating systems.
* **Command Syntax:** `nmap -PM -sn TARGETS`



## 06. Nmap TCP/UDP Host Discovery

## Host Discovery Techniques

| Scan Type | Example Command | Purpose / Protocol |
| :--- | :--- | :--- |
| **ARP Scan** | `sudo nmap -PR -sn 10.200.6.0/24` | Layer 2 discovery for local subnets (fastest and most accurate). |
| **ICMP Echo Scan** | `sudo nmap -PE -sn 10.200.6.0/24` | Standard ICMP Echo Request (`Type 8`). |
| **ICMP Timestamp Scan** | `sudo nmap -PP -sn 10.200.6.0/24` | Requests host system timestamp (`Type 13`); bypasses basic ICMP blocks. |
| **ICMP Address Mask Scan** | `sudo nmap -PM -sn 10.200.6.0/24` | Requests subnet mask (`Type 17`); legacy alternative when Echo is filtered. |
| **TCP SYN Ping Scan** | `sudo nmap -PS22,80,443 -sn 10.200.6.0/30` | Sends TCP SYN packets to target ports (e.g., 22, 80, 443). |
| **TCP ACK Ping Scan** | `sudo nmap -PA22,80,443 -sn 10.200.6.0/30` | Sends TCP ACK packets to bypass simple stateless firewalls. |
| **UDP Ping Scan** | `sudo nmap -PU53,161,162 -sn 10.200.6.0/30` | Sends UDP probes to trigger ICMP Port Unreachable responses. |



## Essential Flag Options

| Option | Purpose | Operational Benefit |
| :--- | :--- | :--- |
| `-sn` | **Host Discovery Only** | Disables port scanning phase for faster target sweeps. |
| `-n` | **Disable DNS Resolution** | Prevents reverse DNS lookups to speed up scans and lower detection noise. |
| `-R` | **Force Reverse DNS** | Resolves DNS names for all target IPs (useful for host classification). |



<img width="665" height="234" alt="image" src="https://github.com/user-attachments/assets/97419025-095e-42c0-8189-0dcbbee27a07" />

<img width="635" height="237" alt="image" src="https://github.com/user-attachments/assets/4d9fa6ed-a343-4d0f-b4c5-bb99f6e5176d" />

<img width="716" height="236" alt="image" src="https://github.com/user-attachments/assets/c240c459-94ae-4008-9947-0c8f7435d5d4" />

<img width="900" height="556" alt="image" src="https://github.com/user-attachments/assets/ce08269e-3710-4d62-bca3-9d4e37bb5351" />




















