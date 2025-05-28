# HTB Operation Blackout - Transcoded

## Challenge Overview

**Challenge Name:** Transcoded  
**Category:** Cryptography  
**Difficulty:** easy  
**Flag:** `HTB{B4s364_3nc0d!ng_w1th_cust0m_4lph4b3t_!s_r3v3rs!bl3_t00_a27d47ce33372e498b0d5a277281523b}`

This challenge involves reverse-engineering a custom base64 encoding scheme. The server uses a shuffled alphabet instead of the standard base64 alphabet, requiring recovery of the mapping to decode the flag.

## Given Files

- `server.py` - The server implementation
- `decode.py` - A template solution using pwntools

## Technical Analysis

### Server Implementation Analysis
```python
from base64 import b64encode
from random import shuffle
import string

STANDARD_ENCODING    = list(b'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/')
INTERCEPTED_ENCODING = list((string.digits + string.ascii_letters[:-2] + string.punctuation[8:12]).encode())
shuffle(INTERCEPTED_ENCODING)

def covert_transcode(m: bytes) -> bytes:
    encoded = b64encode(m).strip(b'=')
    return bytes(INTERCEPTED_ENCODING[STANDARD_ENCODING.index(encoded[i])] for i in range(len(encoded)))
```

### Vulnerability Assessment
- **Standard Base64 Alphabet:** `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/`
- **Intercepted Alphabet:** Contains `0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwx)+,-` (shuffled)
- **Encoding Process:** Maps each standard base64 character to corresponding shuffled alphabet character

## Attack Chain

### 1. Server Menu Analysis
```
1. Transcode intercepted training message - Encode our input using custom scheme
2. View archived operative signal - Returns the encoded flag  
3. Disconnect - Exit
```

### 2. Mapping Recovery Strategy
Base64 encoding mechanics enable systematic mapping recovery:
- Each base64 character represents exactly 6 bits
- Input bytes can be crafted to produce specific base64 characters
- The first 32 characters can be mapped using bit manipulation

### 3. Systematic Mapping Recovery
```python
def get_mappings_efficiently():
    """Get mappings efficiently and decode the flag"""
    host, port = "94.237.123.89", 33562
    p = remote(host, port)
    
    mapping = {}
    
    # Phase 1: Get the standard 32 mappings
    print("Getting basic mappings...")
    for j in range(32):
        byte0 = (j << 2) & 0xFF
        if byte0 > 126:
            break

        try:
            p.recvuntil(b'> ')
            p.sendline(b'1')
            p.recvuntil(b':: ')
            p.send(bytes([byte0, 0, 0]) + b'\n')
            
            out = p.recvline(timeout=3).strip()
            if out:
                standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
                mapping[chr(out[0])] = standard[j]
        except:
            break
    
    return mapping
```

**Bit Manipulation Explanation:**
- `(j << 2)` puts the value `j` in the high 6 bits of the first byte
- When base64 encoded, the first character will be `STANDARD[j]`
- The server returns this mapped to the shuffled alphabet

### 4. Gap Filling with Targeted Tests
```python
# Phase 2: Quick targeted tests for remaining characters
quick_tests = [
    '000', '111', '222', '333', '444', '555', '666', '777', '888', '999',
    'AAA', 'BBB', 'CCC', 'DDD', 'EEE', 'FFF', 'GGG', 'HHH', 'III', 'JJJ',
    'aaa', 'bbb', 'ccc', 'ddd', 'eee', 'fff', 'ggg', 'hhh', 'iii', 'jjj',
    '+++', '///', 'A1a', 'Z9z', 'ABC', 'XYZ', 'xyz'
]

for test_str in quick_tests:
    try:
        p.recvuntil(b'> ')
        p.sendline(b'1')
        p.recvuntil(b':: ')
        p.send(test_str.encode() + b'\n')
        
        result = p.recvline(timeout=2).strip()
        if result:
            expected_b64 = base64.b64encode(test_str.encode()).decode().rstrip('=')
            
            for intercept_byte, standard_char in zip(result, expected_b64):
                intercept_char = chr(intercept_byte)
                if intercept_char not in mapping:
                    mapping[intercept_char] = standard_char
    except:
        continue
```

### 5. Intelligent Brute Force for Missing Characters
```python
def smart_decode(flag_encoded, mapping):
    """Decode with intelligent gap filling"""
    decoded_b64 = ""
    missing = set()
    
    for byte_val in flag_encoded:
        char = chr(byte_val)
        if char in mapping:
            decoded_b64 += mapping[char]
        else:
            missing.add(char)
    
    if missing and len(missing) <= 6:
        standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
        used = set(mapping.values())
        unused = [c for c in standard if c not in used]
        
        from itertools import permutations
        missing_list = list(missing)
        
        for perm in permutations(unused[:len(missing)]):
            guess_mapping = dict(zip(missing_list, perm))
            full_mapping = {**mapping, **guess_mapping}
            
            test_b64 = ""
            for byte_val in flag_encoded:
                test_b64 += full_mapping[chr(byte_val)]
            
            while len(test_b64) % 4:
                test_b64 += '='
            
            try:
                decoded = base64.b64decode(test_b64).decode('utf-8')
                
                if (decoded.startswith('HTB{') and decoded.endswith('}') and 
                    len(decoded) > 20 and len(decoded) < 200):
                    readable_chars = sum(1 for c in decoded if c.isprintable() and ord(c) < 128)
                    if readable_chars / len(decoded) > 0.8:
                        print(f"FLAG: {decoded}")
                        return decoded
            except:
                continue
```

### 6. Flag Extraction
```python
# Get flag from server (option 2)
p.recvuntil(b'> ')
p.sendline(b'2')
flag_encoded = p.recvline().strip()

# Apply complete decoding process
flag = smart_decode(flag_encoded, mapping)
```

## Key Technical Concepts

### Base64 Encoding Mechanics
- Each base64 character represents exactly 6 bits (0-63)
- Input bit positions determine which base64 characters appear
- Predictable relationships enable controlled mapping recovery

### Session Consistency Requirements
- Shuffled alphabet generated once per server session
- Mapping recovery and flag extraction must occur in same connection
- Each new connection has different shuffled alphabet

### Cryptographic Analysis Techniques
- Systematic bit manipulation for character space exploration
- Pattern recognition for flag format validation
- Brute force optimization for small search spaces

## Mitigation Recommendations

### Encoding Security
- Use cryptographically secure random number generators for alphabet shuffling
- Implement session timeouts to limit mapping recovery attempts
- Add rate limiting to prevent systematic character space exploration

### Protocol Design
- Avoid exposing encoding/decoding oracles in production systems
- Implement authentication before allowing encoding operations
- Use proper encryption instead of custom encoding schemes

### Input Validation
- Limit input length and character sets for encoding operations
- Implement anomaly detection for systematic mapping attempts
- Log and monitor for suspicious encoding patterns

## Tools Used

- Python with pwntools library for network communication
- Base64 analysis and manipulation utilities
- Itertools for permutation-based brute forcing

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)