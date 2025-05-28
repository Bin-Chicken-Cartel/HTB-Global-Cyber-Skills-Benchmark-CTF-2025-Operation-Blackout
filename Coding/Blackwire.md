# Blackwire - CTF Write-Up

## Challenge Overview

**Challenge Name:** Blackwire  
**Category:** Coding  
**Difficulty:** Medium  
**Flag:** `HTB{f1n4l_st4t3_th3_b0mB_1s_4rm3d_7bdbf012642329cf8397ffc212f35e13}`

**Challenge Description:**
> A firmware update contains a logic bomb implemented as a finite state machine (FSM). You must count how many unique execution paths through an opcode stream advance the FSM from state 0 to its final state T, given a transition table and a sequence of executed opcodes.

## Problem Analysis

### Finite State Machine Specifications
- **States:** 0 → 1 → 2 → ... → T (sequential progression)
- **Transition Table:** T entries, each 20 bits:
  - 12 bits = current state S
  - 8 bits = opcode that allows S → S+1
- **Opcode Stream:** Length L bits (L/8 opcodes total)
- **Transition Rule:** Each matching opcode gives the *option* to advance one state (not forced)
- **Goal:** Count total distinct ways to move strictly one state at a time from 0 to T

### Input Format

| Field | Description |
|-------|-------------|
| T | Number of transitions (and final state index) |
| L | Number of bits in opcode stream (multiple of 8) |
| Next T×20 bits | Transition table entries (concatenated) |
| Next L bits | Opcode execution stream (8 bits per opcode) |

**Constraints:**
- 3 ≤ T ≤ 4,000
- 80 ≤ L ≤ 80,000
- Result fits in 64-bit unsigned integer

## Vulnerability Analysis

### Logic Bomb Implementation
The challenge simulates a **logic bomb** - malicious code that executes when specific conditions are met:

1. **State-Based Activation:** Bomb arms when FSM reaches final state T
2. **Conditional Execution:** Multiple valid paths create timing uncertainty
3. **Firmware Integration:** Embedded in legitimate firmware updates
4. **Stealth Mechanism:** Normal operation until trigger conditions met

### FSM Security Implications
- **Predictable Behavior:** State transitions follow defined rules
- **Path Enumeration:** Multiple execution paths complicate detection
- **Timing Analysis:** Understanding all possible activation scenarios
- **Reverse Engineering:** Analyzing transition tables to understand logic

## Exploitation Strategy

### Step 1: Parse Transition Table

Extract transition information from the binary data:

```python
def parse_transition_table(bits, T):
    """Parse transition table from binary string"""
    transitions = [None] * T
    
    for i in range(T):
        # Extract 20-bit entry
        entry = bits[20*i:20*(i+1)]
        
        # Split into state (12 bits) and opcode (8 bits)
        state = int(entry[:12], 2)
        opcode = int(entry[12:], 2)
        
        # Map state to required opcode for transition
        transitions[state] = opcode
    
    return transitions
```

### Step 2: Decode Opcode Stream

Convert binary data to sequence of 8-bit opcodes:

```python
def decode_opcode_stream(bits, L):
    """Decode opcode stream from binary string"""
    opcodes = []
    
    for i in range(0, L, 8):
        opcode = int(bits[i:i+8], 2)
        opcodes.append(opcode)
    
    return opcodes
```

### Step 3: Dynamic Programming Solution

Use DP to count unique paths through the state machine:

```python
def count_fsm_paths(transitions, opcodes, T):
    """
    Count unique paths from state 0 to state T
    
    dp[s] = number of ways to reach state s
    """
    # Initialize DP array
    dp = [0] * (T + 1)
    dp[0] = 1  # One way to start at state 0
    
    # Process each opcode in sequence
    for opcode in opcodes:
        # Update backwards to avoid overwriting current iteration
        for state in range(T - 1, -1, -1):
            # Check if this opcode allows transition from current state
            if transitions[state] == opcode:
                # Add paths from current state to next state
                dp[state + 1] += dp[state]
    
    return dp[T]
```

### Step 4: Complete Implementation

```python
#!/usr/bin/env python3
import sys

def main():
    # Read input data
    data = sys.stdin.read().split()
    T, L = map(int, data[:2])
    bits = "".join(data[2:])
    
    # Split input into transition table and opcode stream
    table_bits = bits[:20*T]
    stream_bits = bits[20*T:20*T+L]
    
    # Parse transition table: state → required opcode
    transitions = [None] * T
    for i in range(T):
        entry = table_bits[20*i:20*(i+1)]
        state = int(entry[:12], 2)    # First 12 bits
        opcode = int(entry[12:], 2)   # Last 8 bits
        transitions[state] = opcode
    
    # Decode opcode stream
    opcodes = [int(stream_bits[i:i+8], 2) for i in range(0, L, 8)]
    
    # Dynamic programming: dp[s] = number of ways to reach state s
    dp = [0] * (T + 1)
    dp[0] = 1  # Starting position
    
    # Process each opcode
    for opcode in opcodes:
        # Update backwards to prevent overwriting
        for state in range(T - 1, -1, -1):
            if transitions[state] == opcode:
                dp[state + 1] += dp[state]
    
    # Output final result
    print(dp[T])

if __name__ == "__main__":
    main()
```

### Step 5: Algorithm Analysis

**Time Complexity:** O(T + (L/8) × T)
- Transition table parsing: O(T)
- Opcode processing: O((L/8) × T)
- Total: Linear in input size

**Space Complexity:** O(T)
- DP array size: T+1
- Transition table: T entries
- Optimal space usage

## Attack Chain Summary

```
1. Input Parsing → 2. Transition Table Analysis → 3. Opcode Stream Decoding → 4. Path Enumeration via DP → 5. Logic Bomb Path Count
```

## Key Technical Concepts

1. **Finite State Machines:** Computational models with discrete states and transitions
2. **Dynamic Programming:** Optimization technique for counting problems
3. **Logic Bombs:** Malicious code that activates under specific conditions
4. **Firmware Analysis:** Reverse engineering embedded system logic
5. **Path Enumeration:** Counting distinct execution sequences

## Logic Bomb Detection Techniques

### Static Analysis
```python
def analyze_transition_patterns(transitions):
    """Analyze FSM for suspicious patterns"""
    
    # Check for irreversible paths
    reachable_states = set()
    for state, opcode in enumerate(transitions):
        if opcode is not None:
            reachable_states.add(state + 1)
    
    # Identify potential trigger states
    trigger_candidates = []
    for state in range(len(transitions)):
        if state not in reachable_states[:-1]:  # Terminal states
            trigger_candidates.append(state)
    
    return trigger_candidates
```

### Dynamic Analysis
```python
def simulate_execution_paths(transitions, opcodes, max_paths=1000):
    """Simulate FSM execution to identify activation paths"""
    
    paths = []
    
    def dfs(state, path, opcode_index):
        if len(paths) >= max_paths:
            return
        
        if state == len(transitions):  # Reached final state
            paths.append(path.copy())
            return
        
        if opcode_index >= len(opcodes):
            return
        
        current_opcode = opcodes[opcode_index]
        
        # Option 1: Don't advance state
        dfs(state, path, opcode_index + 1)
        
        # Option 2: Advance state if possible
        if transitions[state] == current_opcode:
            path.append((state, current_opcode))
            dfs(state + 1, path, opcode_index + 1)
            path.pop()
    
    dfs(0, [], 0)
    return paths
```

## Security Implications

### Firmware Security
1. **Code Review:** Analyze state machines for suspicious logic
2. **Path Analysis:** Understand all possible execution paths
3. **Timing Analysis:** Identify when logic bombs might activate
4. **Input Validation:** Verify opcode sequences against expected patterns

### Detection Strategies
1. **Static Analysis:** Examine transition tables for anomalous patterns
2. **Dynamic Monitoring:** Monitor state transitions during execution
3. **Behavioral Analysis:** Look for unexpected state machine behaviors
4. **Integrity Checking:** Verify firmware hasn't been tampered with

## Example Verification

### Sample Input
```
3 240
(60 bits of transition table)
(240 bits of opcode stream)
```

### Expected Output
```
4
```

The solution correctly identifies 4 unique paths through the FSM from state 0 to state 3.

## Real-World Applications

### Malware Analysis
- **State Machine Reversing:** Understanding malware logic flows
- **Trigger Condition Analysis:** Identifying activation mechanisms
- **Behavioral Prediction:** Anticipating malware actions

### Firmware Security
- **Logic Bomb Detection:** Identifying malicious state machines
- **Path Coverage Analysis:** Ensuring all execution paths are tested
- **Timing Attack Prevention:** Understanding state transition timing

## Tools and Techniques Used

- **Dynamic Programming:** Efficient path counting algorithm
- **Binary Data Parsing:** Converting bit strings to structured data
- **State Machine Analysis:** Understanding FSM behavior and properties
- **Path Enumeration:** Counting distinct execution sequences

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)