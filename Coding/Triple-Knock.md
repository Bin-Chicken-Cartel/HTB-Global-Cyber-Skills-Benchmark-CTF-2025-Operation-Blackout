# HTB Operation Blackout - Triple Knock

## Challenge Overview

**Challenge Name:** Triple Knock  
**Category:** Coding / Algorithm Design  
**Difficulty:** easy  
**Flag:** `HTB{kn0ck_kN0cK_Kn0Ck_a5c56af1ef7b195bf92d7d39dfc8e63b}`

Leaked credentials were observed in TOR traffic. Attack logs have been shuffled. The mission is to identify user accounts that are being brute-forced by detecting any user with at least three failed login attempts within a rolling 10-minute window.

## Problem Statement

### Input Format
1. First line: two integers S (number of log entries) and N (number of users)
2. Next S lines contain:
   - `user_id` (e.g., `user_1`)
   - `timestamp` (DD/MM HH:MM format, all months = 30 days)
   - `status` ([success] or [failure])

### Constraints
- 10 ≤ S ≤ 10⁵
- 2 ≤ N ≤ 200

### Objective
Output flagged user IDs in increasing lexicographical order where a user has at least 3 failed login attempts within any 10-minute rolling window.

## Technical Analysis

### Algorithm Design
The solution requires implementing a sliding window algorithm to detect patterns of failed login attempts:

1. **Timestamp Normalization:** Convert DD/MM HH:MM to minutes since start of year
2. **Failure Collection:** Group failed login attempts by user
3. **Sliding Window Detection:** Use two-pointer technique to find 3+ failures within 10-minute window

### Time Complexity
- O(S log S) for sorting timestamps per user
- O(S) for sliding window detection
- Overall: O(S log S)

## Attack Chain

### 1. Timestamp Conversion Implementation
```python
def parse_ts(date, time):
    d, m = map(int, date.split('/'))
    h, mi = map(int, time.split(':'))
    return ((m-1)*30 + (d-1)) * 1440 + h*60 + mi
```

### 2. Failure Pattern Detection
```python
def find_targets(entries):
    fails = defaultdict(list)
    for user, ts, st in entries:
        if st == 'failure':
            fails[user].append(ts)
    
    targets = []
    for user, times in fails.items():
        times.sort()
        i = 0
        for j in range(len(times)):
            while times[j] - times[i] > 10:
                i += 1
            if j - i + 1 >= 3:
                targets.append(user)
                break
    return sorted(targets)
```

### 3. Complete Solution
```python
#!/usr/bin/env python3
import sys
from collections import defaultdict

def parse_ts(date, time):
    d, m = map(int, date.split('/'))
    h, mi = map(int, time.split(':'))
    return ((m-1)*30 + (d-1)) * 1440 + h*60 + mi

def find_targets(entries):
    fails = defaultdict(list)
    for user, ts, st in entries:
        if st == 'failure':
            fails[user].append(ts)
    targets = []
    for user, times in fails.items():
        times.sort()
        i = 0
        for j in range(len(times)):
            while times[j] - times[i] > 10:
                i += 1
            if j - i + 1 >= 3:
                targets.append(user)
                break
    return sorted(targets)

def main():
    data = sys.stdin.read().split()
    it = iter(data)
    S, N = int(next(it)), int(next(it))
    logs = []
    for _ in range(S):
        user = next(it)
        date = next(it)
        time = next(it)
        status = next(it).strip()[1:-1]
        logs.append((user, parse_ts(date, time), status))
    print(' '.join(find_targets(logs)))

if __name__ == "__main__":
    main()
```

### 4. Test Case Validation
**Sample Input:**
```
13 4
user_2 23/07 15:41 [success]
user_1 10/06 05:17 [failure]
user_3 20/04 13:53 [failure]
user_1 06/04 17:07 [success]
user_1 10/06 05:19 [failure]
user_3 18/11 10:32 [success]
user_1 12/08 11:52 [success]
user_1 10/06 05:25 [failure]
user_3 20/04 13:59 [failure]
user_3 24/02 22:44 [failure]
user_3 16/02 17:16 [success]
user_3 20/04 13:54 [failure]
user_3 21/11 11:44 [success]
```

**Expected Output:** `user_1 user_3`

## Key Technical Concepts

### Sliding Window Algorithm
- Two-pointer technique efficiently processes sorted timestamps
- Maintains window constraint while detecting pattern occurrences
- Optimal for time-series pattern detection problems

### Timestamp Normalization
- Converting human-readable timestamps to numeric values enables mathematical operations
- Uniform time representation allows for accurate interval calculations
- Essential for sliding window time-based algorithms

### Pattern Recognition in Security Logs
- Failed login attempt clustering indicates potential brute-force attacks
- Time-based analysis reveals attack patterns and persistence
- Lexicographical sorting ensures consistent output format

## Mitigation Recommendations

### Brute Force Protection
- Implement account lockout after multiple failed attempts
- Use exponential backoff for repeated failures
- Deploy CAPTCHA systems after initial failed attempts

### Monitoring and Detection
- Real-time analysis of authentication logs for pattern detection
- Automated alerting for users exceeding failure thresholds
- Geographic and temporal anomaly detection for login attempts

### Response Mechanisms
- Temporary account suspension for flagged users
- Multi-factor authentication requirements after suspicious activity
- IP-based blocking for sources generating multiple failures

## Tools Used

- Python 3 with collections module for efficient data structures
- Standard input/output processing for competitive programming format
- Algorithmic techniques: sorting, sliding window, two-pointer method

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)