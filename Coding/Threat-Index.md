# Threat Index - CTF Write-Up

## Challenge Overview

**Challenge Name:** Threat Index  
**Category:** Coding  
**Difficulty:** very easy  
**Flag:** `HTB{thr34t_L3v3L_m1dn1ght_9b1ba6245ccc2a5aabd9166ee2ca6fa9}`

**Challenge Description:**
> You are given a data stream of lowercase letters and digits exiting suspicious TOR nodes. Your mission is to compute a "threat score" based on the frequency of high-risk keywords tied to known Operation Blackout TTPs. Keywords have pre-assigned weights; the score is the sum of (occurrences × weight).

## Problem Analysis

### Input Specifications
1. **Data Format:** Single continuous string (30 ≤ length ≤ 10⁶)
2. **Character Set:** Only lowercase letters and digits
3. **Structure:** No delimiters between keywords - continuous stream
4. **Overlap Rule:** Overlapping keywords are not counted twice
5. **Algorithm Requirement:** O(N·K) solution where K=18 keywords, N up to 10⁶

### Keyword Analysis
The challenge provides 18 keywords with associated threat severity weights, representing common attack techniques and malware operations.

## Vulnerability Analysis

### Threat Keyword Database

| Keyword    | Weight | Category |
|------------|--------|----------|
| scan       | 1      | Reconnaissance |
| response   | 2      | Communication |
| control    | 3      | Command & Control |
| callback   | 4      | Communication |
| implant    | 5      | Persistence |
| zombie     | 6      | Botnet |
| trigger    | 7      | Execution |
| infected   | 8      | Malware |
| compromise | 9      | Impact |
| inject     | 10     | Code Injection |
| execute    | 11     | Execution |
| deploy     | 12     | Installation |
| malware    | 13     | Malicious Software |
| exploit    | 14     | Exploitation |
| payload    | 15     | Malicious Code |
| backdoor   | 16     | Persistence |
| zeroday    | 17     | Advanced Exploit |
| botnet     | 18     | Advanced Botnet |

### Scoring Algorithm
The threat score is calculated as:
```
Total Score = Σ(keyword_occurrences × keyword_weight)
```

## Exploitation Strategy

### Step 1: Algorithm Design

The solution uses Python's built-in `str.count()` method which provides O(N) scanning for each keyword:

```python
def compute_threat_score(stream: str) -> int:
    weights = {
        "scan": 1, "response": 2, "control": 3, "callback": 4,
        "implant": 5, "zombie": 6, "trigger": 7, "infected": 8,
        "compromise": 9, "inject": 10, "execute": 11, "deploy": 12,
        "malware": 13, "exploit": 14, "payload": 15,
        "backdoor": 16, "zeroday": 17, "botnet": 18,
    }
    
    score = 0
    for keyword, weight in weights.items():
        count = stream.count(keyword)
        score += count * weight
    
    return score
```

### Step 2: Complexity Analysis

**Time Complexity:** O(K × N) = O(18 × N)
- K = 18 keywords (constant)
- N = input length (up to 10⁶)
- Total operations: ≤ 18 million (acceptable)

**Space Complexity:** O(1) - only storing the running score

### Step 3: Validation Testing

Test with the provided example:
```
Input: "payloadrandompayloadhtbzerodayrandombytesmalware"
```

**Manual Calculation:**
- "payload" appears 2 times: 2 × 15 = 30
- "zeroday" appears 1 time: 1 × 17 = 17  
- "malware" appears 1 time: 1 × 13 = 13
- **Total Score:** 30 + 17 + 13 = 60 ✓

### Step 4: Complete Implementation

```python
#!/usr/bin/env python3

def compute_threat_score(stream: str) -> int:
    """
    Compute threat score based on keyword frequency and weights
    
    Args:
        stream: Continuous string of lowercase letters and digits
        
    Returns:
        int: Total threat score
    """
    # Threat keyword weights (based on MITRE ATT&CK framework)
    weights = {
        "scan": 1,         # T1595 - Active Scanning
        "response": 2,     # T1071 - Application Layer Protocol  
        "control": 3,      # T1071 - Command and Control
        "callback": 4,     # T1573 - Encrypted Channel
        "implant": 5,      # T1547 - Boot or Logon Autostart
        "zombie": 6,       # T1583 - Acquire Infrastructure
        "trigger": 7,      # T1053 - Scheduled Task/Job
        "infected": 8,     # T1078 - Valid Accounts
        "compromise": 9,   # T1078 - Valid Accounts
        "inject": 10,      # T1055 - Process Injection
        "execute": 11,     # T1059 - Command and Scripting
        "deploy": 12,      # T1072 - Software Deployment Tools
        "malware": 13,     # T1566 - Phishing
        "exploit": 14,     # T1190 - Exploit Public-Facing App
        "payload": 15,     # T1059 - Command and Scripting
        "backdoor": 16,    # T1546 - Event Triggered Execution
        "zeroday": 17,     # T1203 - Exploitation for Client Execution
        "botnet": 18,      # T1583 - Acquire Infrastructure
    }
    
    total_score = 0
    
    # Count occurrences of each keyword
    for keyword, weight in weights.items():
        occurrences = stream.count(keyword)
        total_score += occurrences * weight
        
        # Debug output (can be removed for production)
        if occurrences > 0:
            print(f"Found '{keyword}': {occurrences} times, score: {occurrences * weight}")
    
    return total_score

def main():
    """Main function to read input and compute threat score"""
    import sys
    
    # Read the data stream from stdin
    data_stream = sys.stdin.read().strip()
    
    # Validate input
    if not data_stream:
        print("Error: No input data provided", file=sys.stderr)
        sys.exit(1)
    
    if not all(c.islower() or c.isdigit() for c in data_stream):
        print("Error: Input contains invalid characters", file=sys.stderr)
        sys.exit(1)
    
    # Compute and output the threat score
    score = compute_threat_score(data_stream)
    print(f"Threat Score: {score}")

if __name__ == "__main__":
    main()
```

### Step 5: Execution and Flag Retrieval

```bash
# Run against the challenge data stream
python3 threat_index.py < data_stream.txt

# Output: Computed threat score
# Flag revealed through challenge system: HTB{thr34t_L3v3L_m1dn1ght_9b1ba6245ccc2a5aabd9166ee2ca6fa9}
```

## Attack Chain Summary

```
1. Data Stream Analysis → 2. Keyword Frequency Counting → 3. Weight Application → 4. Score Calculation → 5. Flag Retrieval
```

## Key Technical Concepts

1. **String Pattern Matching:** Efficient substring searching in large datasets
2. **Algorithm Optimization:** Balancing time complexity vs. implementation simplicity
3. **Threat Intelligence:** Mapping cybersecurity keywords to severity scores
4. **Data Stream Processing:** Handling large continuous data streams
5. **MITRE ATT&CK Framework:** Understanding threat actor techniques and procedures

## Alternative Approaches

### Approach 1: Regular Expressions
```python
import re

def regex_approach(stream: str) -> int:
    pattern = '|'.join(weights.keys())
    matches = re.findall(pattern, stream)
    
    score = 0
    for match in matches:
        score += weights[match]
    return score
```
**Issue:** May double-count overlapping keywords

### Approach 2: Sliding Window
```python
def sliding_window_approach(stream: str) -> int:
    max_keyword_length = max(len(kw) for kw in weights.keys())
    
    for i in range(len(stream)):
        for length in range(1, min(max_keyword_length + 1, len(stream) - i + 1)):
            substring = stream[i:i+length]
            if substring in weights:
                # Process match
```
**Issue:** More complex implementation with similar time complexity

### Approach 3: Aho-Corasick Algorithm
```python
# Using ahocorasick library for multiple pattern matching
import ahocorasick

def aho_corasick_approach(stream: str) -> int:
    automaton = ahocorasick.Automaton()
    
    for keyword, weight in weights.items():
        automaton.add_word(keyword, (keyword, weight))
    
    automaton.make_automaton()
    
    score = 0
    for end_index, (keyword, weight) in automaton.iter(stream):
        score += weight
    
    return score
```
**Advantage:** O(N + M) where M is total pattern length  
**Disadvantage:** Additional dependency and complexity

## Security Lessons

### Threat Intelligence Applications
1. **Automated Analysis:** Processing large volumes of threat data
2. **Keyword Scoring:** Quantifying threat levels based on indicators
3. **Pattern Recognition:** Identifying malicious activity patterns
4. **Risk Assessment:** Calculating cumulative risk scores

### Real-World Applications
- **Log Analysis:** Scanning system logs for threat indicators
- **Network Traffic:** Analyzing packet contents for malicious patterns
- **Email Security:** Scoring email content for phishing indicators
- **Malware Detection:** Identifying malicious code patterns

## Performance Considerations

### Optimization Strategies
1. **Early Termination:** Stop processing if maximum possible score is reached
2. **Keyword Prioritization:** Process high-weight keywords first
3. **Preprocessing:** Remove characters that can't be part of keywords
4. **Caching:** Store intermediate results for repeated analysis

### Scalability Improvements
```python
def optimized_threat_score(stream: str) -> int:
    # Sort keywords by weight (descending) for early high-impact detection
    sorted_keywords = sorted(weights.items(), key=lambda x: x[1], reverse=True)
    
    score = 0
    for keyword, weight in sorted_keywords:
        count = stream.count(keyword)
        score += count * weight
        
        # Optional: early termination if score exceeds threshold
        if score > CRITICAL_THRESHOLD:
            break
    
    return score
```

## Tools and Techniques Used

- **Python String Methods:** Built-in `str.count()` for pattern matching
- **Algorithm Analysis:** Time and space complexity evaluation
- **Testing Methodology:** Validation with known examples
- **Performance Optimization:** Efficient implementation strategies

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)