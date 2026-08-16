# 🚀 Web Security: Python HTTP Server Exposure & Data Leakage

Python includes a built-in web server module that allows anyone to spin up a fully functioning HTTP server with a single command line string. While designed for quick local testing, this convenience frequently leads to severe data exposures on penetration tests when left running indefinitely on public networks or cloud instances.

---

## ⚙️ 1. Core Syntax & Stack Behavior

The server is initiated via the command line and targets a specific listener port (commonly port `8000`):
```bash
# This command serves the current working directory over HTTP on port 8000
python3 -m http.server 8000
```

### The Fundamental Security Flaw
Unlike professional enterprise web servers (like Apache or Nginx) which feature robust configuration modules, access control rules, and disabled directory listings by default, Python's module has exactly **one operational mode: serve everything**. 

There are no access control lists, no authentication gates, and no configuration files (like `.htaccess`) to restrict data traffic. If a file exists anywhere inside the folder tree where the command was executed, it can be downloaded by anyone who can reach the port.

---

## 📁 2. Automated Directory Listings

When a user connects to the root directory (`http://target.local`), the server checks for the presence of a default index file (like `index.html`).

*   **If `index.html` exists**: The server renders that specific webpage normally.
*   **If `index.html` is missing**: The server automatically generates an HTML page displaying every file and subfolder in the directory line-by-line.

### Probing Directory Listings via Command Line
You can capture the raw HTML structure of this file index using `curl`:
```bash
curl -s http://10.48.130.146:8000/
```
The output will render an HTML directory index with the distinctive title: `<title>Directory listing for /</title>`. Finding this specific text string is a definitive indicator of an exposed Python server thread.

---

## 🎯 3. High-Value Targets for Exploitation

Because the server exposes files verbatim, attackers focus on capturing specific files containing sensitive hardcoded data.

### A. Exposing Hidden Dotfiles (`.env`)
In Linux operating systems, files starting with a dot (like `.env` or `.git`) are classified as hidden system assets. While hidden from normal terminal views, **Python's HTTP server ignores this convention entirely** and serves them to the public internet like any other text document.

Developers frequently use `.env` files to store operational configuration secrets, which can leak deep access into an organization:
```bash
curl -s http://10.48.130.146:8000/.env
```
#### Expected Leaked Configuration Output:
```text
SECRET_KEY=dev-secret-key-do-not-use
DATABASE_URL=postgresql://webapp:S3cur3DBPass!@localhost/production
DEBUG=True
```
*   **Analysis**: This output completely compromises the internal backend environment by leaking plaintext database passwords (`S3cur3DBPass!`) and administrative security keys.

### B. Extracting Legacy Backup Archives (`.zip`, `.tar.gz`)
Administrators often bundle application source folders or database states into local compression archives and forget to remove them. Finding an archive file in the directory listing allows you to extract full database schemas and application source logic locally.

#### Automated Analysis Workflow:
```bash
# 1. Download the remote archive file locally
curl -s http://10.48.130.146:8000/backup.zip -o backup.zip

# 2. Extract the contents into a dedicated folder
unzip backup.zip -d backup-contents/

# 3. Read the extracted database structure logs
cat backup-contents/db_dump.sql
```
#### Expected Leaked Database Output:
```sql
-- Database dump for staging environment
CREATE TABLE users (id INTEGER PRIMARY KEY, username VARCHAR(50));
INSERT INTO users VALUES (1, 'admin', 'admin@company.com');
```
*   **Analysis**: This discloses administrative user email schemas and table structures, allowing attackers to map out targeted brute-force operations against other corporate portals.

---

## 🧠 4. Why This Finding Matters

From a reporting perspective, an exposed Python HTTP server is a critical finding that **requires no technical exploit payloads to succeed**. The software is not broken; it is operating exactly as it was designed to function. 

The security vulnerability is entirely a **human misconfiguration flaw**—running an unauthenticated file-sharing process in an unmonitored zone where sensitive data assets are allowed to leak to unauthorized users.

---

## 🛡️ 5. Remediation and Countermeasures

To completely eliminate the risk of accidental file exposure through Python modules:

1.  **Terminate Temporary Processes**: Enforce strict administrative policies that forbid running `python3 -m http.server` on any production environment or public-facing server instance. Kill any orphan background processes immediately.
2.  **Enforce Strict Firewall Constraints**: Ensure that common testing ports (like `8000`, `8080`, and `3000`) are blocked by default at the network perimeter firewall layer (`iptables` or cloud Security Groups), allowing traffic only through verified standard web ports (`80`/`443`).
3.  **Migrate to Secure File-Transfer Solutions**: If files must be shared between server nodes, replace ad-hoc HTTP scripts with secure, authenticated transmission protocols such as **SCP**, **SFTP**, or enterprise-managed cloud storage layers that log traffic and require multi-factor authentication.

# 🚀 Web Security: Apache Server Misconfigurations & Gobuster Discovery
## Express.js Reconnaissance Workflow

1. **Fingerprint**: Run `curl -sI` to check for the `X-Powered-By: Express` response header.

2. **Read Version**: Query the root URL (`/`) to catch baseline JSON version data.

3. **Trigger Errors**: Request broken paths (like `/api/users`) to leak system file paths and raw queries via stack traces.

4. **Map Routes**: Visit `/api/routes` to dump the full list of application endpoints instantly.

5. **Dump Environment**: Request `/api/debug/env` to extract plaintext database passwords and backend keys.

6. **Scan Static Assets**: Pull front-end scripts (like `/static/config.js`) to find internal API URLs and hidden configuration constants.

**Apache HTTP Server** is one of the most widely deployed web servers in the world. Because of its massive installation footprint, it is a primary target during security assessments. While the core software is stable, its default Ubuntu deployment patterns frequently leave structural data exposed. These exposures give penetration testers a direct view into an organization's hidden internal assets.

---

## 🕵️ 1. Version Disclosure & OS Leaks

By default, an unhardened Apache installation on Ubuntu operates with the directive `ServerTokens OS`. This means that whenever a client interfaces with the platform, the server explicitly leaks both its exact release version and the underlying Linux operating system identity.

### Checking for Version Leaks via Command Line
```bash
# -S/I grabs headers only, grep isolates the server string case-insensitively
curl -SI http://target-app.local:80 | grep -i server
```
#### Expected Leaked Output:
```text
Server: Apache/2.4.58 (Ubuntu)
```
*   **Security Risk**: Disclosing the exact software version allows an attacker to quickly look up matching CVE vulnerability catalogs to map out targeted exploits against your environment.

---

## 📁 2. The `Options +Indexes` Directive (Directory Listing)

When an internet user navigates to a directory folder (e.g., `/files/`) that does not contain a default lander page like `index.html`, Apache checks its directory configuration rules.

If the **`Options +Indexes`** directive is left enabled, Apache automatically compiles a live HTML directory page displaying a list of all internal files stored inside that folder.

### The Attack Surface
The generated index lists exact filenames, file byte sizes, and precise last-modified dates. Attackers manually check every file listed in these open zones, as they regularly contain abandoned **CSV spreadsheets, database backups, developer notes, or staging media**.
<img width="1195" height="459" alt="image" src="https://github.com/user-attachments/assets/4b015d8f-e1f2-4eb8-8886-90e68ce7ee77" />

---

## 📊 3. The `mod_status` Leak (`/server-status`)

Apache ships with a built-in monitoring page powered by the **`mod_status`** module, normally mapped to the `/server-status` URI path.

### The Misconfiguration Flaw
By default, the framework restricts this page to local connections (`Require local`). However, if a developer mistakenly applies a **`Require all granted`** access policy *anywhere* else inside a Virtual Host configuration block, it silently overrides the security rule, exposing the diagnostic panel to the public internet.
<img width="1485" height="752" alt="image" src="https://github.com/user-attachments/assets/339163dd-f325-4145-8082-4e9dce1528a1" />

### What an Attacker Can See on `/server-status`:
*   **Real-time Traffic Tracking**: A live feed showing exactly what paths other internet visitors are currently browsing.
*   **Internal System Infrastructure**: Exposure of internal server uptime, total process worker counts, and server restart logs.
*   **Targeted Route Harvesting**: Revelation of hidden internal API paths or parameter query values passing through active browser connections.

---

## 🛠️ 4. Finding Unlinked Files with Gobuster Extension Sweeps

Many highly sensitive files sit inside a web server's root folder without being linked from any public webpage or directory index. Developers frequently create temporary copies or leave server configurations lying around. Attackers use directory brute-forcing to actively guess these names using targeted extension additions.

### The Advanced Gobuster Multi-Extension Scan
```bash
# -x appends extensions to every single word in the wordlist file
gobuster dir -u http://target-app.local:80 \
             -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
             -x bak,txt,html \
             -t 20
```
<img width="1666" height="665" alt="image" src="https://github.com/user-attachments/assets/20f5f4fa-c199-4efd-b997-b8a6baa6b14c" />

### 📋 High-Value Target Discoveries Explained

*   **`.bak` / Backup Files (e.g., `/backup.bak`)**: These are literal snapshots of source files or configuration states. Developers create them before changing code and forget to delete them.
    ```bash
    curl -s http://target-app.local
    # Output reveals hidden comments: "user: dbadmin pass: Backup2024!"
    ```
*   **`.htaccess` / `.htpasswd` (Status Code `403 Forbidden`)**: If Gobuster reports a `403 Forbidden` status code on paths like `/.htpasswd`, it positively confirms the directory exists but is guarded. The `.htpasswd` file stores username strings and cryptographically hashed passwords used for **HTTP Basic Authentication**. If an attacker manages to download an exposed `.htpasswd` file, they can attempt to crack the password hashes offline.

> 💡 **Performance Optimization Tip**: Appending multiple extensions via `-x bak,txt,html` multiplies the number of web requests significantly (e.g., a 4,000-word list becomes a 16,000-request scan). To maximize efficiency, **run a baseline scan without extension flags first** to uncover active folders, then execute a highly targeted sweep appending only `-x bak` to isolate backup artifacts.

---

## 🛡️ 5. Remediation and Server Hardening

To securely close down the default information exposure channels within an Apache deployment:

1.  **Disable Automated Directory Indexing**: Locate your configuration block files (`/etc/apache2/apache2.conf` or individual site files) and change the directory parameter to explicitly subtract the `Indexes` parameter:
    ```apache
    <Directory /var/www/>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    ```
2.  **Minimize Server Tokens**: Restrict Apache from communicating its system capabilities and OS tags to public internet clients:
    ```apache
    # Inside /etc/apache2/conf-enabled/security.conf
    ServerTokens Prod
    ServerSignature Off
    ```
3.  **Secure the Server Status Page**: Audit your virtual host blocks to guarantee that `/server-status` is locked behind local authorization mechanisms or strict IP access control lists (ACLs):
    ```apache
    <Location /server-status>
        SetHandler server-status
        Require local
        # Or require specific management IPs: Require ip 192.168.10.50
    </Location>
    ```
4.  **Audit the Document Root**: Implement automated cleanup routines within CI/CD pipelines to ensure backup file variations (`.bak`, `.old`, `.swp`) are entirely scrubbed from production deployment folders (`/var/www/html/`).


## Node.js (Express)
Node.js applications behave differently from Apache and Python HTTP servers. They are not serving static files from a configured document root. They are running application code, and that code decides what to return for every request. That flexibility is powerful, but it is also where many mistakes occur.

Attackers look at Node.js Express applications specifically because developers often leave development-mode features enabled when they push to production. Debug endpoints, verbose error responses, and exposed environment variables all stem from the same habit: code that worked in development went live as-is. The result is an application that tells you exactly how it is built and, sometimes, what credentials it uses.

### Framework Fingerprinting
The headers on port 3000 confirm what you are dealing with:

```text
Terminal
root@ip-10-81-64-63:~# curl -sI http://10.49.130.22:3000
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 56
ETag: W/"38-K8iCfm09rMr0MV0NsgqdAb94DAk"
Date: Sat, 11 Apr 2026 07:27:28 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

You will see `X-Powered-By: Express` in the response. Express sets this header automatically unless the developer explicitly disables it. This confirms you are dealing with an Express application, which tells you what to look for next.

### Reading the Application Version
The root path of many Express applications returns a JSON status response:

```text
Terminal
root@ip-10-81-64-63:~# curl -s http://10.49.130.22:3000
{"status":"ok","app":"company-portal","version":"1.2.0"}
```

If the response includes a version field, note it. Matching the application version against known vulnerabilities or changelogs can surface useful information.

### Triggering Verbose Errors
Express's built-in error handler suppresses stack traces when `NODE_ENV` is set to production. In development mode, it passes them through. But developers often write their own error handlers, and those custom handlers expose stack traces regardless of `NODE_ENV`. In the attached VM, the web app uses a custom error handler. The detailed JSON response you see is not part of Express behaviour; it is the result of application code written for debugging and never hardened. Both patterns appear on real engagements: the `NODE_ENV=production` check is only meaningful if no custom error handler overrides it. Consider a scenario where there is an api call being made to the Express server, as shown below:

```text
Terminal
root@ip-10-81-64-63:~# curl -s http://10.49.130.22:3000/api/users | python3 -m json.tool
{
    "error": "connect ECONNREFUSED 127.0.0.1:5432",
    "stack": "Error: connect ECONNREFUSED 127.0.0.1:5432\n    at /opt/nodeapp/app.js:16:15\n    at Layer.handle [as handle_request] (/opt/nodeapp/node_modules/express/lib/router/layer.js:95:5)\n    at next (/opt/nodeapp/node_modules/express/lib/router/route.js:149:13)\n    at Route.dispatch (/opt/nodeapp/node_modules/express/lib/router/route.js:119:3)\n    at Layer.handle [as handle_request] (/opt/nodeapp/node_modules/express/lib/router/layer.js:95:5)\n    at /opt/nodeapp/node_modules/express/lib/router/index.js:284:15\n    at Function.process_params (/opt/nodeapp/node_modules/express/lib/router/index.js:346:12)\n    at next (/opt/nodeapp/node_modules/express/lib/router/index.js:280:10)\n    at expressInit (/opt/nodeapp/node_modules/express/lib/middleware/init.js:40:5)\n    at Layer.handle [as handle_request] (/opt/nodeapp/node_modules/express/lib/router/layer.js:95:5)",
    "query": "SELECT * FROM users"
}
```
<img width="1906" height="448" alt="image" src="https://github.com/user-attachments/assets/8bfcb6b1-bcfd-47c7-957d-b0d3cdf5e8e1" />

A 500 Internal Server Error from an API endpoint is worth investigating further. In development mode, the response body often contains the error message, stack trace, and context about what the application was trying to do when it failed. The stack trace is the key finding here, as it reveals internal file paths, module names, and, sometimes, the database query that triggered the failure. Each of those details tells you something about the application's internals that a simple status code does not.

### Enumerating Routes via Debug Endpoints
One of the most useful things a misconfigured Express application can do is tell you all of its own routes. Developers sometimes build a debug endpoint that lists every registered route for convenience during development, then forget to remove it before going live.

```text
Terminal
root@ip-10-81-64-63:~# curl -s http://10.49.130.22:3000/api/routes
[{"method":"GET","path":"/"},{"method":"GET","path":"/api/users"},{"method":"GET","path":"/api/routes"},{"method":"GET","path":"/api/debug/env"}]
```

If this endpoint exists, the response is a list of every path the application handles. This saves enumeration time and shows you which endpoints exist before you spend time guessing them with Gobuster.

> 📝 **Note:** The `/api/routes` endpoint works by reading Express's internal `app._router.stack` property to enumerate registered routes. This is a recognised Express misconfiguration pattern, but `app._router.stack` is an internal property whose structure can vary between Express versions, and Express 5 changed the router internals enough to break implementations that relied on the Express 4 structure. If you encounter a route endpoint that returns an unexpected format or an error, the application may be running a different Express version than the one this lab uses.

### Exposed Environment Variables
Environment variables in Node.js applications often contain database credentials, API keys, and configuration flags. A debug endpoint that returns `process.env` is a significant finding:

```text
Terminal
root@ip-10-81-64-63:~# curl -s http://10.49.130.22:3000/api/debug/env
{"NODE_ENV":"development","DB_PASSWORD":"NodeDBPass2024!","PORT":"3000","DB_HOST":"localhost:5432","APP_NAME":"company-portal"}
```
<img width="1749" height="152" alt="image" src="https://github.com/user-attachments/assets/7263b1f9-3dff-4ba9-ac6a-3f707d65ecbc" />

If the response includes `DB_PASSWORD`, `SECRET_KEY`, or similar fields, document them. The `NODE_ENV` value is also telling. `NODE_ENV=development` on a production server is a signal that the application was deployed without proper hardening.

### Static File Serving
Express applications commonly use the **`express.static()` middleware to serve front-end assets like JavaScript files, stylesheets, and configuration.** The static files route, if it exists, serves everything in a directory. Client-side JavaScript files sometimes contain API endpoint URLs, internal hostnames, or debug flags embedded as constants.

Once you know the routes from the `/api/routes` endpoint, check what is being served statically:

```text
Terminal
root@ip-10-81-64-63:~# curl -s http://10.49.130.22:3000/static/config.js
// Client-side configuration
const API_BASE = 'http://internal-api.company.local:8080';
const DEBUG = true;
const VERSION = '1.2.0';
```
<img width="1091" height="200" alt="image" src="https://github.com/user-attachments/assets/29fd4b4e-380e-4128-8e94-af467d6c9a41" />

Configuration files served as static assets are easy to overlook because they are technically meant to be public (the browser needs them). But "meant to be public" is different from "contains only public information."

> 📝 **Note:** `express.static()` silently returns a 404 for dotfiles (files starting with `.`) by default. This is the opposite of Python's HTTP server, which serves dotfiles like any other file.
>  If you are looking for a `.env` or similar file on an Express static route and get a 404, that does not mean the file is absent; it means the middleware is blocking it. You would need shell access or a different route to reach it.

### Putting it All Together
Take a step back and look at what we just did. We started with header inspection to confirm the framework,
then triggered an error to see what the application leaks in failure mode, then used a debug endpoint to enumerate routes,
then checked for credential exposure in environment variables, and finally followed the route list to the static files where the flag lives.
This pattern works because each step narrows the scope for the next one. Headers tell you the framework. Errors tell you the internals.
Debug endpoints tell you the routes. Static files tell you what the developers assumed was safe to expose.

# 🚀 Advanced Nginx Security: Version Disclosure, Autoindex, Status Leaks & Configuration Analysis

Nginx follows the same investigative methodology as Apache but uses different configuration directives and file locations. The core reconnaissance workflow remains consistent: fingerprint the server, inspect exposed directories, identify monitoring endpoints, and understand how configuration decisions create information disclosure risks.

## ⚡ Nginx Reconnaissance Workflow

1. **Fingerprint**: Run `curl -sI` to identify the `Server` response header and determine the Nginx version.
2. **Verify Error Pages**: Request a non-existent path to check whether `server_tokens` leaks version information.
3. **Browse Autoindex Directories**: Visit exposed folders (such as `/files/`) to enumerate publicly accessible files.
4. **Inspect Every Listed File**: Download operational documents, backups, and configuration files exposed through directory listings.
5. **Check `nginx_status`**: Query the monitoring endpoint to view live connection statistics and server activity.
6. **Review Configuration Paths**: If shell access exists, inspect `/etc/nginx/sites-available/` to identify exposed locations and enabled modules.

---

## 🌐 Understanding Nginx's Role

After Apache and Node.js, the **Nginx** investigation follows the same template but with its own configuration vocabulary. The categories of misconfiguration—version disclosure, directory listing, and exposed status pages—appear again, but the specific directives and file paths differ.

Nginx occupies a different role from Apache and Node.js. It commonly functions as a **reverse proxy**, **load balancer**, or **high-performance static file server**. In production environments, it often sits in front of an application server and handles all public-facing traffic. Because of that position, configuration mistakes can expose internal structure, operational metrics, or sensitive files.

> **Info:** In the attached VM, Nginx runs on **port 8080** instead of the standard **80** because Apache already occupies port 80. In real deployments, Nginx typically listens on **80** or **443**.

---

# 🔍 Version Disclosure

Like Apache, begin by examining the `Server` response header.

### Command

```bash
curl -sI http://10.49.130.22:8080 | grep -i server
```

### Output

```text
Server: nginx/1.24.0 (Ubuntu)
```

By default, Nginx exposes its version through the `Server` header. Unlike Apache's `ServerTokens`, Nginx controls this behavior using the **`server_tokens`** directive.

### Why it matters

- Identifies the exact installed version.
- Helps map known security fixes and changelog entries.
- Provides useful information for penetration testing reports.

## `server_tokens` Behavior

The `server_tokens` directive controls both:

- The `Server` response header.
- The version banner displayed on default error pages.

Setting:

```nginx
server_tokens off;
```

removes the version from both locations simultaneously.

### Verify Using a 404 Page

Request a path that does not exist.

```bash
curl -s http://10.49.130.22:8080/nonexistent-path
```

### Example Output

```html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
```

If the version still appears here, `server_tokens` is either enabled or only partially configured.

---

# 📁 Directory Listing with Autoindex

Unlike Apache, Nginx **does not enable directory listing by default**.

Developers explicitly enable it using the `autoindex` directive.

### Example Configuration

```nginx
location /files/ {
    autoindex on;
    root /var/www/nginx/;
}
```

This configuration is legitimate for public file-sharing directories, but it becomes a security issue when sensitive operational files are stored there.

## Enumerating an Autoindex Directory

Browse the exposed directory.

```bash
curl -s http://10.49.130.22:8080/files/
```

### Example Output

```html
<html>
<head><title>Index of /files/</title></head>
<body>
<h1>Index of /files/</h1><hr><pre><a href="../">../</a>
<a href="deploy-notes.txt">deploy-notes.txt</a>                                   03-Apr-2026 18:23                 148
<a href="old-backup.tar.gz">old-backup.tar.gz</a>                                  03-Apr-2026 18:23                 236
<a href="server-config.txt">server-config.txt</a>                                  03-Apr-2026 18:23                 135
</pre><hr></body>
</html>
```

## What Nginx Reveals

The generated directory listing includes:

- File names
- Last-modified timestamps
- File sizes

These details can reveal deployment history, backup archives, configuration notes, or other operational artifacts.

> **Tip:** Read every exposed file. Developers frequently place migration notes, backups, and server documentation inside directories originally intended for internal use.

---

# 📊 The `nginx_status` Endpoint

Nginx includes a lightweight monitoring module called **`stub_status`**.

A secure configuration limits access to localhost.

### Secure Example

```nginx
location /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

### Misconfigured Example

```nginx
location /nginx_status {
    stub_status;
    allow all;
}
```

This exposes live operational metrics to anyone.

## Query the Endpoint

```bash
curl -s http://10.49.130.22:8080/nginx_status
```

### Example Output

```text
Active connections: 1
server accepts handled requests
 15 15 15
Reading: 0 Writing: 1 Waiting: 0
```

## Understanding the Numbers

The second line contains three cumulative counters:

| Counter | Meaning |
|---------|---------|
| First | Total accepted connections |
| Second | Total handled connections |
| Third | Total requests processed |

The final line shows current worker states.

| State | Meaning |
|-------|---------|
| Reading | Processing incoming requests |
| Writing | Sending responses |
| Waiting | Idle keep-alive connections |

Although this endpoint does not directly expose credentials, it leaks valuable operational intelligence, including server load and monitoring configuration.

---

# 📂 Configuration File Locations

If shell access is available, inspect Nginx's configuration directory.

### Primary Configuration

```text
/etc/nginx/
```

### Site Configurations

```text
/etc/nginx/sites-available/
```

Reading these files reveals:

- Exposed directories
- Enabled modules
- Reverse proxy routes
- Autoindex locations
- Status endpoint configuration

---

# 🛡️ Hardening Nginx

## Step 1: Hide Version Information

Edit the main configuration.

```nginx
server_tokens off;
```

This removes version disclosure from both headers and default error pages.

## Step 2: Disable Autoindex

Replace:

```nginx
autoindex on;
```

with:

```nginx
autoindex off;
```

Only enable directory listings when absolutely required.

## Step 3: Restrict `nginx_status`

Use localhost-only access.

```nginx
location /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

If remote monitoring is necessary, whitelist only trusted management IPs.

## Step 4: Remove Sensitive Files

Ensure deployment pipelines automatically remove:

- `*.bak`
- `*.old`
- backup archives
- development notes
- temporary configuration files

before publishing content.

---

# 🎯 Putting It All Together

The Nginx investigation mirrors the Apache workflow:

1. Check the `Server` header.
2. Verify error pages for version leakage.
3. Browse directories with `autoindex`.
4. Download every exposed file.
5. Check the `nginx_status` endpoint.
6. Review `/etc/nginx/sites-available/` when shell access is available.

Although the directives differ (`server_tokens`, `autoindex`, and `stub_status`), the investigative methodology remains the same: identify unnecessary information disclosure, enumerate exposed resources, and document configuration weaknesses for remediation.
