# Command Injection — Attack Model & Notes

Command Injection occurs when **attacker-controlled input is interpreted as part of an OS command and executed by the application**.

---

## 🔥 Core Attack Model

```text
CONTROL → FLOW → EXECUTION = COMMAND INJECTION
```

* **CONTROL** → where attacker input enters
* **FLOW** → how input is inserted into a command
* **EXECUTION** → where the system runs the command

---

## ⚔️ Code Reading Mindset

Do NOT read like a developer. Read like an attacker.

### Look for:

* **CONTROL**

  * `$_GET`
  * `$_POST`
  * form inputs
  * URL parameters

* **FLOW**

  * string concatenation
  * variables inserted into commands

* **EXECUTION**

  * `system()`
  * `exec()`
  * `passthru()`
  * `subprocess.Popen(..., shell=True)`

---

## ⚔️ Input Channels

### 🔹 GET

```text
Data sent via URL
```

Example:

```text
http://target.com/ping?ip=127.0.0.1
```

---

### 🔹 POST

```text
Data sent in request body (not visible in URL)
```

Example:

```text
ip=127.0.0.1
```

---

## ⚔️ Output Behavior

### 🔹 Verbose Injection

* Command output is shown directly
* Easy to confirm execution

---

### 🔹 Blind Injection

* No visible output
* Must confirm via side effects

Examples:

```bash
sleep 5
ping -c 5 127.0.0.1
```

---

## ⚔️ Useful Payloads

### 🔍 Basic Enumeration

```bash
whoami
id
ls
```

---

### ⏱ Blind Testing

```bash
sleep 5
ping -c 5 127.0.0.1
```

---

### 💥 Foothold / Shell

```bash
nc
```

(Used after confirming command execution)

---

## ⚔️ Shell Operators

| Operator | Behavior                                |   |                                      |
| -------- | --------------------------------------- | - | ------------------------------------ |
| `;`      | Always executes next command            |   |                                      |
| `&&`     | Executes only if first command succeeds |   |                                      |
| `        |                                         | ` | Executes only if first command fails |

### 🎯 Key Insight

```text
Use `;` for reliable testing
```

---

## ⚔️ Filtering & Validation

### 🔴 Client-Side

* HTML `pattern=`
* JavaScript validation

```text
Weak — can be bypassed easily
```

---

### 🟡 Server-Side

* Input validation on backend

```text
Stronger, but NOT guaranteed safe
```

---

### ⚠️ Attacker Mindset

```text
Never trust filters — always test bypass
```

---

## ⚔️ Privilege Context

* Commands run as the **web server user** (e.g., `www-data`)
* Even low privilege is useful:

  * file reading
  * credential discovery
  * system enumeration
  * pivot to privilege escalation

---

## 🔥 Key Lessons

* Command Injection = input reaching execution
* Always identify:

  * where input enters
  * where it flows
  * where it executes
* Output behavior determines your strategy:

  * verbose → direct enumeration
  * blind → indirect confirmation
* Shell operators control execution behavior
* Low privilege ≠ useless

---

## ❌ Common Mistakes

* Trying to understand every line of code
* Ignoring blind injection scenarios
* Assuming filters make the app safe
* Forgetting execution context (which user runs the command)

---

## 🔥 Final Mental Model

```text
Find CONTROL
→ Trace FLOW
→ Reach EXECUTION
→ Prove execution
→ Expand access
```
