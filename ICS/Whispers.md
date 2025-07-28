# HVAC Data Exfiltration - CTF Write-Up

## Challenge Overview

**Challenge Name:** Whispers HVAC Data Exfiltration  
**Category:** ICS / Network Forensics / Steganography  
**Difficulty:** very easy  
**Files Provided:** `hvac.pcap`  
**Flag:** `HTB{Unlock3d_th3_v3nt_0f_d4t4_15rupt1}`

**Challenge Description:**
> Analyze network traffic from an industrial HVAC (Heating, Ventilation, and Air Conditioning) system. Something suspicious appears to be happening within what looks like normal system telemetry data.

## Initial Reconnaissance

### Challenge Analysis
We're given a PCAP file containing network traffic from an industrial HVAC system. Industrial control systems like HVAC often use specialized protocols to communicate sensor data, system status, and control commands.

The challenge suggests that sensitive data might be hidden within legitimate HVAC communications - a classic case of **steganography in industrial networks**.

### File Structure Analysis
```bash
file hvac.pcap
# Expected: tcpdump capture file or similar
```

### Tools Required
- **Wireshark:** Primary network protocol analysis
- **Python:** Hex data processing and decoding
- **Text Analysis:** Pattern recognition and extraction

## Vulnerability Analysis

### Industrial Protocol Steganography
The attack vector involves:
1. **Data Hiding:** Sensitive information concealed within legitimate HVAC telemetry
2. **Protocol Abuse:** Using non-standard fields to embed encoded data
3. **Fragmented Exfiltration:** Data split across multiple packets to avoid detection
4. **Industrial Camouflage:** Malicious traffic disguised as normal system operations

### HVAC Communication Pattern
Normal HVAC systems communicate various parameters:
- **Temperature Control:** Sensor readings and setpoints
- **HVAC Components:** Fan speeds, cooling percentages, heating status
- **System Monitoring:** Compressor status, pressure readings, error codes

## Exploitation Strategy

### Step 1: PCAP Analysis with Wireshark

Open the PCAP file and examine the traffic patterns:

```bash
# Open in Wireshark
wireshark hvac.pcap
```

**Initial Observations:**
- Multiple packets containing HVAC system telemetry data
- Temperature readings, fan speeds, and system status information
- Mixed binary and ASCII data in packet payloads

### Step 2: Legitimate HVAC Data Identification

Normal HVAC telemetry data found in the traffic:

```
TEMP:20C
TEMP:25C  
TEMP:40C
SETPOINT:22C
MAX:30C
MIN:15C
FAN:50%
COOL:75%
HEAT:OFF
COMP:ON
PRES:5bar
VENT:RECIRC
AIR:1000m3/h
STATE:RUNNING
ERR:0
```

This represents legitimate industrial control system data:
- **Temperature Sensors:** Current readings and setpoints
- **HVAC Components:** Fan speeds, cooling percentages, heating status
- **System Monitoring:** Compressor status, pressure readings, error codes

### Step 3: Suspicious Data Discovery

While examining packet payloads, unusual patterns emerge mixed with legitimate data:

**Suspicious "KEY:" entries:**
```
KEY:4854427b556e6c
KEY:30636b316e675f  
KEY:7468335f76336e
KEY:74735f30665f64
KEY:31357275707431
```

**Anomaly Indicators:**
1. **Non-standard Protocol:** Real HVAC systems don't use "KEY:" fields
2. **Hex Format:** Values are clearly hexadecimal encoded
3. **Consistent Pattern:** Multiple similar entries suggest intentional exfiltration

### Step 4: Wireshark Filtering for Suspicious Traffic

Use filters to isolate the malicious data:

```
tcp contains "KEY:" or udp contains "KEY:"
```

This helps focus on packets containing hidden data while filtering out normal HVAC telemetry.

### Step 5: Hex Data Extraction and Analysis

**Extracted hex strings:**
1. `4854427b556e6c`
2. `30636b316e675f`
3. `7468335f76336e` 
4. `74735f30665f64`
5. `31357275707431`

### Step 6: Hex to ASCII Conversion

Create a Python script to decode the hex data:

```python
#!/usr/bin/env python3

# Extracted hex strings from HVAC traffic
hex_strings = [
    "4854427b556e6c",
    "30636b316e675f", 
    "7468335f76336e",
    "74735f30665f64",
    "31357275707431"
]

def decode_hex_strings(hex_list):
    decoded_parts = []
    for hex_str in hex_list:
        try:
            ascii_str = bytes.fromhex(hex_str).decode('ascii')
            decoded_parts.append(ascii_str)
            print(f"{hex_str} -> {ascii_str}")
        except ValueError as e:
            print(f"Failed to decode {hex_str}: {e}")
    
    return decoded_parts

# Decode the strings
decoded_parts = decode_hex_strings(hex_strings)

# Reconstruct the complete message
complete_message = ''.join(decoded_parts)
print(f"\nReconstructed message: {complete_message}")
```

**Decoding Results:**
```
4854427b556e6c -> HTB{Unl
30636b316e675f -> ock3d_th
7468335f76336e -> e_v3nt_
74735f30665f64 -> 0f_d4t4
31357275707431 -> _15rupt1
```

### Step 7: Flag Reconstruction

Concatenating all decoded parts in order:
```
HTB{Unl + ock3d_th + e_v3nt_ + 0f_d4t4 + _15rupt1
```

**Complete Flag:** `HTB{Unlock3d_th3_v3nt_0f_d4t4_15rupt1}`

## Attack Chain Summary

```
1. PCAP Analysis → 2. HVAC Traffic Identification → 3. Anomaly Detection → 4. Hex String Extraction → 5. ASCII Decoding → 6. Flag Reconstruction
```

## Key Technical Concepts

1. **Industrial Protocol Analysis:** Understanding HVAC/SCADA system communications
2. **Network Steganography:** Hiding data within legitimate protocol traffic
3. **Hex Encoding:** Using hexadecimal encoding for data obfuscation
4. **PCAP Forensics:** Network packet analysis and filtering techniques
5. **Data Exfiltration:** Covert channels in industrial control systems

## Security Implications

### IoT/ICS Vulnerabilities
This attack demonstrates several critical issues:
- **Compromised Industrial Equipment:** HVAC systems used for data exfiltration
- **Protocol Abuse:** Legitimate protocols subverted for malicious purposes
- **Detection Evasion:** Malicious traffic hidden within normal operations
- **Supply Chain Risks:** Industrial equipment potentially compromised at manufacturing

### Real-World Impact
- **Data Theft:** Sensitive information exfiltrated through industrial systems
- **Operational Disruption:** Potential for system manipulation beyond data theft
- **Network Compromise:** Industrial networks used as attack vectors
- **Critical Infrastructure Risk:** HVAC systems control building environments

## Mitigation Recommendations

### For Industrial Networks
1. **Deep Packet Inspection:** Monitor for non-standard protocol fields
2. **Baseline Monitoring:** Establish normal communication patterns
3. **Network Segmentation:** Isolate industrial systems from corporate networks
4. **Protocol Validation:** Implement strict protocol conformance checking
5. **Anomaly Detection:** Deploy systems to detect unusual data patterns

### For HVAC/ICS Security
1. **Firmware Validation:** Verify integrity of industrial system firmware
2. **Communication Encryption:** Encrypt industrial protocol communications
3. **Access Controls:** Implement strong authentication for system access
4. **Regular Auditing:** Periodic security assessments of industrial systems
5. **Incident Response:** Procedures for handling compromised industrial systems

### Example Security Implementation
```python
def validate_hvac_packet(packet_data):
    """Validate HVAC packet for suspicious content"""
    
    # Check for non-standard fields
    suspicious_patterns = [
        r'KEY:',
        r'[A-Fa-f0-9]{14,}',  # Long hex strings
        r'HTB\{',             # CTF flag patterns
    ]
    
    for pattern in suspicious_patterns:
        if re.search(pattern, packet_data):
            log_security_alert(f"Suspicious pattern detected: {pattern}")
            return False
    
    # Validate against known HVAC parameters
    valid_hvac_fields = [
        'TEMP:', 'SETPOINT:', 'FAN:', 'COOL:', 
        'HEAT:', 'COMP:', 'PRES:', 'STATE:'
    ]
    
    return any(field in packet_data for field in valid_hvac_fields)
```

## Tools and Techniques Used

### Primary Analysis Tools
- **Wireshark:** Network packet analysis and protocol dissection
- **Python:** Hex decoding and string manipulation
- **Regular Expressions:** Pattern matching for suspicious data

### Key Wireshark Filters
```
tcp contains "KEY:"
tcp contains "TEMP:"
tcp contains "4854427b"
frame.len > 100
```

### Python Analysis Script
```python
def analyze_hvac_exfiltration(pcap_file):
    """Complete analysis script for HVAC data exfiltration"""
    
    # Extract suspicious hex patterns
    hex_patterns = extract_hex_from_pcap(pcap_file)
    
    # Decode hex to ASCII
    decoded_fragments = []
    for hex_str in hex_patterns:
        decoded = bytes.fromhex(hex_str).decode('ascii')
        decoded_fragments.append(decoded)
    
    # Reconstruct hidden message
    hidden_message = ''.join(decoded_fragments)
    
    return hidden_message
```

## Flag Analysis

The flag cleverly incorporates HVAC terminology:
- **"Unlock3d th3 v3nt"** - References HVAC ventilation systems
- **"0f d4t4 15rupt1"** - "of data interrupt" - indicating data exfiltration/disruption

This wordplay connects the technical solution to the industrial theme of the challenge.

## Write-Up Credit: [SR20DET](https://ctf.hackthebox.com/user/profile/605510)