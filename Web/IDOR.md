# IDOR (Insecure Direct Object Reference)

## 🧠 Definition

IDOR is an authorization vulnerability where a user controls a reference to a resource (e.g., user ID, file, account), and the server fails to verify that the authenticated user is authorized to access that resource.

---

## 🧱 Types of IDOR

### 🟢 Read IDOR

Allows an attacker to access unauthorized data.

**Example:**
GET /api/user?id=1002
→ returns another user's profile

---

### 🔴 Action (Write) IDOR

Allows an attacker to modify or perform actions on unauthorized resources.

**Example:**
POST /api/update-email
user_id=2002
email=[hacker@gmail.com](mailto:hacker@gmail.com)

→ changes another user's email

---

## 🎯 Control Points (Attack Surface)

* URL parameters (?id=123)
* REST path (/user/123)
* POST body (user_id=123)
* JSON data ({"id":123})
* Cookies (user_id=123)
* Headers (X-User-ID: 123)
* File names (file=report.pdf)

---

## ⚠️ Obfuscation ≠ Security

Encoding and hashing do not prevent IDOR — they only change how the identifier is represented.

* Encoding: reversible (e.g., Base64)
* Hashing: may be predictable or brute-forceable

If authorization is not enforced, the system is still vulnerable.

---

## ⚔️ IDOR Testing Workflow

1. Intercept request (DevTools / Burp)
2. Identify resource reference (id, file, account)
3. Modify the value
4. Observe response differences
5. Confirm missing authorization

---

## 🧪 Lab Example (TryHackMe)

**Endpoint:**
GET /customer?id=50

**Response:**
{
"id": 50,
"username": "hahaha",
"email": "[hahaha@gmail.com](mailto:hahaha@gmail.com)"
}

**Test:**
GET /customer?id=1

**Result:**
Server returns another user's data:
{
"id": 1,
"username": "...",
"email": "..."
}

This confirms IDOR due to missing authorization checks.

---

## 🔥 Key Insight

IDOR is not about guessing IDs — it is about broken authorization. Even encoded, hashed, or unpredictable identifiers are vulnerable if access control is not enforced.
