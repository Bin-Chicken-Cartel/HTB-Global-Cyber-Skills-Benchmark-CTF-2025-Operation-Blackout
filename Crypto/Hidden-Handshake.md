# Hidden Handshake - CTF Write-Up

## Challenge Overview

**Challenge Name:** Hidden Handshake  
**Category:** Crypto  
**Difficulty:** very easy  
**Flag:** `HTB{v3ry_n1c3!___4_b1t_sp1cy_k3ystr34m_r3us3_cb3fa823d7bd2e40e03e41c35709ae76}`

**Challenge Description:**
> Amidst the static hum of Volnaya's encrypted comms, Task Force Phoenix detects a subtle, silent handshake—a fleeting, ghostly link hidden beneath layers of noise. Your objective: capture, decode, and neutralize this quiet whisper before it escalates into a deafening roar that plunges nations into chaos.

## Understanding the Server

The challenge provides a Python server that we can connect to. Key functionalities:

- **Secure Access Key & Codename:** The server asks for an 8-character "secure access key" (`pass2`) and an "Agent Codename" (`user`)
- **Encryption:** It encrypts a secret message containing the flag with format:
  `"Agent {user}, your clearance for Operation Blackout is: {FLAG}. It is mandatory that you keep this information confidential."`
- **Encryption Method:** AES (Advanced Encryption Standard) in CTR (Counter) mode
- **Key Derivation:** The AES key is derived using `kdf(pass1, pass2)` which computes `hashlib.sha256(pass1 + pass2).digest()`
  - `pass1` is a `server_secret` (unknown 8-character password)
  - `pass2` is our 8-character "secure access key"
- **Critical Flaw:** The `nonce` for AES-CTR encryption is directly set to our `pass2`

## Vulnerability Analysis

### The Critical Flaw: Nonce Reuse in AES-CTR

AES-CTR mode works as a stream cipher:
- `Ciphertext = Plaintext XOR Keystream`
- `Keystream = AES_Encrypt(Key, Nonce + Counter)`

**Why Nonce Uniqueness Matters:**
For a given key, the nonce **must be unique** for every message. If you use the same key and nonce for two different plaintexts, you generate the same keystream.

**Consequences of Nonce Reuse:**
- `C1 = P1 XOR Keystream` (First message)
- `C2 = P2 XOR Keystream` (Second message, same key/nonce)
- `C1 XOR C2 = P1 XOR P2` (Keystreams cancel out)

If we know one plaintext (P2), we can recover the other:
- `P1 = (C1 XOR C2) XOR P2`

**How This Applies:**
1. The `server_secret` (`pass1`) is fixed but unknown
2. If we provide the same `pass2` twice, the AES key remains the same
3. Since `pass2` is also used as the nonce, we force nonce reuse

## Exploitation Strategy

### Step 1: Get C1 (Ciphertext with Flag + Minimal User Info)

Connect to the server and:
- Provide a fixed `pass_key` (e.g., `"00000000"`)
- Provide a very short `user` string (e.g., `""`)
- Server encrypts: `P1 = "Agent , your clearance for Operation Blackout is: {FLAG}. It is mandatory..."`
- Receive `C1`

### Step 2: Get C2 (Ciphertext with Known Plaintext)

In a second interaction:
- Provide the **exact same** `pass_key` (ensures same key and nonce)
- Provide a long, known `user` string (e.g., 200 'A's)
- Server encrypts: `P2 = "Agent AAAA..., your clearance for Operation Blackout is: {FLAG}. It is mandatory..."`
- Receive `C2`

The key insight: we know most of `P2` because we control the user string.

### Step 3: Recover P1 and Extract Flag

1. **XOR Ciphertexts:** Calculate `XOR_Result = C1 XOR C2 = P1 XOR P2`
2. **Recover P1:** Since we know most of `P2`, we can calculate `P1 = XOR_Result XOR P2_known`
3. **Extract Flag:** The recovered `P1` contains the flag between the known prefix and suffix

## Complete Exploit Implementation

```python
#!/usr/bin/env python3
import socket

HOST = "127.0.0.1"  # Change to target host
PORT = 1337         # Change to target port

KNOWN_PREFIX_AGENT = b"Agent "
KNOWN_PREFIX_CLEARANCE = b", your clearance for Operation Blackout is: "
KNOWN_SUFFIX = b". It is mandatory that you keep this information confidential."

def xor_bytes(b1, b2):
    return bytes(a ^ b for a, b in zip(b1, b2))

def recv_until(f_socket, delimiter_bytes):
    buffer = b""
    while delimiter_bytes not in buffer:
        data = f_socket.read(1)
        if not data:
            break
        buffer += data
    return buffer

def interact_with_server(f_socket, pass_key_str, user_val_str):
    # Wait for "secure access key" prompt
    recv_until(f_socket, b"Enter your secure access key: ")
    f_socket.write(pass_key_str.encode() + b"\n")
    f_socket.flush()
    
    # Wait for "Agent Codename" prompt
    recv_until(f_socket, b"Enter your Agent Codename: ")
    f_socket.write(user_val_str.encode() + b"\n")
    f_socket.flush()
    
    # Read response to find encrypted transmission
    while True:
        line = f_socket.readline()
        if b"Encrypted transmission:" in line:
            return line.split(b": ")[1].strip().decode()
        if not line:
            break
    return None

def solve():
    pass_key = "00000000"
    user_1 = ""  # For P1 (short user, flag visible)
    user_2 = "A" * 200  # For P2 (long known user)
    
    # Construct known prefixes
    p1_known_prefix_bytes = KNOWN_PREFIX_AGENT + user_1.encode() + KNOWN_PREFIX_CLEARANCE
    p2_known_prefix_bytes = KNOWN_PREFIX_AGENT + user_2.encode() + KNOWN_PREFIX_CLEARANCE
    
    # Connect to server
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((HOST, PORT))
        f_socket = sock.makefile('rwb', buffering=1)
        
        # Read banner (3 lines)
        for _ in range(3):
            f_socket.readline()
        
        # First interaction - get C1
        c1_hex = interact_with_server(f_socket, pass_key, user_1)
        c1_bytes = bytes.fromhex(c1_hex)
        
        # Second interaction - get C2
        c2_hex = interact_with_server(f_socket, pass_key, user_2)
        c2_bytes = bytes.fromhex(c2_hex)
        
        f_socket.close()
    
    # XOR the ciphertexts: C1 XOR C2 = P1 XOR P2
    xor_c_bytes = xor_bytes(c1_bytes, c2_bytes)
    
    # Recover the flag portion of P1
    offset = len(p1_known_prefix_bytes)
    max_len = min(len(xor_c_bytes), len(p2_known_prefix_bytes))
    
    recovered_bytes = bytearray()
    for idx_in_arrays in range(offset, max_len):
        # P1[i] = (C1[i] XOR C2[i]) XOR P2[i]
        p1_byte = xor_c_bytes[idx_in_arrays] ^ p2_known_prefix_bytes[idx_in_arrays]
        recovered_bytes.append(p1_byte)
    
    # Find the known suffix to extract just the flag
    known_suffix_bytes = KNOWN_SUFFIX
    suffix_pos = recovered_bytes.find(known_suffix_bytes)
    
    if suffix_pos != -1:
        flag_bytes = recovered_bytes[:suffix_pos]
        flag = flag_bytes.decode()
        print(f"Flag: HTB{{{flag}}}")
    else:
        print("Could not find the expected suffix in recovered data")
        print(f"Recovered: {recovered_bytes}")

if __name__ == "__main__":
    solve()
```

## Attack Chain Summary

```
1. Connect with Fixed Pass + Short User → 2. Get C1 → 3. Connect with Same Pass + Long Known User → 4. Get C2 → 5. XOR Ciphertexts → 6. Recover P1 → 7. Extract Flag
```

## Mathematical Explanation

Given:
- `C1 = P1 ⊕ Keystream`
- `C2 = P2 ⊕ Keystream` (same keystream due to nonce reuse)

Therefore:
- `C1 ⊕ C2 = (P1 ⊕ Keystream) ⊕ (P2 ⊕ Keystream) = P1 ⊕ P2`

Since we know most of P2 (our controlled input), we can recover P1:
- `P1 = (C1 ⊕ C2) ⊕ P2_known`

## Key Cryptographic Concepts

1. **AES-CTR Mode:** Stream cipher that requires unique nonces
2. **Nonce Reuse Attack:** Classic vulnerability in stream ciphers
3. **Known Plaintext Attack:** Using partial knowledge to recover secrets
4. **XOR Properties:** `A ⊕ A = 0` and `A ⊕ 0 = A`

## Mitigation Recommendations

### For Cryptographic Implementation
1. **Unique Nonces:** Always use cryptographically secure random nonces
2. **Nonce Tracking:** Implement mechanisms to prevent nonce reuse
3. **Key Rotation:** Regularly rotate encryption keys
4. **Mode Selection:** Consider authenticated encryption modes like GCM

### For Secure Communication
1. **Protocol Design:** Don't reuse user input as cryptographic parameters
2. **Input Validation:** Validate all cryptographic inputs
3. **Key Management:** Implement proper key derivation and storage
4. **Security Testing:** Test for common cryptographic vulnerabilities

## Tools and Techniques Used

- **Python Socket Programming:** Direct TCP communication with server
- **Cryptographic Analysis:** Understanding AES-CTR mode weaknesses
- **XOR Operations:** Byte-level manipulation for ciphertext analysis
- **Known Plaintext Attack:** Leveraging controlled input for decryption

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)