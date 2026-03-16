# Lab Writeup: Excessive Data Exposure in Community Posts API

**Category:** Excessive Data Exposure / Information Disclosure  
**Difficulty:** Practitioner  
**Date Completed:** 10/03/2026  
**Platform:** crAPI (Completely Ridiculous API)  
**Lab URL:** http://localhost:8888

---

## 1. Objective

Identify and exploit an API endpoint that returns excessive sensitive information in its response, exposing user data that should not be accessible to regular users, demonstrating how poor API design can lead to mass data leakage.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Excessive Data Exposure (API3:2023)

**CWE Reference:** CWE-359: Exposure of Private Personal Information to an Unauthorized Actor

**OWASP Category:** API3:2023 – Excessive Data Exposure

**What is this vulnerability?**

Excessive Data Exposure happens when an API returns more data than necessary in its responses. Developers often rely on client-side filtering to display only relevant information, but this means sensitive data is still transmitted over the network. In this case, the community posts endpoint was returning complete user objects including email addresses, internal IDs, and other sensitive fields that a regular user shouldn't see. This is a classic case of "return everything, filter on the frontend" which is a dangerous practice.

---

## 3. Reconnaissance & Discovery

When I started exploring the crAPI application, I followed a systematic approach to understand how data flows through the system:

**Initial Application Mapping:**
- Registered an account and logged in to explore available features
- Discovered a "Community" section where users can create and view posts
- Noticed that posts displayed usernames but I wondered what else might be in the API response
- Decided to intercept the traffic using Burp Suite to see the raw API responses

**Traffic Interception Setup:**
- Configured browser to proxy through Burp Suite (localhost:8080)
- Navigated to the community posts section
- Started monitoring all API calls in Burp's HTTP history

**The Discovery Moment:**
While browsing through community posts, I intercepted a GET request to `/community/api/v2/community/posts/recent?limit=10`. The response looked normal in the browser, but when I examined the raw JSON in Burp Suite, I was shocked to see complete user profiles embedded in each post object.

**What I Expected vs What I Got:**
- Expected: Post ID, title, content, author name
- Got: Everything above PLUS email addresses, vehicle IDs, profile picture URLs, internal database IDs, creation timestamps, and more

This was a goldmine of information that shouldn't be exposed to regular users.

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser

---

## 4. Exploitation Steps

### Step 1: Identify the Target Endpoint

After logging into crAPI, I navigated to the Community section and clicked on "Recent Posts". I kept Burp Suite's Proxy intercept on to capture the request.

**Intercepted Request:**
```http
GET /community/api/v2/community/posts/recent?limit=10 HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/community
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYVZ1WTk5dTlvdGlhV0ZPSWpveE5jcyOTYyMzE...
Connection: keep-alive
Cookie: chat_session_id=61355b29-c203-441c-8ba0-0e4d6bb62bb0
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
```

**Key Observation:** This is a simple GET request with authentication. Nothing unusual in the request itself.

### Step 2: Analyze the Response

I forwarded the request and examined the response in Burp Suite's Response tab. What I found was concerning:

**Response (Partial):**
```http
HTTP/1.1 200 OK
Server: openresty/1.27.1.2
Date: Sun, 09 Mar 2026 09:39:23 GMT
Content-Type: application/json
Connection: keep-alive
Access-Control-Allow-Headers: Accept, Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization
Access-Control-Allow-Methods: POST, GET, OPTIONS, PUT, DELETE
Access-Control-Allow-Origin: *
Content-Length: 1003

{
  "posts": [
    {
      "id": "TxzKhiQ5MLmfameoxdycd5",
      "title": "Title 1",
      "content": "Hello world 1",
      "author": {
        "nickname": "Robot",
        "email": "robot001@example.com",
        "vehicleId": "4bae0860-ec77-4de1-a3a0-ba1bcab5e5e5",
        "profile_pic_url": "",
        "created_at": "2026-03-08T07:35:37.6312"
      },
      "comments": [],
      "authorid": 3,
      "CreatedAt": "2026-03-08T07:35:37.6312"
    },
    {
      "id": "JeaUydhyTAmBLqoZ7CEsiH",
      "title": "Title 2",
      "content": "Hello world 2",
      "author": {
        "nickname": "Pcdo",
        "email": "pcdo@a00@example.com",
```

### Step 3: Document the Exposed Data

I carefully documented all the sensitive fields being exposed:

**Exposed Sensitive Information:**
1. **Email Addresses** - `robot001@example.com`, `pcdo@a00@example.com`
2. **Vehicle IDs** - `4bae0860-ec77-4de1-a3a0-ba1bcab5e5e5` (UUID format)
3. **Internal Author IDs** - `authorid: 3` (database primary keys)
4. **Account Creation Timestamps** - `created_at: "2026-03-08T07:35:37.6312"`
5. **Profile Picture URLs** - Could reveal storage structure
6. **User Nicknames** - Combined with emails, perfect for social engineering

### Step 4: Test with Multiple Requests

To confirm this wasn't a one-time glitch, I tested multiple scenarios:

**Test 1: Different Limit Parameters**
```http
GET /community/api/v2/community/posts/recent?limit=50 HTTP/1.1
```
Result: All 50 posts returned with complete user data

**Test 2: Unauthenticated Request**
Removed the Authorization header and resent the request.
Result: Still received full user data (even worse!)

**Test 3: Different User Account**
Logged in with my attacker account and made the same request.
Result: Could see email addresses and details of all other users

### Step 5: Assess the Impact

With this information, I could:
- Build a complete database of all crAPI users
- Extract email addresses for phishing campaigns
- Map vehicle IDs to user accounts
- Identify account creation patterns
- Correlate usernames with email addresses for credential stuffing attacks

### Step 6: Verify What Should Be Returned

I checked what the frontend actually displays:
- Only shows: Nickname, post title, post content
- Doesn't show: Email, vehicle ID, internal IDs, timestamps

This confirmed that developers were relying on client-side filtering, a major security flaw.

---

## 5. Root Cause Analysis

The vulnerability exists because of poor API design and a fundamental misunderstanding of the security model:

**What the Developers Did Wrong:**

1. **Returned Complete Objects:** The API endpoint returns full user objects from the database without filtering sensitive fields. This is likely because they used an ORM (Object-Relational Mapping) tool and just serialized the entire user model.

2. **Trusted the Client:** Developers assumed that because the frontend only displays certain fields, users wouldn't see the rest. This is a dangerous assumption since anyone can intercept HTTP traffic.

3. **No Data Transfer Objects (DTOs):** There's no layer between the database model and the API response. A proper DTO would only include fields meant for public consumption.

4. **Lack of Field-Level Access Control:** The API doesn't check what fields the requesting user should be allowed to see.

**What Should Have Been Done:**

```javascript
// BAD - Current Implementation
app.get('/community/posts/recent', async (req, res) => {
  const posts = await Post.findAll({
    include: [User], // Includes ALL user fields
    limit: req.query.limit
  });
  res.json({ posts }); // Returns everything
});

// GOOD - Secure Implementation
app.get('/community/posts/recent', async (req, res) => {
  const posts = await Post.findAll({
    attributes: ['id', 'title', 'content', 'createdAt'],
    include: [{
      model: User,
      attributes: ['nickname'] // Only public fields
    }],
    limit: req.query.limit
  });
  res.json({ posts });
});
```

The root cause is a "return everything, filter on frontend" mentality which violates the principle of least privilege.

---

## 6. Impact

If this were a production application, an attacker could:

- [x] Mass Data Harvesting
- [x] Privacy Violation (GDPR/CCPA violations)
- [x] Social Engineering Attacks
- [x] Credential Stuffing Preparation
- [x] User Profiling and Tracking
- [ ] Direct Account Takeover (requires additional steps)

**Severity:** High

**Justification:** This vulnerability allows any authenticated user (or even unauthenticated in some cases) to harvest personal information of all users in the system. Email addresses alone are valuable for phishing attacks, and when combined with usernames and vehicle information, attackers can build detailed profiles. This represents a massive privacy breach and could result in regulatory fines under GDPR (up to 4% of annual revenue) or CCPA. The attack requires minimal skill and can be automated to scrape thousands of user records in minutes.

**Real-World Impact:**
- LinkedIn data scraping incidents
- Facebook's Cambridge Analytica scandal (excessive data sharing)
- Twitter API data exposure (2022)

---

## 7. Remediation

Here's how developers should fix this critical vulnerability:

1. **Implement Data Transfer Objects (DTOs):**
   Create specific response models that only include necessary fields
   
   ```javascript
   class PostResponseDTO {
     constructor(post) {
       this.id = post.id;
       this.title = post.title;
       this.content = post.content;
       this.author = {
         nickname: post.author.nickname
         // Only include public fields
       };
       this.createdAt = post.createdAt;
     }
   }
   
   // In the API endpoint
   const posts = await Post.findAll({ include: [User] });
   const sanitizedPosts = posts.map(p => new PostResponseDTO(p));
   res.json({ posts: sanitizedPosts });
   ```

2. **Use ORM Field Selection:**
   Explicitly specify which fields to return at the database query level
   
   ```python
   # Python/Django example
   posts = Post.objects.select_related('author').only(
       'id', 'title', 'content', 'created_at',
       'author__nickname'  # Only nickname from author
   )
   ```

3. **Implement API Response Schemas:**
   Use tools like JSON Schema or OpenAPI to define and validate response structures
   
   ```yaml
   # OpenAPI specification
   PostResponse:
     type: object
     properties:
       id:
         type: string
       title:
         type: string
       author:
         type: object
         properties:
           nickname:
             type: string
         # Email explicitly NOT included
   ```

4. **Add Response Filtering Middleware:**
   Create a middleware layer that strips sensitive fields before sending responses
   
   ```javascript
   const sensitiveFields = ['email', 'password', 'ssn', 'vehicleId'];
   
   app.use((req, res, next) => {
     const originalJson = res.json;
     res.json = function(data) {
       const filtered = removeSensitiveFields(data, sensitiveFields);
       originalJson.call(this, filtered);
     };
     next();
   });
   ```

5. **Conduct API Security Reviews:**
   - Review all API endpoints for excessive data exposure
   - Use automated tools like OWASP ZAP or custom scripts
   - Implement API security testing in CI/CD pipeline

6. **Apply Principle of Least Privilege:**
   - Only return data that the client explicitly needs
   - Different endpoints for different access levels
   - Admin endpoints should be separate from user endpoints

7. **Monitor and Log Data Access:**
   - Log which fields are accessed in API responses
   - Set up alerts for unusual data access patterns
   - Regular audits of API response payloads

---

## 8. Key Takeaways / What I Learned

- **Never Trust the Client:** Just because the frontend doesn't display certain data doesn't mean users can't access it. Always filter on the backend.

- **API Responses Are Public:** Treat every API response as if it will be intercepted and analyzed. Only include data that you're comfortable making public.

- **ORMs Can Be Dangerous:** Object-Relational Mapping tools make it easy to accidentally expose entire database models. Always use explicit field selection or DTOs.

- **Think Like an Attacker:** During development, ask yourself "What if someone intercepts this response?" This mindset helps catch these issues early.

- **Excessive Data Exposure is Common:** This is the #3 vulnerability in OWASP API Security Top 10 because it's so prevalent. Many developers don't realize they're doing it.

- **Privacy Regulations Matter:** With GDPR, CCPA, and other privacy laws, exposing user data can result in massive fines. Security isn't just about preventing hacks; it's about protecting user privacy.

- **Burp Suite is Essential:** For API testing, tools like Burp Suite are invaluable for seeing what's really being transmitted, not just what the UI shows.

---

## 9. References

- [OWASP API Security Top 10 - API3:2023 Excessive Data Exposure](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
- [CWE-359: Exposure of Private Personal Information](https://cwe.mitre.org/data/definitions/359.html)
- [crAPI - Completely Ridiculous API](https://github.com/OWASP/crAPI)
- [GDPR Article 5: Principles relating to processing of personal data](https://gdpr-info.eu/art-5-gdpr/)
- [Burp Suite Documentation](https://portswigger.net/burp/documentation)

---

**Image Reference:**

![Excessive Data Exposure - Community Posts API](images/api-testing-2-1510.png)
*Figure 1: Burp Suite showing the API response with exposed user email addresses, vehicle IDs, and other sensitive information that should not be accessible*

