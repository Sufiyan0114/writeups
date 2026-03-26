Lab Writeup: JWT authentication bypass via flawed signature verification
Category: JWT Attacks | Difficulty: Apprentice | Date Completed: 26/03/2026 | Platform: PortSwigger Web Security Academy

1. Objective
Log into the administrator account by exploiting a flaw in JWT signature verification and use the elevated privileges to delete the user carlos.

2. Vulnerability Overview
Vulnerability Name: JWT Authentication Bypass (Insecure Signature Handling)

CWE Reference: CWE-345: Insufficient Verification of Data Authenticity

OWASP Category: A07:2021 – Identification and Authentication Failures

What is this vulnerability?
JSON Web Tokens (JWT) rely on a cryptographic signature to ensure the data in the payload hasn't been tampered with. However, some servers trust the alg (algorithm) header provided by the user. If an attacker changes the algorithm to none, the server may skip the signature verification entirely, trusting whatever is written in the payload (like changing a username to administrator).

3. Reconnaissance & Discovery
Logged into the application using my own credentials (wiener:peter).

Captured the GET /my-account request in Burp Suite Proxy.

Observation: The session is managed via a session cookie that contains a JWT.

Decoded the JWT using the Inspector panel:

Header: {"alg": "RS256", "typ": "JWT"}

Payload: {"iss": "portswigger", "sub": "wiener", ...}

Attempted to access /admin directly with the original token, but the server returned a 401 Unauthorized or redirected me away.

Tool(s) Used: Burp Suite Professional, JWT Editor Extension.

4. Exploitation Steps
Step 1: Prepare the Attack Request
Sent the /my-account request to Burp Repeater and changed the path to /admin.

Step 2: Modify JWT Claims (Privilege Escalation)
Using the JSON Web Token tab (JWT Editor Extension):

Changed the sub claim from wiener to administrator.

This targets the admin user identity.

Step 3: Bypass Signature Verification
Used the Attack menu in the JWT Editor:

Selected "none" signing algorithm.

Result: The extension automatically changed the header to {"alg": "none"} and stripped the signature from the end of the token while keeping the trailing dot.

Modified JWT Structure:

Plaintext
// Header (Base64)
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.
// Payload (Base64)
eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3NDUxNzY5Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.
// Signature
(Empty)
Step 4: Verify Admin Access
Sent the request to /admin.

Response: HTTP/2 200 OK.

The page rendered successfully, showing the "Admin Panel" and the list of users.

Step 5: Perform Admin Action (Lab Solved ✅)
Identified the delete link for carlos: /admin/delete?username=carlos.
Changed the Repeater path to this URL and sent the request.

Response: HTTP/2 302 Found.

The user carlos was successfully deleted, and the lab was marked as solved.

5. Root Cause Analysis
The application's JWT library or implementation was configured to allow the none algorithm. By design, alg: none is meant for "unsecured" environments, but in a production authentication system, it allows an attacker to dictate the security level. The server failed to enforce that a strong, specific algorithm (like RS256) must be used for all incoming tokens.

6. Impact
Critical: Full account takeover of any user (including administrators).

Privilege Escalation: Standard users can perform administrative actions.

Data Integrity: All user data can be modified or deleted.

Severity: High/Critical

7. Remediation
Disable "none" Algorithm: Configure the JWT parsing library to strictly reject any token where alg is set to none.

Algorithm Whitelisting: Explicitly allow only a single, strong algorithm (e.g., RS256 or HS256) for verification.

Strict Validation: Ensure the signature is verified before any data in the payload is processed by the application logic.

8. Key Takeaways
The none attack is a classic example of "User-Controlled Input" affecting security logic.

Always check the structure of a JWT after a "none" attack; that trailing dot is small but mandatory.

Automated tools like JWT Editor are helpful, but understanding the manual structure (Header.Payload.) is essential for troubleshooting.
