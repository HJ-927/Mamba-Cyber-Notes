# HTB Usage — Learning Journey Writeup

## Box Information

| Item           | Value       |
| -------------- | ----------- |
| Platform       | HTB         |
| Box            | Usage       |
| Difficulty     | Easy        |
| Date Completed | 29 May 2026 |
| Time Taken     | 15 hrs      |

---

# Stage 1 — Reconnaissance

## Signals Observed

### Open Ports

* 22/tcp
* 80/tcp

### Services

* SSH
* HTTP

## My Initial Hypothesis

I chose to connect remotely to the HTTP service first.

### Why I Chose This Direction

Because HTTP services usually expose more unauthenticated attacker-controlled input surfaces.

These input surfaces may interact with backend components such as:

* Databases
* Application logic
* File handling
* Command execution

SSH is usually a later target because:

* SSH normally requires valid credentials before gaining access.
* After finding credentials from another attack path, SSH can become useful for stable access.

## Key Lesson

An open service does **not** automatically mean a vulnerability exists.

Nmap only tells me:

```text
what is reachable
```

Not:

```text
what is exploitable
```

The thought process should be:

```text
service
↓
input surface
↓
trust boundary
↓
validation
↓
exploitability
```

---

# Stage 2 — Hostname & Virtual Host Discovery

## Signals Observed

When browsing:

```text
http://<target-ip>
```

I observed:

```text
Location: http://usage.htb
```

## What Confused Me

I previously mixed up:

* Hostname
* Domain
* Subdomain
* DNS resolution
* IP address
* Path

## Mechanism

My understanding:

1. User enters a hostname in the browser.

Example:

```text
http://usage.htb
```

2. Browser asks the OS resolver:

```text
"What IP address belongs to this hostname?"
```

3. OS checks:

```text
/etc/hosts
```

first.

Example:

```text
10.x.x.x usage.htb
```

4. If not found locally, the OS queries DNS.

5. Once the hostname resolves into an IP address:

* Browser establishes a TCP connection.
* Browser sends an HTTP request containing the hostname.
* Server uses the hostname to determine which website should handle the request.

## Mistakes Made

Previously I confused:

* IP address
* Hostname
* Domain
* Subdomain
* Path

## Corrected Mental Model

### IP Address

```text
Where the server is located
```

### Hostname

```text
Which website I want from that server
```

### Path

```text
Which resource inside that website
```

## Analogy

```text
IP Address
=
Building Address

Domain
=
Queensbay Mall

Subdomain
=
Specific Shop

Path
=
Specific Department Inside The Shop
```

One server/IP can host multiple websites.

Hostname tells the server which website is being requested.

---

# Stage 3 — SQL Injection Discovery

## Signals Observed

Input surfaces discovered:

* Login page
* Registration page
* Reset password page
* Admin login page

Observed:

* Status code differences
* Response length differences
* Response body differences
* TRUE/FALSE behavior differences

## Initial Hypothesis

My payload might be reaching the SQL query.

The backend may be treating attacker-controlled input as SQL logic rather than ordinary string data.

## Validation Process

Rather than randomly dumping payloads:

Observe:

* Response status
* Response length
* Response body
* Application behavior

### Payload: `'`

Purpose:

Test whether special SQL syntax causes backend processing differences.

Important:

A 500 error **does not prove SQLi**.

It only proves:

```text
input
↓
unexpected backend behavior
```

Further validation is required.

### Payloads: `AND 1=1` / `AND 1=2`

Purpose:

Boolean-based SQL injection testing.

Logic:

```text
TRUE condition
↓
Response A

FALSE condition
↓
Response B
```

Consistent behavioral differences suggest attacker influence over SQL logic.

### Payload: `OR 1=1`

Purpose:

Authentication bypass testing.

Logic:

```text
Force SQL condition
↓
TRUE
```

## SQL Injection Mechanism

```text
Attacker Input
↓
Trusted By Application
↓
Sent To SQL Query
↓
Database Interprets As SQL Logic
↓
Query Behavior Changes
```

---

## Laravel Request Flow Lesson

I was initially confused because:

Browser displayed:

```text
Password reset link sent
```

Burp Repeater showed:

```text
302 Redirect
```

### Explanation

Different layers.

```text
POST Request
↓
Laravel Processes Request
↓
Stores Flash Message
↓
Returns 302 Redirect
↓
Browser Automatically Follows Redirect
↓
Page Is Rendered
↓
Message Appears
```

Repeater only shows the first response.

Browser follows the complete workflow.

---

## CSRF Lesson

### `laravel_session`

Purpose:

```text
Which session/user is making the request
```

Maps:

```text
Session ID
↓
Server-Side Session
```

### `_token`

Hidden form CSRF token.

Purpose:

```text
Did this request follow the expected workflow?
```

### `XSRF-TOKEN`

Cookie copy of the CSRF token.

Purpose:

Allows frontend JavaScript frameworks to access the CSRF token.

---

## Mistakes Made

* Thinking different responses automatically confirmed SQLi.
* Dumping payloads without understanding their purpose.
* Mixing:

  * Database response
  * Application response
  * HTTP response
  * Browser display

Correct approach:

```text
Signal
↓
Hypothesis
↓
Validation
↓
Confidence
```

---

# Stage 4 — Blind SQLi Enumeration

## Goal

Extract:

* Database name
* Table names
* Column names
* Usernames
* Password hashes

## Enumeration Process

```text
Database
↓
Tables
↓
Columns
↓
Data
```

## Concepts Learned

### `database()`

Returns the current database name.

### `information_schema`

Metadata database.

Useful for discovering:

* Tables
* Columns
* Database structure

### `COUNT(*)`

Counts matching rows.

Examples:

* How many tables exist?
* How many columns exist?

### `EXISTS()`

Returns TRUE/FALSE depending on whether something exists.

Useful for Boolean SQLi.

### `LIMIT`

Controls how many rows are returned.

### `OFFSET`

Chooses which position to begin returning results from.

Useful for extracting results one at a time.

### `ASCII()` + `SUBSTRING()`

Used to extract characters individually during blind SQLi.

## Biggest Lesson

Instead of asking:

```text
"What payload should I use?"
```

Ask:

```text
"What question am I asking the database?"
```

## Mistakes Made

* Manual extraction was too slow.
* Repetitive Boolean testing should be automated.
* SQL syntax precision matters:

  * Quotes
  * Brackets
  * Query structure

---

# Stage 5 — Laravel Version Disclosure & RCE

## Signals Observed

After gaining admin access:

Found:

* Laravel version
* PHP version
* Server information

## CVE Research

Discovered:

```text
CVE-2023-24249
```

through version research.

## Exploitation Path

Admin user could upload a profile image.

Uploaded a PHP payload disguised as an image.

Modified filename:

```text
file.php.jpg.php
```

Server accepted the upload.

PHP payload became reachable.

## Important Distinctions

```text
Upload Success
≠
PHP Execution
```

```text
PHP Execution
≠
OS Command Execution
```

```text
RCE
≠
Reverse Shell
```

Each layer requires validation.

## Validation Thinking

### Can I Upload?

↓

### Can PHP Execute?

Example:

```php
echo "test";
```

↓

### Can PHP Execute OS Commands?

Example:

```php
system();
```

## Lessons Learned

* Version disclosure helps CVE research.
* Always verify CVE applicability.
* No output does not necessarily mean failure.
* Responses may be filtered or hidden.

---

# Stage 6 — Reverse Shell & User Pivot

## RCE vs Reverse Shell

### RCE

Remote Code Execution means:

```text
Attacker
↓
Application
↓
OS Command Execution
↓
Result
```

But RCE alone does not provide a stable shell.

### Reverse Shell

```text
Attacker Listener
↓
Target Connects Back
↓
Interactive Shell
```

Provides continuous interaction.

---

## Shell Stabilization

### PTY

Improves:

* Interactive behavior
* `su`
* Terminal usability

### `TERM=xterm`

Improves:

* Screen handling
* Interactive programs
* Terminal rendering

---

## Enumeration

Questions to answer:

```text
Who am I?
Where am I?
What can I access?
What interesting files exist?
```

Actions:

* Check current user
* Enumerate application files
* Search for credentials
* Review configuration files

---

## Pivot

Found:

```text
xander credentials
```

Movement:

```text
www-data
↓
xander
```

Important:

```text
Pivot
≠
Root
```

Pivot simply means moving into another security context.

---

## Lessons Learned

```text
RCE
≠
Shell

Shell
≠
Pivot

Pivot
≠
Privilege Escalation

Root
≠
Understanding The Mechanism
```

---

# Stage 7 — Privilege Escalation

## Initial Discovery

```bash
sudo -l
```

Output:

```text
(root) NOPASSWD: /usr/bin/usage_management
```

Meaning:

`xander` can execute this binary as root without a password.

---

## CONTROL / EXECUTION / POWER Analysis

### CONTROL

Unknown.

Requires investigation.

### EXECUTION

Known.

```text
/usr/bin/usage_management
```

runs as root.

### POWER

Known.

```text
root
```

## Key Lesson

Wrong:

```text
sudo binary found
↓
must be exploitable
```

Correct:

```text
sudo binary found
↓
POWER found
↓
search for CONTROL
```

---

## Binary Investigation

### strings

```bash
strings /usr/bin/usage_management
```

Interesting findings:

```text
/usr/bin/7za
/var/backups/project.zip
/var/www/html
```

### Why Interesting?

#### `/usr/bin/7za`

A privileged program is launching another program.

Questions:

* How is it called?
* What inputs reach it?

#### `/var/backups/project.zip`

Potential archive destination.

#### `/var/www/html`

Potential attacker-controlled location.

## Important Lesson

`strings` output is:

```text
clue
```

Not:

```text
proof
```

Presence of a string does not prove:

* Execution
* Reachability
* Exploitability

---

## 7za Command Discovery

Discovered:

```bash
/usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *
```

### Initial Hypothesis

The wildcard made me consider:

```text
Wildcard Option Injection
```

### Why It Failed

Because:

```text
--
```

means:

```text
Stop Option Parsing
```

Anything after becomes normal arguments.

Example:

```text
--checkpoint-action=exec
```

would be treated as a filename.

---

## Parser Lesson

Syntax is not universal.

Examples:

```text
*
@
--
```

must always be interpreted in the context of the program parsing them.

---

# Stage 8 — 7za Listfile Abuse

## Discovery of `@`

In 7za:

```text
@file.txt
```

means:

```text
Open file.txt
↓
Read contents
↓
Treat contents as file list
```

### Important Correction

The listfile is **not**:

```text
@file.txt
```

The listfile is:

```text
file.txt
```

7za simply uses `@` as a signal.

---

## Example

If:

```text
file.txt
```

contains:

```text
/root/.ssh/id_rsa
```

Then:

```text
7za
↓
Reads file.txt
↓
Reads /root/.ssh/id_rsa
↓
Archives It
```

---

## Why It Worked

### CONTROL

Attacker controls listfile contents.

### EXECUTION

Root executes:

```text
usage_management
```

which launches:

```text
7za
```

### POWER

Root.

Full chain:

```text
Attacker-Controlled Data
↓
Trusted Parser
↓
Parser Runs As Root
↓
Root Performs Action
```

---

## Feature ≠ Vulnerability

`@` itself is a legitimate feature.

The vulnerability appears when:

```text
Useful Feature
+
Attacker Control
+
Privileged Execution
```

combine together.

---

# Root Access

## Final Chain

```text
nmap
↓
HTTP Discovery
↓
Hostname Resolution
↓
SQL Injection
↓
Admin Credentials
↓
Admin Access
↓
Vulnerable Upload
↓
RCE
↓
Reverse Shell
↓
Enumeration
↓
Xander Credentials
↓
Pivot
↓
sudo -l
↓
usage_management Analysis
↓
7za Listfile Abuse
↓
Root SSH Key
↓
Root Access
```

---

# Final Reflection

## What I Understand Better Now

* Hostname / Domain / Subdomain / Path
* DNS Resolution
* Virtual Host Behavior
* Laravel Request Flow
* Browser vs Burp Differences
* Flash Messages
* CSRF Tokens
* Session Cookies
* Boolean SQLi Methodology
* Blind SQL Extraction
* Python Automation Purpose
* CVE Validation
* Upload Validation Layers
* RCE vs Reverse Shell
* Privilege Escalation Reasoning
* Strings vs Runtime Behavior
* Parser Semantics
  
---

# Current Weaknesses

Need to improve:

* Payload syntax precision
* SQL query writing speed
* Python automation ability
* Filtering noisy enumeration output
* Binary analysis speed
* Recognizing privilege escalation patterns faster
