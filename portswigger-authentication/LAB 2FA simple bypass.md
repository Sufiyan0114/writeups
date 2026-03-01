# Lab Writeup: 2FA Simple Bypass

**Category:** Authentication
**Difficulty:** Apprentice
**Date Completed:** 27/02/2026
**Platform:** PortSwigger Web Security Academy

---

## 1. Objective

Log into Carlos's account by bypassing the two-factor authentication (2FA) step, without having access to Carlos's email/OTP.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Two-Factor Authentication — Logic Bypass
**CWE Reference:** CWE-287: Improper Authentication
**OWASP Category:** A07:2021 – Identification and Authentication Failures

**What is this vulnerability?**
Two-factor authentication is supposed to add a second layer of security after a correct username/password is entered. In this lab, the application grants a partial session after the first factor (password) is verified. If the application does not enforce that the 2FA step must be completed before giving access to authenticated pages, an attacker who knows the victim's credentials can skip the 2FA screen entirely by directly navigating to a post-login page.

---

## 3. Reconnaissance & Discovery

- Logged into the application using my own credentials (`wiener:peter`) to understand the normal login flow.
- Observed that after entering correct credentials, the application redirects to `/login2` (the 2FA OTP page).
- After completing 2FA, the application redirects to `/my-account?id=wiener`.
- **Key observation:** The session cookie appears to be set *before* the 2FA check is completed. This suggests the server trusts the session after step 1.

**Tool(s) Used:** Burp Suite Community Edition, Browser

---

## 4. Exploitation Steps

### Step 1: Understand Normal Authentication Flow
Logged in as `wiener:peter`. Observed the full flow:
1. `POST /login` → Credentials accepted → Redirected to `/login2`
2. `POST /login2` → OTP submitted → Redirected to `/my-account?id=wiener`

### Step 2: Log in as Carlos (First Factor Only)
Entered Carlos's credentials on the login page:
- **Username:** `carlos`
- **Password:** `montoya`

The application accepted them and redirected to `/login2` (the 2FA OTP page).

**Request:**
```
POST /login HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&password=montoya
```

**Response:**
```
HTTP/2 302 Found
Location: /login2
Set-Cookie: session=<new-session-token>; Secure; HttpOnly
```

### Step 3: Skip the 2FA Step Entirely
Instead of waiting for the OTP prompt, I manually navigated to:
```
/my-account?id=carlos
```

The application loaded Carlos's account page without requesting the OTP.

### Step 4: Lab Solved ✅
The 2FA page was completely skipped. The server accepted the session from Step 1 as fully authenticated.

---

## 5. Root Cause Analysis

The application creates a fully authenticated session cookie **after the first authentication factor (password)** is verified — before the second factor (OTP) is completed. The `/login2` page is purely a frontend redirect; the server does not enforce that the OTP has been verified before serving authenticated content.

A correct implementation would use a temporary, restricted session state after step 1 (e.g., `pending_2fa=true`) that only allows access to the `/login2` page — and only upgrades to a full session after the OTP is verified.

---

## 6. Impact

- [x] Account Takeover (of any user whose password is known)
- [x] Authentication Bypass
- [ ] Data Exfiltration (indirect — depends on what the account contains)

**Severity:** High
**Justification:** An attacker with valid credentials (obtained via phishing, credential stuffing, or data breach) can completely bypass 2FA protection and take over any account. The 2FA control — the primary defense against compromised passwords — is rendered useless.

---

## 7. Remediation

1. **Enforce 2FA at the server side:** After the first factor, issue only a restricted/intermediate session token that cannot access any authenticated endpoint except `/login2`. Only upon successful OTP verification should a full session be issued.
2. **Implement server-side state machine for login:** Track login state (`unauthenticated` → `password_verified` → `fully_authenticated`) and enforce it on every request.
3. **Invalidate intermediate tokens on timeout:** If the user does not complete 2FA within a short window (e.g., 5 minutes), invalidate the intermediate session.

---

## 8. Key Takeaways / What I Learned

- **2FA can be broken at the logic level**, not just at the OTP bypass level — always test if you can simply skip the 2FA page after the first factor succeeds.
- **Session management is the backbone of authentication** — understand when and how session tokens are issued.
- **Always follow redirects manually** during testing to check if access controls are enforced at the server or only on the client (redirect).

---

## 9. References

- [PortSwigger Lab: 2FA Simple Bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)
- [OWASP: Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html)
- [OWASP A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
