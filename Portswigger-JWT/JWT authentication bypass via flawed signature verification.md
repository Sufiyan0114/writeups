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
