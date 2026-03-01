Portswigger lab- Broken access control. 

Lab: Unprotected admin functionality

First i try to access "robots.txt" file and my luck. i accessable.
In this file have administrator admin panel path, now i use this to access admin panel and delete carlos. 

And lab solved.

In this lab i understand, developers should hide admin path or restricted admin panel via public ip, developer should add this specific ip that only allows to admin and only admin can access admin panel instead of anyone.


---
---

# Lab Writeup: Unprotected Admin Functionality
**Category:** Broken Access Control | **Difficulty:** Apprentice | **Date:** 27/02/2026  
**Platform:** PortSwigger Web Security Academy

---

## Objective
Access the admin panel and delete user `carlos` by exploiting an unprotected admin endpoint that is publicly accessible.

---

## Vulnerability
**Broken Access Control — Unprotected Admin Functionality**  
CWE-285: Improper Authorization | OWASP A01:2021 — Broken Access Control

Some developers "hide" admin panels by using obscure URLs, assuming that if users don't know the URL, they can't access it. This is called "security through obscurity" and it's not a real security control. If there's no authentication or IP restriction protecting the admin panel, anyone who discovers the URL gets full admin access. `robots.txt` is a common place where these hidden paths accidentally get disclosed.

---

## How I Found It
First, I explored the application as a normal user — browsing products, checking all the pages, looking for anything interesting.

Then I tried one of the first things I always check: `/robots.txt`. This file is meant to tell search engine crawlers which pages to index or ignore. Developers sometimes add admin paths here with a `Disallow` rule, which unintentionally reveals the path to anyone who checks.

I got lucky — `robots.txt` was accessible, and it listed the admin panel path directly.

**Tool used:** Browser

---

## Exploitation

**Step 1 — Check robots.txt**
```
GET /robots.txt HTTP/2
Host: [lab-id].web-security-academy.net
```

Response:
```
User-agent: *
Disallow: /administrator-panel
```

The admin path `/administrator-panel` was right there.

**Step 2 — Navigate directly to the admin panel**
I browsed to `/administrator-panel`. No login prompt. No access control. The full admin panel loaded.

**Step 3 — Delete Carlos**
I found the "Delete user" button next to `carlos` in the user list and clicked it:

```
GET /administrator-panel/delete?username=carlos HTTP/2
Host: [lab-id].web-security-academy.net
```

**Lab Solved ✅**

---

## Why Does This Happen? (Root Cause)
The developer created an admin panel but only "protected" it by using a non-obvious URL. There's no server-side check verifying that the person accessing `/administrator-panel` is actually an admin. The application trusts that regular users won't find the URL — but `robots.txt` gave it away immediately.

---

## Impact
Any unauthenticated user who discovers the admin URL gets full administrative access to the application — including user management, data deletion, and any other admin functionality. This is a complete privilege escalation.  
**Severity: Critical**

---

## Remediation
1. **Enforce server-side authentication on every admin route:** The server must verify the user's role (admin vs. regular user) on every request to admin endpoints — not just at the login page.
2. **Don't disclose admin paths in robots.txt:** Keep admin paths out of `robots.txt`. If you don't want crawlers there, block it with server-level configuration instead.
3. **IP allowlisting for admin panels:** Restrict access to the admin panel to specific internal IPs or VPN only.
4. **Principle of Least Privilege:** Users should only access what they're authorized for. Admin paths should be inaccessible to everyone by default.

---

## What I Learned
- `robots.txt` is one of the first things to check during recon. Developers often accidentally reveal sensitive paths there.
- "Security through obscurity" is not a security control. A hidden URL’s protection only works until someone finds it.
- Always test whether "restricted" functionality is actually protected at the server level or just hidden from the UI.

---

## References
- [PortSwigger Lab: Unprotected Admin Functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)
- [OWASP: Access Control Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html)
- [CWE-285: Improper Authorization](https://cwe.mitre.org/data/definitions/285.html)
- [OWASP A01:2021 — Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
