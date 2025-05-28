# HTB Operation Blackout - Volnayan Whisperer

## Challenge Overview

**Challenge Name:** Volnayan Whisperer  
**Category:** Hardware / Forensics  
**Difficulty:** easy  
**Flag:** `HTB{d33p_1n_5m5}`

A suspicious employee at a Volnaya mining site was flagged by the intel team. Task Force Phoenix intercepted SMS traffic from the suspect's laptop through a connected modem. Analysis of the captured network traffic revealed covert communications hidden within SMS messages.

## Given Files

- `capture.pcapng` - Network capture containing USB modem communications

## Technical Analysis

### Communication Protocol Analysis
The captured traffic contains USB modem communications using AT commands for SMS transmission. The suspicious activity involves USB bulk transfer packets containing encoded SMS payloads.

### Encoding Investigation
SMS messages sent through modems utilize UCS-2/UTF-16BE encoding, where each Unicode character is represented using 2 bytes (e.g., `00 61` = "a", `00 64` = "d").

## Attack Chain

### 1. Network Traffic Analysis
```bash
# Open capture.pcapng in Wireshark
# Apply display filter to isolate USB modem traffic:
usb or frame contains "AT+CMGS"
```

### 2. USB Bulk Transfer Examination
Located SMS payload within USB bulk transfer packets containing ASCII AT command patterns like `AT+CMGS="..."`. The data segment included the encoded SMS payload.

### 3. Hex Data Extraction
Extracted hex payload from the packet:
```
30 30 36 31 30 30 37 39 30 30 36 43 30 30 36 46 ...
```

This hex data represents Unicode-encoded text in UTF-16BE format.

### 4. Decoding Implementation
```python
hex_data = """
00 61 00 79 00 6C 00 6F 00 61 00 64 00 20 00 69
00 6E 00 74 00 65 00 72 00 63 00 65 00 70 00 74
00 65 00 64 00 2C 00 20 00 48 00 54 00 42 00 7B
00 64 00 33 00 33 00 70 00 5F 00 31 00 6E 00 5F
00 35 00 6D 00 35 00 7D 00 20 00 54 00 68 00 65
00 79 00 27 00 72 00 65 00 20 00 75 00 73 00 69
00 6E 00 67 00 20 00 69 00 74 00 20 00 74 00 6F
00 20 00 62 00 79 00 70 00 61 00 73 00 73 00 20
00 64 00 65 00 74 00 65 00 63 00 74 00 69 00 6F
00 6E 00 2E 00 20 00 4D 00 6F 00 72 00 65 00 20
00 73 00 6F 00 6F 00 6E 00 2E
"""

clean_hex = hex_data.replace("\n", " ").replace(" ", "")
decoded = bytes.fromhex(clean_hex).decode("utf-16-be")
print(decoded)
```

### 5. Message Decryption
**Decoded SMS Message:**
```
payload intercepted, HTB{d33p_1n_5m5} They're using it to bypass detection. More soon.
```

## Key Technical Concepts

### USB Modem Communication
- AT commands used for modem control and SMS transmission
- USB bulk transfer packets contain encoded SMS data
- Wireshark can decode USB modem traffic for analysis

### SMS Encoding Standards
- UCS-2/UTF-16BE encoding common for SMS messaging through modems
- Each Unicode character encoded using 2 bytes
- Big-endian byte ordering in UTF-16BE format

### Covert Communication Analysis
- Legitimate communication channels exploited for espionage
- SMS messages provide plausible deniability for covert operations
- Network traffic analysis reveals hidden intelligence communications

## Mitigation Recommendations

### Network Monitoring
- Implement USB device monitoring on corporate systems
- Deploy network traffic analysis tools to detect unusual modem activity
- Monitor for AT command patterns in network communications

### Communication Security
- Restrict USB modem usage on sensitive systems
- Implement SMS filtering and content inspection
- Use encrypted communication channels for legitimate business communications

### Employee Monitoring
- Deploy behavioral analysis systems to detect suspicious communication patterns
- Implement data loss prevention tools to identify potential exfiltration
- Regular security audits of communication devices and protocols

## Tools Used

- Wireshark for network traffic analysis and USB communication decoding
- Python for hex data processing and UTF-16BE decoding
- Network protocol analyzers for AT command inspection

## Write-Up Credit: [SR20DET](https://ctf.hackthebox.com/user/profile/605510)