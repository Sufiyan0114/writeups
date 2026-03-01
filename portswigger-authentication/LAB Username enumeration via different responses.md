# Lab Writeup: Username Enumeration via Different Responses

**Category:** Authentication
**Difficulty:** Apprentice
**Date Completed:** 27/02/2026
**Platform:** PortSwigger Web Security Academy

---

## 1. Objective

Enumerate a valid username from a list of candidates, then brute-force the password to log into that account.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Username Enumeration via Error Message Difference
**CWE Reference:** CWE-204: Observable Response Discrepancy
**OWASP Category:** A07:2021 – Identification and Authentication Failures

**What is this vulnerability?**
When a login form returns different error messages depending on whether the username or the password is wrong, an attacker can determine which usernames are valid. For example, "Invalid username" vs. "Incorrect password" are two different messages — the second message confirms the username exists. This lets an attacker narrow down enumeration to just valid usernames, making brute-force attacks far more efficient.

---

## 3. Reconnaissance & Discovery

- Attempted login with a random username and random password.
- **Observed response:** `"Invalid username"`
- Attempted login with a known username (`administrator`) and wrong password.
- **Observed response:** `"Incorrect password"`
- **Key Observation:** The error message changes depending on whether the *username* is wrong vs. the *password*. This confirms username enumeration is possible.

**Tool(s) Used:** Burp Suite Community Edition (Intruder)

---

## 4. Exploitation Steps

### Step 1: Establish Baseline Responses
Submit login requests with invalid credentials and observe the error messages.

| Scenario | Error Message |
|---|---|
| Wrong username, wrong password | `Invalid username` |
| Valid username, wrong password | `Incorrect password` |

This difference is the key signal we can exploit.

### Step 2: Enumerate Valid Username using Burp Intruder

Intercepted the login POST request in Burp Suite and sent it to **Intruder**.

**Base Request:**
```
POST /login HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=§test§&password=wrong
```

- **Attack Type:** Sniper
- **Payload Position:** `username` parameter (marked with `§`)
- **Payload List:** PortSwigger's candidate username list

**Result:** When the response no longer contained `"Invalid username"` and instead showed `"Incorrect password"`, that was the valid username.

➡️ **Valid Username Found:** `[identified username]`

### Step 3: Brute-Force Password using Burp Intruder

With the valid username confirmed, sent a new request to Intruder targeting the `password` parameter.

**Base Request:**
```
POST /login HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=[valid-username]&password=§test§
```

- **Payload List:** PortSwigger's candidate password list
- **Signal to watch for:** HTTP 302 redirect (successful login) instead of 200 (failed login)

**Result:** One password returned a `302 Found` response — that was the correct password.

### Step 4: Log In with Discovered Credentials
Used the discovered username and password on the login form.

### Step 5: Lab Solved ✅

---

## 5. Root Cause Analysis

The application leaks information about account existence through its error messages. A secure login form should return the **same, generic error message** regardless of whether the username or password is incorrect — for example: *"Invalid username or password."*

By returning specific errors ("Invalid username" vs. "Incorrect password"), the developer intended to help legitimate users understand what they got wrong — but this user-friendliness inadvertently provides attackers with a oracle to enumerate valid accounts.

---

## 6. Impact

- [x] Account Takeover (via brute-force with enumerated usernames)
- [x] Username Enumeration (privacy breach — confirms which users exist)

**Severity:** High
**Justification:** Username enumeration dramatically reduces the search space for brute-force attacks by confirming which accounts exist. Combined with password spraying or leaked password lists, this leads directly to account takeover. This is especially severe if the enumerated usernames are email addresses (PII disclosure).

---

## 7. Remediation

1. **Use generic error messages:** Return the same message for both invalid username and invalid password: *"Invalid username or password."*
2. **Implement account lockout or rate limiting:** Limit the number of failed login attempts per IP or account to prevent brute-force attacks.
3. **Add CAPTCHA for repeated failures:** After N failed attempts, require CAPTCHA to slow automated attacks.
4. **Ensure consistent response timing:** Even if logic differs, ensure response times are identical for wrong username vs. wrong password (prevent timing-based enumeration).
5. **Monitor for enumeration patterns:** Alert on high volumes of failed login attempts from a single IP.

---

## 8. Key Takeaways / What I Learned

- **Information leakage** is a vulnerability, even if nothing is technically "hacked" — simply confirming that a username exists has real-world impact.
- **Burp Intruder** is the go-to tool for automated parameter fuzzing. Understanding payload positions (Sniper vs. Cluster Bomb) is essential.
- **Response analysis is key:** Learn to compare response length, status codes, and body content to identify successful vs. failed attempts programmatically.
- **Attack chaining:** This lab demonstrates a two-stage attack — enumeration followed by brute-force. Real-world attacks are rarely single-step.

---

## 9. References

- [PortSwigger Lab: Username Enumeration via Different Responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)
- [OWASP: Testing for Account Enumeration](https://owasp.org/www-project-web-security-testing-guide/stable/4-Web_Application_Security_Testing/03-Identity_Management_Testing/04-Testing_for_Account_Enumeration_and_Guessable_User_Account)
- [CWE-204: Observable Response Discrepancy](https://cwe.mitre.org/data/definitions/204.html)
