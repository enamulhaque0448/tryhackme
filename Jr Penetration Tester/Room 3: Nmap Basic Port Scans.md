<img width="1522" height="275" alt="image" src="https://github.com/user-attachments/assets/dc3fbb8d-7f31-4b48-9bc0-b091537e04ee" />
<img width="1493" height="599" alt="image" src="https://github.com/user-attachments/assets/6cf10341-6b78-4520-b5b0-43555cfbe1bd" />
<img width="1516" height="303" alt="image" src="https://github.com/user-attachments/assets/a960f369-3cea-4e99-8c01-3d0abcbc8b96" />
<img width="1507" height="459" alt="image" src="https://github.com/user-attachments/assets/0a76f249-3f97-403e-9a2b-5a3256e1aaf6" />

---
In the following Wireshark packet capture window, we see Nmap sending TCP packets with the SYN flag set to various ports, 5900, 22, 80, and so on. By default, 
Nmap will attempt to connect to the 1000 most common ports.**A closed TCP port responds to a SYN packet with RST/ACK to indicate that it is not open.**
This pattern will repeat for all the closed ports as we attempt to initiate a TCP 3-way handshake with them.
<img width="985" height="575" alt="image" src="https://github.com/user-attachments/assets/ced14697-2742-4f7d-8f43-a45536251f63" />

---
We notice that port 80 is open, so it replied with a SYN/ACK, and Nmap completed the 3-way handshake by sending an ACK.
The figure below shows all the packets exchanged between our Nmap host and the target system’s port 80. 
The first three packets are the TCP 3-way handshake. Then the fourth packet tears it down with an RST/ACK.
---
<img width="990" height="210" alt="image" src="https://github.com/user-attachments/assets/5c09e350-b0ad-43ef-9638-5517167f3fe5" />



# TCP SYN Scan vs. TCP ACK Scan

The main difference between a TCP SYN scan and a TCP ACK scan is how they treat target ports and what they are trying to discover. A SYN scan is used to find open ports, while an ACK scan is used to find firewall filtering rules.

---

## 🧭 Visualizing the Core Mechanism

### 1. TCP SYN Scan (`-sS` or `-PS`)

Known as a **Half-Open scan**, it mimics the very first step of starting a new connection.

```text
                  ┌── [Target Port Open]   ──> Responds with: SYN/ACK
[Tester] ── SYN ─>┤
                  └── [Target Port Closed] ──> Responds with: RST

```

* **How it works:** You send a packet pretending you want to open a brand-new connection (`SYN`).
* **The Response:** An open port welcomes you with a `SYN/ACK`. A closed port rejects you with an `RST` (Reset).
* **Main Function:** Maps out exactly which services are actively listening and open on a target machine.

---

### 2. TCP ACK Scan (`-sA` or `-PA`)

Known as an **Unsolicited Response scan**, it mimics a packet that is already part of an ongoing conversation.

```text
                  ┌── [No Firewall Block] ──> Responds with: RST (Port open OR closed)
[Tester] ── ACK ─>┤
                  └── [Firewall Present]  ──> Responds with: Nothing (Packet Dropped)

```

* **How it works:** You send a packet stating you are continuing an old connection (`ACK`). Because the target has no record of this connection, it gets confused.
* **The Response:** If the path is clear, the target OS kernel sends an `RST` back (regardless of whether that specific port is open or closed). If a firewall blocks the path, the packet gets dropped entirely, and you get no response.
* **Main Function:** Maps out firewall rules and determines if ports are **Filtered** or **Unfiltered**. It does *not* tell you if a port is open.

---

## 📊 Summary Comparison Table

| Feature | TCP SYN Scan | TCP ACK Scan |
| --- | --- | --- |
| **Primary Goal** | Find Open Ports | Map Firewall Rules |
| **Packet Sent** | `SYN` (Let's connect) | `ACK` (Continuing our talk) |
| **Open Port Reply** | `SYN/ACK` | `RST` |
| **Closed Port Reply** | `RST` | `RST` |
| **Firewall Blocked** | Confirms state as `Filtered` | Confirms state as `Filtered` |
| **Nmap Output Result** | Lists ports as `open` or `closed` | Lists ports as `unfiltered` or `filtered` |

## The fundamental difference between a privileged (`root`/`sudo`) scan and an unprivileged (standard user) scan in Nmap comes down to **Raw Socket Access**.

* Privileged users can construct and send custom network packets directly through **raw sockets**, whereas unprivileged users are restricted by the Operating System kernel to using standard system calls (`connect()`).

---

### Comparison Matrix

| Feature | Privileged Scan (`sudo` / `root`) | Unprivileged Scan (Standard User) |
| --- | --- | --- |
| **Packet Crafting Method** | **Raw Sockets:** Crafts custom IP/TCP/UDP headers bit-by-bit. | **OS Kernel API:** Uses standard `connect()` system calls. |
| **Default TCP Scan Type** | **TCP SYN Scan (`-sS`):** Half-Open scan. | **TCP Connect Scan (`-sT`):** Full 3-Way Handshake. |
| **TCP Handshake State** | **Incomplete:** Sends `RST` immediately after receiving `SYN/ACK`. | **Complete:** Completes `SYN` $\rightarrow$ `SYN/ACK` $\rightarrow$ `ACK` cycle. |
| **Stealth & Logging** | **High:** Rarely recorded in application-level logs (e.g., HTTP/FTP logs). | **Low:** Readily logged by target applications because a connection opens. |
| **Local Host Discovery** | Uses raw **ARP** (`-PR`) and raw **ICMP** requests. | Uses system TCP connection probes to ports `80` and `443`. |
| **Advanced Probe Support** | Fully supports ACK (`-sA`), FIN (`-sF`), NULL (`-sN`), Xmas (`-sX`). | **Unsupported:** Cannot send custom or broken TCP flags. |

---

### Key Operational Differences

#### 1. Connection Completion (Half-Open vs. Full Handshake)

* **Privileged (`-sS`):** Nmap sends a `SYN`. If the port replies with `SYN/ACK`, Nmap sends an `RST` (Reset) packet to tear down the connection before it establishes. Because the session never fully opens, target applications (like Apache, Nginx, or SSH) usually do **not** log an event.
* **Unprivileged (`-sT`):** The OS kernel manages the socket and completes the full TCP 3-way handshake (`SYN` $\rightarrow$ `SYN/ACK` $\rightarrow$ `ACK`). The target application establishes a real session, triggering an entry in the target's system and application logs.

#### 2. Local vs. Remote Host Discovery

* **Privileged:** On local Ethernet networks, Nmap bypasses standard IP routing and sends raw **ARP queries (`-PR`)** directly to target MAC addresses. On remote networks, it can craft custom ICMP or raw TCP SYN/ACK probes.
* **Unprivileged:** The user account cannot open raw sockets to send standalone ARP or ICMP packets. Nmap asks the OS kernel to establish standard TCP connections to ports `80` and `443` to deduce if a host is active.

#### 3. Custom Packet Control & Firewall Evasion

* **Privileged:** Pentesters can modify IP headers, adjust Time to Live (TTL) values (`--ttl`), fragment packets (`-f`), spoof source IPs (`-S`), or send raw TCP flags to probe firewall rules (e.g., TCP ACK scans).
* **Unprivileged:** Restricted strictly to standard network behaviors dictated by the OS networking stack. Custom flags, fragmentation, and header spoofing are impossible.
  
**run the command (`sudo` vs. no `sudo`), but primarily in how Nmap automatically changes its default behavior behind the scenes.**

---

### 1. The Command Prefix (`sudo`)

The most basic difference in the command line is whether you run Nmap with root/administrative privileges:

* **Privileged Command:**
```bash
sudo nmap 10.10.10.10

```


* **Unprivileged Command:**
```bash
nmap 10.10.10.10

```



---

### 2. Automatic Default Fallback

If you run a basic scan command without specifying a scan type flag (e.g., `nmap 10.10.10.10`), Nmap detects your user privileges and **automatically changes the underlying scan method**:

* **With `sudo`:** Nmap defaults to a **TCP SYN Scan (`-sS`)** using raw sockets.
* **Without `sudo`:** Nmap automatically falls back to a **TCP Connect Scan (`-sT`)** using standard OS kernel system calls.

---

### 3. Explicit Flag Restrictions & Errors

If an unprivileged user explicitly tries to request a scan flag that requires raw socket access, **Nmap will fail and throw a permission error**:

| Explicit Flag | Command | Run as `sudo` | Run as Standard User |
| --- | --- | --- | --- |
| **`-sS`** (SYN Scan) | `nmap -sS 10.10.10.10` | Runs Half-Open SYN Scan | **Error:** `Requested scan type requires root privileges` |
| **`-sT`** (Connect Scan) | `nmap -sT 10.10.10.10` | Runs Full Connect Scan | Runs Full Connect Scan (Allowed) |
| **`-sA`** (ACK Scan) | `nmap -sA 10.10.10.10` | Runs ACK Firewall Scan | **Error:** `Requested scan type requires root privileges` |
| **`-PR`** (ARP Discovery) | `nmap -PR -sn 10.10.10.0/24` | Sends Raw ARP Packets | **Error:** `ARP scan requires root privileges` |

---

### Summary Comparison of Commands

```bash
# 1. Privileged SYN Scan (Fast, Stealthy, Half-Open)
sudo nmap -sS 10.10.10.10

# 2. Unprivileged Connect Scan (Slower, Full 3-Way Handshake, Logged)
nmap -sT 10.10.10.10

```

**Key Insight:**  do not always need to type different flags—simply omitting `sudo` forces Nmap to automatically switch its execution engine from raw sockets (`-sS`) to kernel socket API calls (`-sT`).
---

**Key Insight:** Privileged scans leverage raw sockets to bypass the OS kernel, allowing testers to craft fast, stealthy, half-open probes, whereas unprivileged scans rely on standard kernel system calls, forcing full connection handshakes that leave clear footprints in application logs.

<img width="1143" height="459" alt="image" src="https://github.com/user-attachments/assets/5c226ad5-0869-4b0c-a0ff-2a8791751e15" />

<img width="1303" height="712" alt="image" src="https://github.com/user-attachments/assets/39cc6f8c-16c8-4a7a-af23-2b7e0f8edc8c" />
