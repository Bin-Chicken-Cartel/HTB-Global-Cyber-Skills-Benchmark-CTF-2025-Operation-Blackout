# Terminal - CTF Write-Up

## Challenge Overview

**Challenge Name:** Terminal  
**Category:** Mobile/Reverse Engineering  
**Difficulty:** easy  
**Platform:** HackTheBox

**Challenge Description:**
> An intercepted terminal system believed to be part of Volnaya's covert infrastructure has been acquired. Operatives are tasked with investigating its interface, uncovering hidden functionalities, and extracting critical intelligence without raising alarms.

We are given a single file: `terminal.apk`

---

## Tools Required

Before we begin, ensure you have the following tools installed:

- **APKTool** - For APK extraction and decompilation
- **JADX** or **dex2jar + JD-GUI** - For Java decompilation
- **strings** command - For string extraction
- **Python 3** - For custom decryption scripts
- **Text editor** - For analysis

### Installing Tools (Ubuntu/Debian)

```bash
# Install APKTool
sudo apt update
sudo apt install apktool

# Install JADX (alternative method)
wget https://github.com/skylot/jadx/releases/latest/download/jadx-1.4.7.zip
unzip jadx-1.4.7.zip
sudo mv jadx-1.4.7 /opt/jadx
sudo ln -s /opt/jadx/bin/jadx /usr/local/bin/jadx

# Ensure Python 3 is installed
sudo apt install python3 python3-pip
```

---

## Step 1: Initial APK Analysis

Let's start by examining the APK file to understand its structure and gather basic information.

```bash
# Check file type and basic info
file terminal.apk
ls -la terminal.apk

# Get APK information using aapt (if available)
aapt dump badging terminal.apk 2>/dev/null || echo "aapt not available, proceeding with manual analysis"
```

---

## Step 2: APK Extraction and Decompilation

### Method 1: Using APKTool (Recommended)

```bash
# Create a working directory
mkdir terminal_analysis
cd terminal_analysis

# Extract the APK using apktool
apktool d ../terminal.apk -o extracted_apk

# Examine the extracted structure
ls -la extracted_apk/
```

### Method 2: Manual Extraction (Alternative)

```bash
# APK files are ZIP archives, so we can extract them manually
mkdir manual_extraction
cd manual_extraction
unzip ../terminal.apk
ls -la
```

---

## Step 3: Analyzing the Application Structure

After extraction, we should see a typical Android app structure:

```bash
cd extracted_apk
tree -d -L 2  # Show directory structure

# Key directories to examine:
# - assets/ - Application assets and resources
# - res/ - Android resources
# - smali/ - Converted Dalvik bytecode (if using apktool)
# - AndroidManifest.xml - App permissions and components
```

### Examining the AndroidManifest.xml

```bash
# View the manifest to understand app permissions and components
cat AndroidManifest.xml

# Look for key information:
# - Package name
# - Permissions (especially INTERNET, which indicates network capability)
# - Main activity
# - Any suspicious permissions
```

From the manifest, we can identify:
- **Package Name:** `com.example.terminal`
- **Main Activity:** `com.example.terminal.MainActivity`
- **Key Permissions:** 
  - `android.permission.INTERNET` (can make network connections)
  - `android.permission.DUMP` (can access system dump information)

---

## Step 4: Decompiling Java Code

### Using JADX

```bash
# Decompile the APK to readable Java code
jadx -d decompiled_java terminal.apk

# Navigate to the main application code
cd decompiled_java/sources/com/example/terminal/
ls -la
```

### Examining MainActivity.java

```bash
cat MainActivity.java
```

**Analysis Result:**
```java
package com.example.terminal;

import io.flutter.embedding.android.FlutterActivity;
import kotlin.Metadata;

@Metadata(...)
public final class MainActivity extends FlutterActivity {
}
```

**Key Findings:**
- This is a **Flutter application** (cross-platform framework)
- The main logic is not in Java code but in Flutter/Dart compiled bytecode
- We need to analyze the Flutter assets instead

---

## Step 5: Analyzing Flutter Assets

Flutter applications store their main logic in compiled Dart bytecode. Let's examine the Flutter assets:

```bash
# Navigate to Flutter assets
cd extracted_apk/assets/flutter_assets/
ls -la

# Key files to examine:
# - kernel_blob.bin - Compiled Dart code
# - AssetManifest.json - Asset declarations
# - isolate_snapshot_data - VM snapshot data
```

### Examining Asset Manifest

```bash
cat AssetManifest.json
```

This typically shows only standard Flutter icons, confirming this is a Flutter app.

---

## Step 6: String Analysis of Flutter Bytecode

Since the main application logic is in the compiled Dart bytecode (`kernel_blob.bin`), we need to extract readable strings from it:

```bash
# Extract all strings from the kernel blob
strings kernel_blob.bin > extracted_strings.txt

# Look for interesting strings
grep -i "terminal" extracted_strings.txt
grep -i "command" extracted_strings.txt
grep -i "password" extracted_strings.txt
grep -i "secret" extracted_strings.txt
```

### Finding Application-Specific Strings

```bash
# Search for package-specific strings
grep "package:terminal" extracted_strings.txt

# Look for main application files
grep "lib/main.dart" extracted_strings.txt -A 10 -B 5
```

**Key Discovery:**
We find references to the application structure:
```
package:terminal/commands.dart
package:terminal/terminal_controller.dart
package:terminal/terminal_view.dart
```

---

## Step 7: Extracting Application Logic

Let's search for the core application functionality:

```bash
# Search for command-related functionality
grep -A 20 -B 5 "helpCommand" extracted_strings.txt

# Look for hidden commands
grep -E "(c2|secret|hidden)" extracted_strings.txt -i

# Search for authentication mechanisms
grep -A 10 -B 5 "password" extracted_strings.txt
```

**Critical Discovery:**
```bash
# Look for the C2 mode functionality
grep -A 15 -B 5 "c2-mode" extracted_strings.txt
```

**Output reveals:**
```
c2-mode: password required to activate.
C2 mode activated. Use 'get-secret' to retrieve information.
c2-mode: incorrect password.
```

---

## Step 8: Finding the C2 Password

Continue analyzing the strings to find the hardcoded password:

```bash
# Search for password assignment
grep -A 5 -B 5 "c2Password" extracted_strings.txt
```

**Critical Finding:**
```
String c2Password = "w0w_y0u_f0und_m3";
```

**Discovery Summary:**
- **C2 Password:** `w0w_y0u_f0und_m3`
- **Hidden Command:** `c2-mode` (to activate)
- **Secret Command:** `get-secret` (available after C2 activation)

---

## Step 9: Analyzing the Encryption Mechanism

Let's find the encrypted payload:

```bash
# Search for base64 encoded data (long strings with padding)
grep -E "[A-Za-z0-9+/]{40,}=" extracted_strings.txt

# Look for encryption-related code
grep -A 10 -B 10 "base64Secret" extracted_strings.txt
```

**Critical Discovery:**
```bash
# Found in the get-secret command implementation
const String base64Secret = '3+3haP80SM7i/ijRRRlwt09zsPzc1o8tMQezph0Gm63nfkKl9GA=';
```

### Understanding the Decryption Process

From the string analysis, we can reconstruct the decryption logic:

```bash
# Search for encryption details
grep -A 15 -B 5 "RC4" extracted_strings.txt
grep -A 10 -B 5 "currentPath" extracted_strings.txt
```

**Algorithm Discovery:**
- **Encryption:** RC4 stream cipher
- **Key:** Current path (UTF-8 encoded)
- **Payload:** Base64 encoded encrypted data

---

## Step 10: Creating the Decryption Script

Based on our analysis, let's create a Python script to decrypt the payload:

```python
#!/usr/bin/env python3
import base64

def rc4(key, data):
    """RC4 encryption/decryption algorithm"""
    # Key scheduling
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]
    
    # Pseudo-random generation algorithm
    i = j = 0
    result = []
    for byte in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        k = S[(S[i] + S[j]) % 256]
        result.append(byte ^ k)
    
    return bytes(result)

# The encrypted payload from the app
base64_secret = "3+3haP80SM7i/ijRRRlwt09zsPzc1o8tMQezph0Gm63nfkKl9GA="

# Decode base64
encrypted_bytes = base64.b64decode(base64_secret)
print(f"Encrypted bytes length: {len(encrypted_bytes)}")

# Try different possible paths as keys
possible_paths = ["/", "/home", "/home/secret", "/documents", "/root"]

print("--- Trying different paths as keys ---")
for path in possible_paths:
    key = path.encode('utf-8')
    decrypted = rc4(key, encrypted_bytes)
    try:
        message = decrypted.decode('utf-8')
        print(f"Path '{path}': {message}")
    except:
        print(f"Path '{path}': [Cannot decode as UTF-8]")
```

Save this as `decrypt_payload.py` and run it:

```bash
python3 decrypt_payload.py
```

---

## Step 11: Finding the Correct Decryption Key

From our analysis of the application logic, we need to understand when the `get-secret` command is executed. Let's examine the filesystem structure:

```bash
# Look for filesystem references in the strings
grep -A 10 -B 5 "fileSystem" extracted_strings.txt
grep -A 5 "/home/secret" extracted_strings.txt
```

**Discovery:**
The application has a simulated filesystem with a `/home/secret` directory. When navigating to this directory and executing `get-secret`, the current path becomes the decryption key.

**Running the decryption script reveals:**
```
Path '/home/secret': HTB{arent-you-hacker-wannabe_3fe48bcc}
```

---

## Solution Summary

### Complete Attack Chain:

1. **APK Decompilation:** Extract and analyze the Flutter application
2. **String Analysis:** Extract readable strings from compiled Dart bytecode
3. **Logic Reconstruction:** Understand the hidden C2 functionality
4. **Credential Discovery:** Find the hardcoded C2 password: `w0w_y0u_f0und_m3`
5. **Encryption Analysis:** Identify RC4 encryption with path-based key
6. **Key Discovery:** Determine that `/home/secret` is the correct decryption key
7. **Flag Extraction:** Decrypt the payload to obtain the flag

### Application Behavior:

To obtain the flag through normal application usage:
1. Navigate to `/home/secret` using: `cd /home/secret`
2. Activate C2 mode using: `c2-mode w0w_y0u_f0und_m3`
3. Execute secret command: `get-secret`
4. The app decrypts the payload using current path (`/home/secret`) as RC4 key

### Final Flag:
```
HTB{arent-you-hacker-wannabe_3fe48bcc}
```

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)