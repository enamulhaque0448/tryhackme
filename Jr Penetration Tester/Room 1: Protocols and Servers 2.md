**Enamul Haque**
<img width="1855" height="146" alt="image" src="https://github.com/user-attachments/assets/d03784f4-82fc-4430-b15a-25cfe7eef8ab" />
Authentication Factors
## Authentication, or proving your identity, can be achieved through one of the following, or a combination of two or more:

* Something you know, such as a password or PIN code.
* Something you have, such as a phone, security key, or smart card.
* Something you are, such as a fingerprint or facial recognition.
## Hydra To do apssworad attack on a services like  FTP, POP3, IMAP, SMTP, SSH, and all methods related to HTTP.
```bash
hydra -l username -P wordlist.txt server service
```
## Examples are: 

```bash
# Attack FTP with username mark
hydra -l mark -P /usr/share/wordlists/rockyou.txt 10.49.188.203 ftp

# Alternative syntax (equivalent to above)
hydra -l mark -P /usr/share/wordlists/rockyou.txt ftp://10.49.188.203

# Attack SSH with username frank
hydra -l frank -P /usr/share/wordlists/rockyou.txt 10.49.188.203 ssh

# Attack IMAP with username lazie
hydra -l lazie -P /usr/share/wordlists/rockyou.txt 10.49.188.203 imap

# Attack with a list of usernames (credential stuffing style)
hydra -L users.txt -P passwords.txt 10.49.188.203 ssh
```
## Useful Hydra Options

| Option | Description |
| :--- | :--- |
| `-l username` | Single username to attack |
| `-L users.txt` | File containing a list of usernames |
| `-p password` | Single password to try |
| `-P wordlist.txt` | File containing a list of passwords |
| `-s PORT` | Specify a non-default port |
| `-V` or `-vV` | Verbose output showing attempts |
| `-t n` | Number of parallel connections (threads) |
| `-d` | Debug mode for troubleshooting |
| `-f` | Stop after the first valid password found |
| `-w n` | Wait time between connections |

> **Note:** Once the password is found, you can issue `CTRL-C` to end the process.

## Other Password Attack Tools

While Hydra is excellent for network service attacks, other tools serve different purposes:

*   **Medusa:** Similar to Hydra but with a modular design. Some find it more stable for certain protocols.
*   **Ncrack:** Developed by the Nmap project and designed for high-speed parallel authentication testing.
*   **CrackMapExec (CME) / NetExec:** Specialises in Windows/Active Directory environments and can spray passwords across SMB, WinRM, LDAP, and other protocols.
*   **Burp Suite Intruder:** Useful for attacking web-based login forms where Hydra's HTTP modules may not work correctly.
*   **Hashcat and John the Ripper:** Used for cracking password hashes offline rather than attacking live services. If you obtain password hashes (from a database breach, for example), these tools can recover the plaintext passwords much faster than attacking a live service.

## Mitigating Password Attacks

*   **Password Policies:** Prioritizes longer passwords and blocks leaked credentials over complex character rules.
*   **Account Lockout:** Locks accounts after repeated failed attempts to stop brute-forcing.
*   **Rate Limiting:** Implements intentional delays or exponential backoffs to slow down automated scripts.
*   **CAPTCHA:** Uses modern behavioral risk scoring to distinguish human users from automated machines.
*   **Multi-Factor Authentication (MFA):** Requires a secondary physical token or authenticator app code to block unauthorized access.
*   **Passwordless Authentication:** Removes passwords completely via cryptographic passkeys, magic links, or physical hardware tokens.
*   **Breached Password Detection:** Integrates external APIs to block users from using known, leaked credentials.
*   **Behavioral Analysis:** Flags impossible travel scenarios, unusual timing, or high-velocity login anomalies.
*   **IP Controls:** Restricts traffic using geographic boundaries or blocks requests originating from known malicious networks.
## Example Attack

<img width="1911" height="349" alt="image" src="https://github.com/user-attachments/assets/29c956d4-f3c9-4b01-9317-4748e0c41cc4" />

 ## Room Summary 

## 📌 Key Takeaways
*   **Cleartext Is Risky:** Unencrypted protocols are vulnerable to sniffing and MITM. 
*   **Secure Alternatives:** Switch to HTTPS, SSH, SFTP, and IMAPS/POP3S/SMTPS.
*   **Defense-in-Depth:** Combine encryption with strong passwords, rate limiting, and MFA.

---

## 🗺️ Protocol & Port Reference

| Protocol | TCP Port | Application | Security |
| :--- | :--- | :--- | :--- |
| **FTP** / **HTTP** / **Telnet** | 21 / 80 / 23 | Files / Web / Access | ❌ Cleartext |
| **IMAP** / **POP3** / **SMTP** | 143 / 110 / 25 | Email Services | ❌ Cleartext |
| **FTPS** / **HTTPS** | 990 / 443 | Files / Worldwide Web | 🔒 Encrypted (Implicit TLS) |
| **IMAPS** / **POP3S** / **SMTPS** | 993 / 995 / 465 | Email Services | 🔒 Encrypted (Implicit TLS) |
| **SSH** / **SFTP** | 22 | Remote Access / Files | 🔒 Encrypted (SSH) |
| **SMTP Submission** | 587 | Email Client Submission | 🔄 STARTTLS Upgrade |

---

## 🛠️ Hydra Quick Reference

*   `-l` / `-L`: Single username / Username list file.
*   `-p` / `-P`: Single password / Password list file.
*   `-s PORT`: Use if the target service runs on a non-default port.
*   `-t n` / `-w n`: Set parallel threads (`n`) / Set wait time seconds (`n`).
*   `-V` / `-f` / `-d`: Verbose output / Stop on first match / Debug mode.
*   `server service`: Target definition at the end (e.g., `10.10.10.10 ssh`).

---

## 🧰 Alternative Tools

*   **Traffic Analysis:** Wireshark, `tcpdump`, `mitmproxy`.
*   **MITM / Auditing:** Bettercap, `testssl.sh`, Nmap.
*   **Exploitation:** NetExec (CrackMapExec), Hashcat, John the Ripper, Burp Suite.

---

## 🛡️ Defensive Checklist

1. [ ] Force TLS 1.2/1.3 and fully disable legacy cleartext protocols.
2. [ ] Enforce SSH key-based authentication; disable password logins.
3. [ ] Implement account lockouts, rate limiting, and global MFA.
4. [ ] Enable HSTS headers on web servers to block SSL stripping.
5. [ ] Segment networks to prevent local packet sniffing and MITM sweeps.




