# Phoenix Sentinel - CTF Write-Up

## Challenge Overview

**Challenge Name:** Phoenix Sentinel  
**Category:** Secure Coding  
**Difficulty:** very easy  
**Flag:** `HTB{l00k0ut_f0r_55rf_cr055_pr0t0c0l_c03ee117c221cf76058f32f9bdfb8ec0}`

**Challenge Description:**
> A patch for SSRF & Flag-Seeding Vulnerability in Incident-Report Service. Volnaya's covert proxy (fetch-and-preview) could be SSRF-weaponized to retrieve internal resources. Meanwhile, the "flag" report was hard-coded to a fake placeholder. This guide shows how to seed the real flag from /flag.txt and remove the localhost-only restriction on incident-reports.

## Vulnerability Analysis

### 1. SSRF (Server-Side Request Forgery) Exposure
The utility in `utils.js` uses an SSRF filter, but reports are only viewable via the localhost-restricted `/incident-reports` endpoint. Attackers controlling `targetUrl` can fetch internal URLs (e.g., `http://localhost:8080/...`).

### 2. Fake Flag Seeding Issue
In `db.js`, the `insertFlagReport()` function always writes:
```javascript
"HTB{FAKE_FLAG_FOR_TESTING}"
```
This occurs even if `/flag.txt` contains the real flag, creating a dead-end for attackers.

### Impact Assessment
- **SSRF Potential:** Attackers can make requests to internal endpoints but only see fake data
- **Flag Accessibility:** Real flag remains unread from `/flag.txt` and never stored in the database
- **Security Bypass:** Localhost restriction prevents normal access to seeded reports

## Exploitation Strategy

### Understanding the Vulnerability Chain

The challenge requires understanding how to:
1. **Seed the Real Flag:** Replace hardcoded fake flag with actual flag from disk
2. **Enable Access:** Remove localhost restriction to allow SSRF exploitation
3. **Chain the Attack:** Use SSRF to access the now-properly seeded flag

### Step 1: Analyze the Vulnerable Code

**Database Flag Seeding (`db.js`):**
```javascript
function insertFlagReport(userId, flagContent) {
    // ... VALUES (?,?,?,?,?,?)",
    [userId,"completed",100,"http://example.com/flag","url","HTB{FAKE_FLAG_FOR_TESTING}"],
    );
}
```

**Route Restriction (`routes/routes.js`):**
```javascript
router.get('/incident-reports', async (req, res) => {
  const clientIp = req.ip;
  if (clientIp !== "127.0.0.1" && clientIp !== "::1") {
    return res.status(403).send("Forbidden");
  }
  // ... rest of endpoint
```

### Step 2: Required Code Modifications

**Fix 1: Read and Seed the Real Flag**

Modify `db.js` to read the actual flag:
```diff
--- a/db.js
+++ b/db.js
@@
- const fs      = require('fs'); 
+ const fs      = require('fs');
+ const FLAG    = fs.readFileSync(path.resolve(__dirname, 'flag.txt'),'utf8').trim();

@@ function seedReports() {
-    db.get("SELECT * FROM users WHERE username = ?", ['admin'], (err, adminRow) => {
+    db.get("SELECT * FROM users WHERE username = ?", ['admin@phoenix.htb'], (err, adminRow) => {
       …
-      if (!adminRow) {
-        db.run("INSERT INTO users …", ['admin@phoenix.htb', crypto.randomBytes(48).toString('base64url')], function(err) {
-          if (!err) insertFlagReport(this.lastID);
-        });
-      } else {
-        insertFlagReport(adminRow.id, flagContent);
-      }
+      if (!adminRow) {
+        db.run(
+          "INSERT INTO users (username,password) VALUES (?,?)",
+          ['admin@phoenix.htb', crypto.randomBytes(48).toString('base64url')],
+          function(err) {
+            if (!err) insertFlagReport(this.lastID, FLAG);
+          }
+        );
+      } else {
+        insertFlagReport(adminRow.id, FLAG);
+      }

@@ function insertFlagReport(userId, flagContent) {
-    … VALUES (?,?,?,?,?,?)",
-    [userId,"completed",100,"http://example.com/flag","url","HTB{FAKE_FLAG_FOR_TESTING}"],
+    … VALUES (?,?,?,?,?,?)",
+    [userId,"completed",100,"http://example.com/flag","url", flagContent],
    );
}
```

**Fix 2: Remove Localhost-Only Restriction**

Modify `routes/routes.js` to allow external access:
```diff
--- a/routes/routes.js
+++ b/routes/routes.js
@@ router.get('/incident-reports', async (req, res) => {
-  const clientIp = req.ip;
-  if (clientIp !== "127.0.0.1" && clientIp !== "::1") {
-    return res.status(403).send("Forbidden");
-  }
+  // IP restriction removed so users can access seeded flag via SSRF
```

### Step 3: Exploitation Methods

After implementing the fixes, the flag can be accessed through:

**Method 1: Direct Access**
```bash
# Navigate directly to the endpoint (if localhost restriction is removed)
curl http://target/challenge/incident-reports
```

**Method 2: SSRF Exploitation**
```bash
# Submit a report with target URL pointing to the internal endpoint
curl -X POST http://target/challenge/submit-report \
  -H "Content-Type: application/json" \
  -d '{"target": "http://localhost:8080/challenge/incident-reports"}'
```

## Attack Chain Summary

```
1. Code Analysis → 2. Identify Flag Seeding Issue → 3. Locate Access Restrictions → 4. Implement Fixes → 5. SSRF to Access Flag → 6. Flag Extraction
```

## Key Vulnerabilities Identified

1. **Hardcoded Test Data:** Fake flag prevents successful exploitation
2. **IP Address Restriction:** Localhost-only access blocks external requests
3. **SSRF Potential:** Internal endpoint accessible via Server-Side Request Forgery
4. **Insecure File Operations:** Reading sensitive files without proper access controls

## Secure Coding Principles Violated

### 1. Hardcoded Secrets
```javascript
// BAD: Hardcoded fake flag
"HTB{FAKE_FLAG_FOR_TESTING}"

// GOOD: Read from secure configuration
const FLAG = process.env.FLAG || fs.readFileSync('/flag.txt', 'utf8').trim();
```

### 2. Inadequate Access Controls
```javascript
// BAD: IP-based restriction only
if (clientIp !== "127.0.0.1") return res.status(403);

// GOOD: Proper authentication and authorization
if (!req.user || !req.user.hasRole('admin')) return res.status(403);
```

### 3. SSRF Vulnerability
```javascript
// BAD: Unrestricted URL fetching
fetch(userProvidedUrl)

// GOOD: URL validation and allowlisting
if (!isAllowedUrl(url)) throw new Error('Invalid URL');
```

## Mitigation Recommendations

### For Development Teams
1. **Environment-Based Configuration:** Use environment variables for sensitive data
2. **Proper Authentication:** Implement robust authentication mechanisms
3. **Input Validation:** Validate and sanitize all user inputs
4. **URL Allowlisting:** Restrict SSRF attack surface through URL validation
5. **Security Testing:** Regular security testing of all endpoints

### For Production Deployment
1. **Network Segmentation:** Isolate internal services from external access
2. **Access Controls:** Implement proper authorization at multiple layers
3. **Monitoring:** Log and monitor all internal service requests
4. **Secret Management:** Use secure secret management systems

### Example Secure Implementation
```javascript
// Secure flag handling
const getFlag = () => {
  if (process.env.NODE_ENV === 'production') {
    return process.env.FLAG;
  }
  return fs.readFileSync('/flag.txt', 'utf8').trim();
};

// Secure access control
const requireAuth = (req, res, next) => {
  if (!req.session.user || req.session.user.role !== 'admin') {
    return res.status(403).json({ error: 'Insufficient privileges' });
  }
  next();
};

// Secure SSRF protection
const isAllowedUrl = (url) => {
  const allowedDomains = ['example.com', 'api.trusted.com'];
  const parsedUrl = new URL(url);
  return allowedDomains.includes(parsedUrl.hostname);
};
```

## Technical Concepts

1. **Server-Side Request Forgery (SSRF):** Vulnerability allowing attackers to make requests from the server
2. **Access Control Bypass:** Circumventing security restrictions through design flaws
3. **Configuration Management:** Proper handling of sensitive configuration data
4. **Secure Development Lifecycle:** Integrating security considerations into development process

## Tools and Techniques Used

- **Static Code Analysis:** Manual review of source code for vulnerabilities
- **Configuration Analysis:** Understanding application configuration and restrictions
- **SSRF Testing:** Techniques for exploiting server-side request forgery
- **Secure Coding Practices:** Implementation of security best practices

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)