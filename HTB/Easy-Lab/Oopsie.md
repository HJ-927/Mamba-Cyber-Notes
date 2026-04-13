# HTB — Oopsie

## Overview
- **Difficulty:** Easy
- **Category:** Web + Linux
- **Date Completed:** 13 April 2026
- **Time Taken (First Run):** 5 hours
- **Time Taken (Redo):** 30 minutes

---

## Attack Chain Summary

Enumerated open ports. Connected to HTTP port and searched 
for login page through page source. Changed id in the URL 
parameter to get admin info. Modified cookies to access 
the upload page. Created and uploaded a PHP webshell. 
Executed commands through the webshell and found robert's 
credentials in db.php. Logged in as robert via SSH. Found 
a SUID binary calling cat without absolute path and used 
PATH hijack to get root.

---

## Attack Flow

→Enumerate Ports
→ HTTP Discovery
→ Page Source (found hidden login path)
→ Login as Guest
→ IDOR (id parameter)
→ Cookie Tampering (escalate to admin)
→ Upload PHP Webshell
→ RCE via curl
→ Found robert's creds in db.php
→ SSH as robert
→ SUID Enumeration
→ PATH Hijack → Root

---

## Phase 1 — Scanning Ports

**What I did:**
Used nmap to enumerate open ports.

**Why I chose this:**
nmap shows me all the open ports. I can know which ports 
are enumeration-friendly, which ones allow me to gather 
information, and which ones help me understand the machine 
better.

**What I found:**
Two open ports — port 22 (SSH) and port 80 (HTTP).

**What it means for the next step:**
I decided to go with HTTP first because it is an 
application layer protocol — I can interact with it 
directly from the browser and look for attacker-controlled 
surfaces like parameters, cookies, and forms. SSH requires 
credentials which I don't have yet, so it's not useful 
at this stage.

---

## Phase 2 — Web Enumeration (Unauthenticated)

**What I did:**
Connected to http://target-ip and walked through the page. 
Tried the visible buttons. Checked the Network tab in 
inspect. Ran gobuster. Read the page source.

**Why I chose this:**
- **Walk through page** — check if any buttons are 
  interactive and whether I can control anything in the 
  URL parameter
- **Network tab** — the browser UI only shows what the 
  developer wanted me to see. The Network tab shows the 
  real browser-server conversation — endpoints, methods, 
  cookies, headers, and redirects that are invisible from 
  the page alone
- **Gobuster** — web servers don't advertise hidden 
  directories, so I brute force common path names to find 
  attack surface that isn't linked anywhere on the page
- **Page source** — reveals hidden files, linked 
  JavaScript, and routes not visible from the rendered UI

The order matters: visible UI → Network tab → page source 
→ linked JavaScript → fuzzing. Each layer reveals what 
the previous one missed.

**What I found:**
Visible buttons had no useful attack surface. Network tab 
showed nothing immediately useful. Gobuster returned some 
outputs but nothing actionable yet. Page source revealed 
`/cdn-cgi/login/script.js` — a JavaScript file inside a 
login-related directory. JavaScript files in login 
directories often contain endpoints, hidden routes, and 
form actions. I navigated to `/cdn-cgi/login` and it 
worked — confirmed active login page.

Gobuster missed this because the filename wasn't in the 
wordlist, and I wasn't fuzzing for .js extensions at that 
point. This is why page source matters — gobuster can miss 
things that are directly referenced in the HTML.

**What it means for the next step:**
Found a login page. Can attempt guest login and enumerate 
authenticated attack surface.

---

## Phase 3 — Post-Login Enumeration as Guest

**What I did:**
Logged in as guest. Walked through the authenticated 
pages. Checked URL parameters and inspected the Network 
tab for cookies and request structure.

**Why I chose this:**
After login the application assigns session cookies and 
exposes user-specific parameters. I need to understand 
what the server trusts from the client side now that I 
have a session.

**What I found:**
- URL parameter `id` was something I could change
- Network tab showed two cookies: `role=guest` 
  and `user=2233`
- There was an uploads button but it only allowed 
  super admin access

**What it means for the next step:**
Both the id parameter and the cookies are attacker-controlled 
inputs the server appears to trust. I should try changing 
the id to see other users' data, then use that info to 
attempt cookie tampering and get into the upload page.

---

## Phase 4 — IDOR + Cookie Tampering

**What I did:**
Changed the `id` value in the URL parameter and found 
admin account info including the admin's user value. Used 
browser console to modify the cookies — changed `role` 
to `admin` and `user` to the admin's value. This gave me 
access to the upload page.

**Why I chose this:**
When I changed the id parameter, I didn't become the other 
user — I stayed as guest but the server showed me that 
user's data anyway. This is IDOR — the server authenticated 
me correctly as guest, but failed to check whether I was 
authorized to read that object. Authentication answers 
"who are you." Authorization answers "what are you allowed 
to access." The app failed the second one.

Cookie tampering worked because the server used the 
cookie values I sent directly for access control without 
checking server-side. I used the browser console because 
it lets me run JavaScript directly including 
`document.cookie` to overwrite cookie values. Note: after 
modifying cookies, the Network tab still shows old values 
because it records historical requests, not current browser 
state. The Application tab shows the actual current cookie 
state.

**What I found:**
Admin account details through IDOR. Access to upload page 
after cookie modification.

**What it means for the next step:**
I can upload files to the server. Need to check if the 
server will execute PHP — if yes, I get remote code 
execution.

---

## Phase 5 — PHP Webshell + RCE

**What I did:**
Created a PHP webshell:
```php
<?php system($_GET['cmd']); ?>
```
Uploaded it through the upload page. Accessed it at 
`/uploads/attacker.php` and tested execution by passing 
commands through the `cmd` parameter. Used curl in 
terminal to execute commands like `id`, `pwd`, `ls -la` 
to enumerate the system.

**Why I chose this:**
For a web shell to work, three conditions must be met 
separately:
- **WRITE** — can I place the file on the server
- **REACH** — can I access it by URL
- **EXECUTE** — will the server interpret it as PHP

All three were satisfied here.

Important lesson: `/uploads` returning 403 did not mean 
dead end. 403 just means the server refused that specific 
request — in this case directory listing was disabled. 
Direct file access at `/uploads/attacker.php` was still 
allowed. 403 means "blocked somehow," not "this path is 
useless."

I used curl instead of the browser URL bar because it 
bypasses browser rendering quirks and makes repetitive 
commands cleaner. Curl and browser are both just separate 
clients talking to the same server — they don't interact 
with each other. Server state changed when browser 
uploaded the file, then curl simply retrieves and 
interacts with it after.

**What I found:**
Confirmed RCE as `www-data`. Found `db.php` containing 
robert's plaintext password. Used `view-source:` in 
browser to read db.php cleanly because the browser was 
rendering special characters instead of showing raw text.

**What it means for the next step:**
Pivot to robert using his credentials via SSH.

---

## Phase 6 — SSH as Robert

**What I did:**
Logged in as robert via SSH using the password found in 
db.php. Checked identity using `id` and `pwd`.

**Why I chose this:**
After getting a foothold, I always check identity first. 
Knowing who I am tells me what I can access, what groups 
I belong to, and what my next enumeration priorities are.

**What I found:**
I am in the `bugtracker` group.

**What it means for the next step:**
Group membership matters — being in a specific group often 
means access to specific binaries or files. Need to 
enumerate for misconfigurations and privilege escalation 
vectors.

---

## Phase 7 — Privilege Escalation (PATH Hijack)

**What I did:**
Ran `sudo -l` and `find / -perm -4000 -type f 2>/dev/null` 
to look for privesc vectors. Found a suspicious SUID binary 
at `/usr/bin/bugtracker`. Used `ls -la` and `strings` to 
understand its behavior.

**Why I chose this:**
SUID binaries run as their file owner regardless of who 
executes them. If the owner is root, anything the binary 
does happens as root. `strings` lets me read readable text 
from a binary without fully reversing it — I can see what 
system calls and commands it uses.

**What I found:**
`strings` output showed `system()`, `getuid`, and `cat` 
called without absolute path. When a binary calls a command 
without absolute path, it searches `$PATH` directories in 
order to find it. If I put a fake binary earlier in `$PATH`, 
the SUID binary will execute my fake version as root instead.

Performed PATH hijack:
```bash
echo '/bin/sh' > /tmp/cat
chmod +x /tmp/cat
export PATH=/tmp:$PATH
/usr/bin/bugtracker
```

Got root shell.

---

## Concepts Learned

- **Recon escalation order** — visible UI → Network tab → 
  page source → linked JS → fuzzing. Each layer reveals 
  what the previous missed. Don't skip layers.

- **Network tab vs Application tab** — Network tab records 
  historical requests. After modifying cookies, check 
  Application tab for current browser cookie state, not 
  Network tab.

- **IDOR vs Authentication** — changing id kept my guest 
  session but exposed other users' data. Authorization 
  flaw, not authentication flaw. Auth = who are you. 
  Authz = what can you access.

- **403 is not a dead end** — means server refused that 
  specific request. Directory listing blocked doesn't mean 
  files inside are inaccessible.

- **WRITE + REACH + EXECUTE** — three separate conditions 
  for web shell RCE. Debug each one separately when 
  upload exploits fail.

- **Post-exploitation order** — after landing as web user, 
  enumerate application files first. The app context that 
  gave you access is your richest source of leverage before 
  jumping to system-wide enumeration.

- **SUID + relative path = PATH hijack** — SUID binary 
  calling commands without absolute path can be hijacked 
  by controlling $PATH.

---

## Mistakes I Made

- Found the SUID binary early as `www-data` and spent time 
  on it before realizing I couldn't execute it. Should have 
  enumerated application files first before jumping to 
  system-wide privesc vectors. Wasted significant time.

- Treated `/uploads` 403 as a dead end initially and didn't 
  immediately try direct file access at 
  `/uploads/attacker.php`.

- Modified cookies via console but got confused when Network 
  tab still showed old values. Didn't know to check 
  Application tab for current cookie state.

---

## Weaknesses Identified

- No instinct yet for post-exploitation enumeration order. 
  Jumped to SUID before exhausting application context.

- Recon escalation order not automatic yet — needed hints 
  to try page source after gobuster didn't give clear 
  direction.

- Need to learn Burp Suite. Console works but Burp gives 
  cleaner and more reliable request control, especially 
  for cookie path scoping issues.

---

## What I Would Do Differently

Follow recon escalation order strictly without skipping 
layers. After getting web shell, enumerate application 
files and config files before touching SUID binaries. 
Verify current user context and group membership 
immediately after every pivot.

---

## Key Takeaways

- Page source reveals what gobuster misses
- 403 means blocked, not dead end
- IDOR = authorization flaw, not authentication flaw  
- Web shell needs WRITE + REACH + EXECUTE — all three
- After foothold, enumerate app context before system context
- SUID + relative path command = always check for PATH hijack
