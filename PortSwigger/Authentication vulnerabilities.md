# PortSwigger Web Security Academy — Authentication Lab 1

## Username Enumeration via Different Responses

**Platform:** PortSwigger Web Security Academy  
**Topic:** Authentication Vulnerabilities  
**Difficulty:** Apprentice  
**Status:** ✅ Completed  
**Date:** 2 August 2026

---

## Overview

This was my first PortSwigger Web Security Academy lab.

The objective was to exploit an authentication mechanism that leaks whether a username exists through different login responses, identify a valid username, determine the corresponding password, and access the user's account.

The exploitation itself was relatively straightforward because I had previously practiced Burp Suite Intruder and Sniper attacks on TryHackMe.

However, the main value of this lab was not simply finding the credentials. It helped reinforce a more disciplined methodology:

> **Establish baseline → form hypothesis → change one variable → observe → find evidence → validate → conclude**

I also learned several details about Burp response analysis that I had previously overlooked.

---

# 1. Environment Setup

Before starting the actual lab, I needed to configure Burp Suite.

Burp's built-in browser did not launch properly on my machine, so I configured Firefox manually to proxy traffic through Burp.

The traffic flow became:

```text
Firefox
   ↓
127.0.0.1:8080
   ↓
Burp Proxy
   ↓
PortSwigger Lab
```

Because the lab uses HTTPS, Firefox initially rejected Burp's generated certificates.

I therefore downloaded Burp's CA certificate from:

```text
http://burp
```

and imported it into the Firefox testing profile.

After this setup, Burp could inspect the HTTPS requests and responses between Firefox and the PortSwigger lab.

### Tools Used

- Burp Suite Community Edition
- Burp Proxy / HTTP History
- Burp Repeater
- Burp Intruder
- Firefox
- PortSwigger Web Security Academy

---

# 2. Mapping the Login Request

The login functionality generated a request similar to:

```http
POST /login HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

username=<username>&password=<password>
```

The primary attacker-controlled inputs relevant to the authentication hypothesis were:

```text
username
password
```

Although other HTTP components such as the method, path, headers, and cookies can also potentially be modified, they were not the variables I wanted to test at this stage.

---

# 3. Establishing a Baseline

Instead of immediately using a wordlist, I first submitted deliberately invalid credentials.

Example:

```text
username = random-invalid-user
password = hahaha
```

The response contained:

```text
Status: 200 OK
Message: "Invalid username"
No redirect
```

This became my baseline for a nonexistent username.

This step reinforced an important principle:

> Before searching for abnormal behavior, I need to understand what normal failure looks like.

---

# 4. Testing the Username Parameter

My next hypothesis was:

> The application may expose whether a username exists through different authentication responses.

To test this properly, I changed only one variable.

I kept:

```text
password = hahaha
```

constant while changing:

```text
username
```

Multiple random usernames continued producing:

```text
Invalid username
```

Therefore, there was no meaningful difference yet.

At this point, manually inventing additional usernames would be inefficient, so I moved to Burp Intruder.

---

# 5. Username Enumeration with Burp Intruder

I sent the login request to Intruder and configured only the username as the payload position.

Conceptually:

```text
username=§PAYLOAD§&password=hahaha
```

I then loaded the candidate username list provided by the lab.

Most requests returned the same baseline response:

```text
Invalid username
```

However, one username produced:

```text
Invalid password
```

This was much stronger evidence than simply seeing a different response length.

The authentication flow appeared to behave like:

```text
Username supplied
      ↓
Does username exist?
      │
      ├── NO → "Invalid username"
      │
      └── YES
             ↓
        Check password
             ↓
        "Invalid password"
```

The application was therefore leaking internal authentication state.

This created a **username enumeration oracle**.

---

# 6. Password Testing

After identifying the valid username, I reversed the experiment.

This time:

```text
Controlled variable:
username = valid username

Variable under test:
password
```

I first manually tested multiple incorrect passwords.

They produced the same baseline behavior:

```text
200 OK
"Invalid password"
```

Once I understood the normal password-failure response, I used Burp Intruder again.

The password became the only payload position:

```text
username=<valid-user>&password=§PAYLOAD§
```

I loaded the candidate password list and started the attack.

---

# 7. Identifying Successful Authentication

Most candidate passwords produced:

```text
200 OK
"Invalid password"
```

One request behaved very differently:

```http
HTTP/2 302 Found
Location: /my-account?id=<username>
Set-Cookie: session=<session-token>
Content-Length: 0
```

This was a much stronger signal than merely seeing a different response length.

The application had transitioned from:

```text
Login failure
→ 200
→ login/error page
```

to:

```text
Successful authentication
→ 302
→ redirect to /my-account
```

I manually entered the discovered credentials into the browser and successfully accessed the account page.

This confirmed the hypothesis.

---

# 8. Intruder Length vs Content-Length

One unexpected observation during this lab was that Burp Intruder displayed response lengths such as:

```text
3354
3441
```

while the HTTP response contained values such as:

```http
Content-Length: 3246
```

Initially, I expected these numbers to match.

They do not necessarily represent the same thing.

## Content-Length

`Content-Length` represents the size of the HTTP response body.

For example:

```http
Content-Length: 3246
```

means the response body contains 3246 bytes.

## Burp Intruder Length

Burp's Intruder `Length` value is used to compare the overall recorded responses and can be affected by additional response metadata.

For example, two failed authentication responses may contain the same:

```http
Content-Length: 3246
```

while one additionally contains:

```http
Set-Cookie: session=<token>
```

This additional header can cause Burp's displayed Intruder length to differ even though the response body itself has not meaningfully changed.

---

# 9. Response Length Is a Signal, Not Proof

This was one of the most useful lessons from the lab.

A different response length does **not** automatically mean:

```text
different length = correct credential
```

Response lengths can change because of:

- different response bodies
- different error messages
- additional headers
- `Set-Cookie`
- `Location`
- redirects
- other application behavior

Therefore, response length should mainly be treated as a way to identify potential outliers.

After finding an outlier, I should inspect **why** the response is different.

Useful signals include:

```text
Status code
Response body
Error message
Content-Length
Intruder Length
Location header
Set-Cookie
Redirect behavior
Subsequent application access
```

---

# 10. Observation vs Evidence vs Conclusion

Another important lesson was separating these concepts.

A weak reasoning chain would be:

```text
Observation:
Different response length

Conclusion:
Correct password
```

That skips the evidence.

A stronger chain is:

```text
Observation:
One response differs from the baseline.

        ↓

Investigation:
The response is 302 instead of 200.

        ↓

Evidence:
Location points to /my-account.
Authentication error disappears.
Session state changes.

        ↓

Validation:
Use the credentials manually.

        ↓

Result:
Account page is accessible.

        ↓

Conclusion:
The credentials are valid.
```

This distinction is important because an observable difference does not automatically explain **why** the difference occurred.

---

# 11. Methodology Learned

The most important takeaway from this lab was not the specific Intruder attack.

It was the methodology.

```text
Identify input surface
        ↓
Establish baseline
        ↓
Observe behavior
        ↓
Ask a question
        ↓
Form hypothesis
        ↓
Change ONE relevant variable
        ↓
Compare against baseline
        ↓
Identify meaningful signal
        ↓
Investigate the cause
        ↓
Validate hypothesis
        ↓
Automate when justified
        ↓
Manually confirm result
        ↓
Conclusion
```

I should avoid this approach:

```text
See parameter
→ dump wordlist
→ find different length
→ assume vulnerability
```

Instead:

```text
Understand behavior
→ design experiment
→ control variables
→ observe difference
→ understand difference
→ validate
→ automate
→ conclude
```

---

# 12. Mistakes and Improvements

## Mistake 1 — Focusing Too Much on Response Length

Initially, I gave response length too much importance.

### Improvement

Treat response length as a **signal for investigation**, not proof of a vulnerability or successful authentication.

---

## Mistake 2 — Mixing Attacker-Controlled Surfaces With Relevant Variables

An attacker may technically modify many parts of an HTTP request:

```text
Method
Path
Headers
Cookies
Parameters
Body
```

However, changing everything simultaneously creates a poor experiment.

### Improvement

Identify the variable relevant to the current hypothesis and keep the other important variables constant.

For username enumeration:

```text
Variable:
username

Controlled:
password
method
path
session state where possible
```

---

## Mistake 3 — Moving From Observation Toward Conclusion Too Quickly

A different response does not automatically explain what happened.

### Improvement

Use:

```text
Observation
→ Evidence
→ Hypothesis
→ Validation
→ Conclusion
```

and avoid:

```text
Observation
→ Conclusion
```

---

# 13. Reflection

The exploitation technique in this lab felt relatively easy because I had previously practiced Burp Intruder/Sniper-style attacks on TryHackMe.

The lab therefore did not introduce a completely new attack technique.

However, it was still valuable because it improved my **testing discipline**.

Previously, it would have been easy to think:

```text
Load wordlist
→ run Intruder
→ sort by length
→ find credential
```

After this lab, I want my reasoning to become:

```text
What is the baseline?

What variable am I testing?

What behavior am I expecting?

What changed?

Why did it change?

Does that difference actually support my hypothesis?

How can I validate it?
```

The goal is not simply to become faster at finding unusual responses.

The goal is to understand what those responses reveal about the application's internal behavior.

---

# Key Takeaway

> **Automation should scale an experiment that I already understand. It should not replace the reasoning required to design the experiment.**

And:

> **A response difference is a signal. Understanding and validating what caused that difference turns the signal into evidence.**

---

## Progress

**PortSwigger Web Security Academy**

- [x] Authentication — Username Enumeration via Different Responses
- [ ] Continue Authentication Vulnerabilities

First PortSwigger lab completed. ✅
