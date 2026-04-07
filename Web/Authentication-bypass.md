# Authentication Bypass — Attack Model

**Platform:** TryHackMe
**Topic:** Authentication Bypass
**Focus:** Trust boundary, input → logic → exploit

---

## ⚡ TL;DR

Authentication Bypass occurs when the server **trusts attacker-controlled input**.

Attack pattern:

```text
CONTROL → CHECK → BYPASS → GAIN
```

* CONTROL → what input attacker can manipulate
* CHECK → what validation the server performs
* BYPASS → flaw in logic or trust
* GAIN → access (account / page / session)

---

## 🧠 Core Principle

```text
Anything from the client = attacker-controlled
(GET, POST, COOKIE)
```

```text
If attacker controls it AND server trusts it → exploit
```

---

## 1️⃣ Username Enumeration

### Concept

* Application reveals whether a username exists
* Example:

  * “Username already taken” → valid username
  * “Username available” → not used

### Why it matters

```text
Reduces attack from:
(username + password)
→
(password only)
```

### Tool

```bash
ffuf -u http://target -w wordlist -X POST -d "username=FUZZ"
```

### Key Idea

```text
Look for DIFFERENT response
(text / status / size)
```

---

## 2️⃣ Brute Force

### Concept

* Try multiple username/password combinations
* Detect success using response differences

### Limitations

* Large password space
* Rate limiting
* Account lockout
* Weak or noisy signals

---

## 3️⃣ ffuf — Signal-Based Detection

### Key Rule

```text
Do NOT memorize blindly
→ Observe signal first
```

---

### Common Options

```text
-u  → target URL
-d  → POST data
-H  → headers
-mr → match response text
-fc → filter status code
-fs → filter response size
```

---

### Common Signals

```text
200 → normal response
302 → redirect (often success)
403 → forbidden
```

---

### 🔥 Key Concept

```text
Signal = observable difference between success and failure
(status, size, response text, cookies)
```

---

### Takeaway

```text
Choose detection method based on signal, not habit
```

---

## 4️⃣ Logic Flaw (GET / POST / REQUEST)

---

### Input Sources

```text
$_GET      → URL (query string)
$_POST     → request body (form input)
$_REQUEST  → combined input (GET + POST + COOKIE depending on config)
```

```text
POST often overrides GET when keys collide
```

---

### Key vs Value

```text
username=robert
key   = username
value = robert
```

```text
$_POST['username'] ONLY reads "username="
```

---

### Critical Rule

```text
You can SEND anything
But server only USES what it reads
```

---

### Vulnerability Pattern

```text
WHO   = source A
WHERE = source B
→ mismatch → exploit
```

---

### Example

```php
$user = find_user($_GET['email']);     // WHO
send_email($_REQUEST['email']);        // WHERE
```

```text
GET  email = victim
POST email = attacker
```

```text
WHO = victim
WHERE = attacker
→ exploit
```

---

### Fix (Secure Design)

```text
Use SAME trusted source for identity + action
```

---

## 5️⃣ Cookies (State & Trust)

---

### Why Cookies Exist

```text
HTTP is stateless → server forgets each request
Cookie = client-side state storage sent back to server
```

---

### How Cookies Work

```text
Server → Set-Cookie
Browser → stores
Browser → sends back each request
```

---

### Cookie = INPUT

```text
Cookies are attacker-controlled
```

---

### Vulnerable Example

```text
Cookie: admin=true
Server: "ok admin"
→ attacker modifies → admin access
```

---

### Safe vs Dangerous

#### ✅ Safe (non-security)

```text
theme=dark
language=en
```

#### ❌ Dangerous (security)

```text
admin=true
role=admin
user_id=1
```

---

### Correct Design (Session-Based)

```text
Cookie: session=abc123
```

Server:

```text
abc123 → database → user_id, role
```

```text
Cookie = reference
Server = source of truth
```

---

### Encoding vs Hashing

#### ❌ Encoding (not security)

```text
base64 → reversible
attacker: decode → modify → encode
```

#### Hashing

```text
One-way, but weak inputs can still be guessed
```

---

## 6️⃣ JWT (Modern Session)

---

### Structure

```text
HEADER.PAYLOAD.SIGNATURE
```

---

### Example Payload

```json
{"id":1,"role":"user"}
```

---

### How It Works

```text
Server signs token with secret
Server verifies signature on request
```

---

### Key Property

```text
Attacker CAN read/modify payload
BUT cannot forge valid signature
```

---

### ⚠️ Critical Rule

```text
Server MUST verify signature before trusting payload
```

---

### Vulnerabilities (if misused)

```text
No signature verification ❌
Weak secret ❌
```

---

## 🔥 Final Mental Model

```text
INPUT SOURCES:
- GET
- POST
- COOKIE

All = attacker-controlled
```

```text
Server must:
- validate
OR
- ignore for security decisions
```

---

## 🎯 Ultimate Rule

```text
Cookies are INPUT, not TRUTH
```

```text
Session = "ask server who I am"
JWT     = "prove who I am with signature"
```

---

## ⚔️ Attacker Checklist

```text
1. What input can I control?
2. Where is it used?
3. Is it trusted?
4. Can I desync WHO vs WHERE?
5. Can I modify cookies?
6. Is it protected (session / signature)?
```

---

## 🚀 Final Takeaway

```text
This topic is NOT about tools.
It is about TRUST BOUNDARY.
```

```text
If server trusts client → you win
```
