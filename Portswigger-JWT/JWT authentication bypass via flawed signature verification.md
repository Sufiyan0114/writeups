# Lab Writeup: JWT authentication bypass via flawed signature verification

**Category:** JWT Attacks | **Difficulty:** Apprentice | **Date Completed:** 26/03/2026 | **Platform:** PortSwigger Web Security Academy

---

## 1. Objective
Log into the **administrator** account by exploiting a flaw in JWT signature verification and use the elevated privileges to delete the user `carlos`.

---

## 2. Vulnerability Overview
* **Vulnerability Name:** JWT Authentication Bypass (Insecure Signature Handling)
* **CWE Reference:** [CWE-345: Insufficient Verification of Data Authenticity](https://cwe.miter.org/data/definitions/345.html)
* **OWASP Category:** A07:2021 – Identification and Authentication Failures

**What is this vulnerability?** JSON Web Tokens (JWT) rely on a cryptographic signature to ensure data integrity. However, some servers trust the `alg` (algorithm) header provided by the user. If an attacker changes the algorithm to `none`, the server may skip verification, trusting any modified payload (e.g., changing a username to `administrator`).

---

## 3. Reconnaissance & Discovery
* Logged into the application using standard credentials (`wiener:peter`).
* Captured the `GET /my-account` request in **Burp Suite Proxy**.
* **Observation:** The session is managed via a `session` cookie containing a JWT.
* Decoded the JWT using the **Inspector** panel:
    * **Header:** `{"alg": "RS256", "typ": "JWT"}`
    * **Payload:** `{"iss": "portswigger", "sub": "wiener"}`

**Tool(s) Used:** Burp Suite Professional, JWT Editor Extension.

---

## 4. Exploitation Steps

### Step 1: Prepare the Attack Request
Sent the `/my-account` request to **Burp Repeater** and changed the path to `/admin`.

### Step 2: Modify JWT Claims
Using the **JSON Web Token** tab:
* Changed the `sub` claim from `wiener` to `administrator` to target the admin identity.

### Step 3: Bypass Signature Verification
Used the **Attack** menu in the JWT Editor:
* Selected **"none" signing algorithm**.
* **Result:** The extension changed the header to `{"alg": "none"}` and stripped the signature, ensuring a trailing dot remains.

**Modified JWT Structure:**
```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3NDUxNzY5Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.
Step 4: Verify Admin Access
Sent the request to /admin.

Response: HTTP/2 200 OK.

The Admin Panel loaded successfully.

Step 5: Lab Solved ✅
Identified the delete link for carlos: /admin/delete?username=carlos.
Changed the Repeater path to this URL and sent the request.

Response: HTTP/2 302 Found.

User carlos was deleted successfully.

5. Root Cause Analysis
The application blindly trusts the alg header. By allowing the none algorithm, the server bypasses the only security mechanism (the signature) that prevents unauthorized modification of user claims.

6. Impact
Critical: Full account takeover and privilege escalation.

Severity: High

7. Remediation
Disable "none": Configure the JWT library to strictly reject alg: none.

Whitelist Algorithms: Only allow specific strong algorithms like RS256.

8. Key Takeaways
Always check if the server enforces the signature check.

Small structural details, like the trailing dot in alg: none, are critical for a successful bypass.

9. References
PortSwigger Academy: JWT Attacks

OWASP: JSON Web Token Cheat Sheet


---

### Pro Tip for GitHub:
Jab aap GitHub par naya file banayein, toh extension **`.md`** zaroori hai. Paste karne ke baad upar **"Preview"** tab par click karke check kar lena, woh bilkul professional dikhega.

**Kya aap next lab (Weak Signing Key/Hashcat) ka writeup bhi isi format mein chahte hain?**
