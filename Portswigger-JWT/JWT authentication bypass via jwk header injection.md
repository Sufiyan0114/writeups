# Lab Writeup: JWT authentication bypass via jwk header injection

**Category:** JWT Attacks | **Difficulty:** Practitioner | **Date Completed:** 26/03/2026 | **Platform:** PortSwigger Web Security Academy

---

## 1. Objective
Gain unauthorized access to the administrative interface (`/admin`) by injecting a self-signed JSON Web Key (JWK) into the JWT header and delete the user `carlos`.

---

## 2. Vulnerability Overview
* **Vulnerability Name:** JWT Authentication Bypass (JWK Header Injection)
* **CWE Reference:** [CWE-345: Insufficient Verification of Data Authenticity](https://cwe.miter.org/data/definitions/345.html)
* **OWASP Category:** A07:2021 – Identification and Authentication Failures

**Description:** The `jwk` (JSON Web Key) header parameter is used by servers to identify the public key needed to verify a token's signature. A vulnerability exists if the server blindly trusts the key provided within the JWT header itself. An attacker can generate their own RSA key pair, sign a malicious token with their private key, and embed the matching public key in the `jwk` header to bypass authentication.

---

## 3. Reconnaissance & Discovery
1. Logged into the application using standard credentials (`wiener:peter`).
2. Intercepted the session token in **Burp Suite Proxy**.
3. **Observation:** The JWT uses the `RS256` algorithm.
4. **Initial Test:** Modifying the `sub` claim to `administrator` manually caused a signature mismatch error, confirming that the signature is being validated.

**Tool(s) Used:** Burp Suite Professional, JWT Editor Extension.

---

## 4. Exploitation Steps

### Step 1: Generate an Attacker Key Pair
In the **JWT Editor Keys** tab:
1. Clicked **New RSA Key** -> **Generate**.
2. This created a unique Private/Public key pair for signing malicious tokens.

### Step 2: Modify JWT Payload
In **Burp Repeater**, switched to the **JSON Web Token** tab:
* Changed the `sub` claim from `wiener` to `administrator`.

```json
{
    "iss": "portswigger",
    "exp": 1774519184,
    "sub": "administrator"
}
Step 3: Execute JWK Injection Attack
Using the Attack feature:

Clicked Attack -> Embedded JWK.

Selected the RSA key generated in Step 1.

Observation: The extension automatically updated the header to include the jwk parameter containing the attacker's public key.

Modified Header (Redacted for Privacy):

JSON
{
    "kid": "9813720a-xxxx-xxxx-xxxx-57529291bb39",
    "alg": "RS256",
    "typ": "JWT",
    "jwk": {
        "kty": "RSA",
        "e": "AQAB",
        "kid": "9813720a-xxxx-xxxx-xxxx-57529291bb39",
        "n": "9X0wcDrs53QBDwCL8lhxw1_NSr9eeATowa1P6DpP5__[REDACTED]"
    }
}
Step 4: Access Admin Panel
Sent the request to /admin.

Response: HTTP/2 200 OK.

The server trusted the embedded key and granted administrative access.

Step 5: Lab Solved ✅
Identified the delete URL and sent the final request:
GET /admin/delete?username=carlos HTTP/2

Final Request (Redacted):

HTTP
GET /admin/delete?username=carlos HTTP/2
Host: [LAB-ID].web-security-academy.net
Cookie: session=eyJraWQiOiI5ODEzNzIwYS1bUkVEQUNURURdLmV5SnBjM01pT2pXYjNK...[REDACTED]
5. Root Cause Analysis
The server's JWT validation logic fails to verify the origin of the verification key. By trusting the jwk parameter in the header, the server allows the "claimant" (the user) to provide the very key used to verify their own identity, creating a complete breakdown of the trust model.

6. Impact
Critical: Full administrative bypass and unauthorized access to sensitive data/actions.

Severity: High (Practitioner).

7. Remediation
Whitelist Trusted Keys: The server should only use public keys from a pre-defined, server-side trusted store (e.g., a local file or a hardcoded list).

Reject jwk Parameter: Disable support for the jwk header parameter unless there is a specific, secure use case for dynamic key discovery.

8. Key Takeaways
Asymmetric signing (RS256) is only secure if the Public Key is retrieved from a trusted source.

Always check if a server allows "Dynamic Key Injection" via jwk or jku headers.

9. References
PortSwigger Academy: JWT Header Injection

RFC 7515: JSON Web Signature (JWS)
