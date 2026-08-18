# PortSwigger Web Security Academy — Broken Brute-Force Protection, IP Block

## Overview

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Practitioner  
**Lab:** Broken brute-force protection, IP block  
**Tools:** Burp Suite Repeater, Intruder  
**Status:** Solved  

This lab focused on identifying and exploiting a logic flaw in an application's brute-force protection.

The most important takeaway was not the brute-force attack itself, but learning to analyze a security control as a **stateful mechanism**:

> What causes the protection to increment, trigger, and reset?

---

## 1. Initial Attack Surface

The login request exposed two attacker-controlled parameters:

```http
POST /login

username=...
password=...
````

The initial goal was to establish how the authentication mechanism behaved before attempting any automation.

My approach was:

```text
Observation
    ↓
Hypothesis
    ↓
Controlled Experiment
    ↓
Evidence
    ↓
Update Mental Model
    ↓
Next Experiment
```

---

## 2. Establishing a Baseline

I first submitted completely invalid credentials:

```text
username = random
password = random
```

Response:

```text
Status: 200 OK
Message: "Invalid username"
```

I then changed only one variable at a time.

### Valid Username + Invalid Password

```text
username = wiener
password = random
```

Response:

```text
Status: 200 OK
Message: "Incorrect password"
```

### Invalid Username + Valid Password

```text
username = random
password = peter
```

Response:

```text
Status: 200 OK
Message: "Invalid username"
```

This showed that the application produced distinguishable responses depending on whether the supplied username existed.

A reasonable behavioral model was:

```text
Login request
      │
      ▼
Check username
      │
 ┌────┴────┐
 │         │
Invalid   Valid
 │         │
 ▼         ▼
"Invalid   Check password
username"       │
          ┌─────┴─────┐
          │           │
        Wrong       Correct
          │           │
          ▼           ▼
     "Incorrect    Successful
      password"    authentication
```

This represents **observed behavior**, rather than claiming knowledge of the application's exact source-code implementation.

---

## 3. Investigating the Brute-Force Protection

Repeated failed authentication attempts eventually produced:

```text
You have made too many incorrect login attempts.
Please try again in 1 minute(s).
```

Testing showed that approximately three consecutive failures were enough to trigger the protection.

At this point, the important question became:

> What state is the application maintaining, and what causes that state to reset?

Simply knowing the threshold was not enough.

---

## 4. Testing the Reset Condition

I tested the following sequence:

```text
wrong login
wrong login
correct login
```

Using my known account:

```text
wiener:peter
```

A successful authentication reset the failed-login state.

This suggested a state model similar to:

```text
Failed authentication
        ↓
failure state increases

Failed authentication
        ↓
failure state increases

Successful authentication
        ↓
failure state resets
```

This created another question:

> Is the failure state isolated to each username, or can authentication activity involving another account affect it?

---

## 5. Testing the Account Boundary

I deliberately generated failures against the victim account and then authenticated successfully using my own account.

Example:

```text
carlos : wrong_password
carlos : wrong_password

wiener : peter

carlos : wrong_password
carlos : wrong_password

wiener : peter
```

The successful authentication as `wiener` prevented the expected brute-force protection from stopping the continued attempts against `carlos`.

This revealed the core logic flaw.

The security mechanism's state could be influenced by successful authentication activity involving a different account.

This also weakened the hypothesis that the failed-login counter was purely associated with the victim username.

---

## 6. Exploitation Strategy

Instead of sending the entire password list directly:

```text
carlos : candidate1
carlos : candidate2
carlos : candidate3
carlos : candidate4
...
```

I could periodically insert a successful login:

```text
carlos : candidate1
carlos : candidate2
wiener : peter

carlos : candidate3
carlos : candidate4
wiener : peter

carlos : candidate5
carlos : candidate6
wiener : peter
```

The known successful authentication repeatedly reset the relevant brute-force protection state, allowing the attack against the victim account to continue.

The important point is that the inserted `wiener:peter` requests were not random payloads.

They had a specific purpose:

```text
Victim failure
      ↓
Victim failure
      ↓
Approaching threshold
      ↓
Known successful login
      ↓
RESET
      ↓
Continue victim testing
```

---

## 7. Automating with Burp Intruder

Only after validating the mechanism manually did I automate it.

Two request parameters needed synchronized payloads:

```text
username
password
```

Therefore, I used a **Pitchfork attack**.

Pitchfork pairs corresponding entries:

```text
username[1] + password[1]
username[2] + password[2]
username[3] + password[3]
...
```

### Username Payload Structure

```text
carlos
carlos
wiener
carlos
carlos
wiener
...
```

### Password Payload Structure

```text
candidate1
candidate2
peter
candidate3
candidate4
peter
...
```

This generated:

```text
carlos : candidate1
carlos : candidate2
wiener : peter

carlos : candidate3
carlos : candidate4
wiener : peter

...
```

One important detail was keeping both payload lists synchronized.

If one list shifted by even one entry, the username/password combinations would no longer represent the intended experiment.

---

## 8. Detecting Successful Authentication

A successful login generated a different response:

```http
HTTP/2 302 Found
Location: /my-account?id=<username>
Set-Cookie: session=...
Content-Length: 0
```

However, simply searching for `302` responses was insufficient.

The intentionally inserted:

```text
wiener:peter
```

requests also generated successful authentication responses.

Therefore, the meaningful signal was:

```text
username = carlos
        +
candidate password
        +
302 response
        +
Location: /my-account?id=carlos
```

Eventually, one candidate produced this behavior.

I then manually validated the discovered credentials and successfully accessed the victim's account page.

---

## 9. Understanding the Redirect

A successful login did not mean that the server literally moved the browser to another page.

The actual HTTP flow was:

```text
POST /login
    ↓
Server validates credentials
    ↓
HTTP 302 Found
Location: /my-account?id=carlos
    ↓
Browser receives response
    ↓
Browser follows Location header
    ↓
Browser sends another HTTP request
    ↓
Account page returned
```

This distinction is important when interpreting authentication responses.

---

## 10. What I Learned

### Security Controls Have State

Before this lab, it was easy to think about rate limiting simply as:

```text
Too many requests
      ↓
Blocked
```

A better model is:

```text
Request
   ↓
Authentication result
   ↓
Security mechanism updates state
   ↓
Allow / Block / Reset
```

This creates better questions:

* What increments the state?
* What resets it?
* What identity is the state associated with?
* Is it per account, session, client, IP, or something else?
* Can actions involving one identity influence another?

---

### Test Boundaries, Not Only Values

The critical experiment was not merely:

> Does successful authentication reset the counter?

The stronger question was:

> Can successful authentication as one user affect failures generated while attacking another user?

Testing across that identity boundary exposed the logic flaw.

---

### Do Not Overclaim Backend Implementation

The responses showed:

```text
invalid username
      ↓
"Invalid username"

valid username + wrong password
      ↓
"Incorrect password"
```

This supports the hypothesis that different authentication paths are being taken.

However, black-box observations do not reveal the exact source code.

Therefore:

> "The behavior suggests..."

is more accurate than:

> "The backend definitely executes this exact code..."

---

### Automation Comes After Understanding

I could have immediately copied a Pitchfork configuration from a solution.

That would solve the lab without teaching me much.

Instead:

```text
Manual observation
      ↓
Baseline
      ↓
Hypothesis
      ↓
Controlled experiment
      ↓
Understand mechanism
      ↓
Design attack
      ↓
Automate
```

The principle I want to keep:

> **Automation should scale an experiment I understand, not replace the reasoning required to design the experiment.**

---

### Response Anomalies Need Context

A `302` response alone did not prove that I had discovered the victim's password because my known successful logins also generated `302`.

Evidence became stronger when multiple signals agreed:

```text
Victim username
      +
candidate password
      +
302 response
      +
Location pointing to victim account
      +
new authenticated session
      +
manual login validation
```

---

### Tool Choice Should Follow the Experiment

Pitchfork was not used simply because I remembered the tool.

The experiment required:

```text
username[n] ↔ password[n]
```

Two payload sets needed to advance together in synchronized pairs.

Therefore, Pitchfork matched the experimental requirement.

---

## 11. Final Mental Model

```text
                LOGIN ATTEMPT
                     │
                     ▼
              Authentication
                     │
            ┌────────┴────────┐
            │                 │
         FAILURE           SUCCESS
            │                 │
            ▼                 ▼
     Increase failure       Reset
          state             state
            │
            ▼
      Threshold reached?
          │       │
         No      Yes
          │       │
      Continue   Block
```

The vulnerable behavior allowed a known successful login to repeatedly reset the protection:

```text
Victim failure
Victim failure
      │
      ▼
Known valid login
      │
      ▼
    RESET
      │
      ▼
Victim failure
Victim failure
      │
      ▼
Known valid login
      │
      ▼
    RESET
      │
      ▼
    Repeat
```

This transformed a supposedly limited number of authentication attempts into a repeatable brute-force process.

---

## 12. Methodology Reinforced

The methodology used throughout this lab was:

```text
OBSERVATION
    ↓
QUESTION
    ↓
HYPOTHESIS
    ↓
CONTROLLED EXPERIMENT
    ↓
RESULT
    ↓
UPDATE CONFIDENCE
    ↓
REFINE BACKEND MODEL
    ↓
NEXT EXPERIMENT
    ↓
AUTOMATION
    ↓
EXPLOITATION
    ↓
VALIDATION
```

The main improvement was moving away from:

> "What payload or vulnerability should I try?"

toward:

> "What mechanism could explain this behavior, and what experiment would distinguish between the possible explanations?"

---

## Reflection

The technical exploit in this lab became relatively simple once the logic flaw was understood.

The more valuable part was the investigation process.

Instead of immediately asking:

> "How do I bypass this rate limit?"

I learned to ask:

> **"What state does this security mechanism maintain, what events change that state, and can attacker-controlled actions manipulate those transitions?"**

This is the main lesson I want to carry into future black-box web security labs.

```
```
