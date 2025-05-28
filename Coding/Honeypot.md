# Honeypot Containment - CTF Write-Up

## Challenge Overview

**Challenge Name:** Honeypot  
**Category:** Coding  
**Difficulty:** easy  
**Flag:** `HTB{l34D_th3M_t0_th3_H0nEyP0t_52e33c525d28ba772f6e555d9bae2dce}`

**Challenge Description:**
> An internal router has been breached and is actively trying to move laterally. You must place a single "firewall" (i.e., remove one tree edge) to minimize the number of nodes still reachable from the root (node 1), while ensuring the honeypot node H remains connected.

## Problem Analysis

### Network Security Model
1. **Network Topology:** Tree structure with N nodes (N ≤ 10⁶)
2. **Edge Structure:** N-1 bidirectional edges ("A - B" format)
3. **Firewall Constraint:** Exactly one edge can be "cut" to block traffic
4. **Honeypot Requirement:** Node H must remain in same component as root (node 1)
5. **Objective:** Minimize size of component containing nodes 1 and H

### Security Implications
- **Lateral Movement Prevention:** Cutting edges isolates compromised network segments
- **Honeypot Preservation:** Maintains deception capabilities while containing threats
- **Network Segmentation:** Strategic isolation of network components
- **Incident Response:** Automated containment during active breaches

## Vulnerability Analysis

### Network Containment Strategy
The challenge models a real-world cybersecurity scenario:
1. **Compromised Infrastructure:** Attacker has gained initial access to internal router
2. **Lateral Movement:** Threat actor attempting to expand access across network
3. **Dynamic Containment:** Need to isolate threats while preserving security infrastructure
4. **Honeypot Integration:** Maintain deception assets for intelligence gathering

### Mathematical Approach
Since the network is a tree:
- **Single Path Property:** Only one path exists between any two nodes
- **Subtree Isolation:** Cutting edge (u-v) removes exactly one subtree
- **Constraint Satisfaction:** Only consider subtrees not containing honeypot H
- **Optimization Goal:** Maximize size of removed (isolated) subtree

## Exploitation Strategy

### Step 1: Problem Decomposition

**Key Insight:** To minimize remaining exposed nodes, maximize the size of the removed subtree that doesn't contain the honeypot.

### Step 2: Algorithm Design

```python
#!/usr/bin/env python3
import sys
sys.setrecursionlimit(10**7)

def solve_honeypot_containment():
    """
    Find optimal edge to cut for network containment
    """
    # Read input
    data = sys.stdin.read().strip().split()
    it = iter(data)
    N = int(next(it))
    
    # Build adjacency list
    adj = [[] for _ in range(N+1)]
    for _ in range(N-1):
        a = int(next(it))
        next(it)  # skip '-' separator
        b = int(next(it))
        adj[a].append(b)
        adj[b].append(a)
    
    H = int(next(it))  # Honeypot node
    
    # Initialize tracking arrays
    parent = [0] * (N+1)
    subtree_size = [0] * (N+1)
    contains_honeypot = [False] * (N+1)
    
    return adj, H, parent, subtree_size, contains_honeypot, N
```

### Step 3: Tree Analysis with DFS

```python
def analyze_network_tree(adj, H, parent, subtree_size, contains_honeypot, N):
    """
    Analyze tree structure using iterative DFS for large inputs
    """
    # Iterative DFS for post-order traversal
    stack = [(1, 0, False)]  # (node, parent, visited)
    
    while stack:
        u, p, visited = stack.pop()
        
        if not visited:
            parent[u] = p
            stack.append((u, p, True))  # Mark for post-processing
            
            # Add children to stack
            for v in adj[u]:
                if v == p:  # Skip parent
                    continue
                stack.append((v, u, False))
        else:
            # Post-order processing
            size = 1
            has_honeypot = (u == H)
            
            # Process all children
            for v in adj[u]:
                if v == p:  # Skip parent
                    continue
                size += subtree_size[v]
                if contains_honeypot[v]:
                    has_honeypot = True
            
            subtree_size[u] = size
            contains_honeypot[u] = has_honeypot
    
    return subtree_size, contains_honeypot
```

### Step 4: Optimal Edge Selection

```python
def find_optimal_containment(subtree_size, contains_honeypot, N):
    """
    Find the edge that maximizes isolated network segment
    """
    max_removable = 0
    
    # Check each potential cut point (excluding root)
    for u in range(2, N+1):
        # Only consider subtrees that don't contain honeypot
        if not contains_honeypot[u]:
            max_removable = max(max_removable, subtree_size[u])
    
    # Calculate remaining exposed nodes
    remaining_exposed = N - max_removable
    
    return remaining_exposed
```

### Step 5: Complete Implementation

```python
#!/usr/bin/env python3
import sys
sys.setrecursionlimit(10**7)

def main():
    """
    Main function for honeypot containment solution
    """
    data = sys.stdin.read().strip().split()
    it = iter(data)
    N = int(next(it))
    
    # Build adjacency list from network topology
    adj = [[] for _ in range(N+1)]
    for _ in range(N-1):
        a = int(next(it))
        next(it)  # skip '-' separator  
        b = int(next(it))
        adj[a].append(b)
        adj[b].append(a)
    
    H = int(next(it))  # Honeypot node location
    
    # Initialize analysis arrays
    parent = [0] * (N+1)
    subtree_size = [0] * (N+1)
    contains_honeypot = [False] * (N+1)
    
    # Iterative DFS for post-order traversal (handles large N)
    stack = [(1, 0, False)]
    while stack:
        u, p, visited = stack.pop()
        if not visited:
            parent[u] = p
            stack.append((u, p, True))
            for v in adj[u]:
                if v == p:
                    continue
                stack.append((v, u, False))
        else:
            # Calculate subtree properties
            sz = 1
            has_h = (u == H)
            for v in adj[u]:
                if v == p:
                    continue
                sz += subtree_size[v]
                if contains_honeypot[v]:
                    has_h = True
            subtree_size[u] = sz
            contains_honeypot[u] = has_h
    
    # Find maximum removable subtree (that doesn't contain honeypot)
    max_removable = 0
    for u in range(2, N+1):
        if not contains_honeypot[u]:
            max_removable = max(max_removable, subtree_size[u])
    
    # Output remaining exposed nodes after optimal containment
    print(N - max_removable)

if __name__ == "__main__":
    main()
```

### Step 6: Algorithm Verification

**Test Case Analysis:**
```
Input:
7 nodes
Edges: 1-2, 1-3, 3-4, 3-5, 1-6, 6-7
Honeypot H = 5

Network Structure:
    1 (root)
   /|\ 
  2 3 6
   /|  |
  4 5  7
    ^
  (honeypot)

Optimal Cut: Edge (1-6)
- Removes subtree {6,7} of size 2
- Honeypot remains connected via path 1→3→5
- Remaining exposed: 7 - 2 = 5 nodes
```

## Attack Chain Summary

```
1. Network Topology Analysis → 2. Tree Structure Parsing → 3. DFS Traversal → 4. Subtree Size Calculation → 5. Honeypot Path Analysis → 6. Optimal Edge Selection
```

## Key Technical Concepts

1. **Tree Algorithms:** Graph theory applied to network topology
2. **Dynamic Programming:** Optimal substructure for containment decisions
3. **Network Segmentation:** Strategic isolation of network components
4. **Incident Response:** Automated threat containment strategies
5. **Honeypot Integration:** Maintaining deception while containing threats

## Real-World Applications

### Network Security
```python
class NetworkContainment:
    """Real-world network containment system"""
    
    def __init__(self, network_topology, critical_assets):
        self.topology = network_topology
        self.critical_assets = critical_assets
        self.honeypots = self.identify_honeypots()
    
    def contain_lateral_movement(self, compromised_node):
        """Implement automated containment"""
        # Calculate optimal firewall rules
        optimal_cuts = self.calculate_containment_strategy(compromised_node)
        
        # Preserve honeypot connectivity
        for cut in optimal_cuts:
            if not self.affects_honeypot_connectivity(cut):
                self.implement_firewall_rule(cut)
                break
    
    def calculate_containment_strategy(self, threat_source):
        """Calculate optimal network segmentation"""
        containment_options = []
        
        for edge in self.topology.edges:
            isolation_size = self.calculate_isolation_impact(edge)
            honeypot_preserved = self.preserve_honeypot_access(edge)
            
            if honeypot_preserved:
                containment_options.append((edge, isolation_size))
        
        # Return option that maximizes isolation
        return sorted(containment_options, key=lambda x: x[1], reverse=True)
```

### Incident Response Automation
```python
def automated_containment_response(network, threat_indicators):
    """Automated threat containment system"""
    
    # Detect lateral movement patterns
    compromised_nodes = detect_lateral_movement(threat_indicators)
    
    for node in compromised_nodes:
        # Calculate containment impact
        containment_plan = calculate_optimal_isolation(network, node)
        
        # Implement containment while preserving honeypots
        if containment_plan.preserves_honeypots():
            implement_containment(containment_plan)
            log_containment_action(node, containment_plan)
```

## Performance Optimization

### Scalability Considerations
```python
def optimized_containment_analysis(network_size):
    """Handle large network topologies efficiently"""
    
    if network_size > 10**5:
        # Use iterative DFS to avoid recursion limits
        return iterative_tree_analysis()
    else:
        # Use recursive approach for smaller networks
        return recursive_tree_analysis()

def iterative_tree_analysis():
    """Memory-efficient tree analysis for large networks"""
    # Implementation handles large N without stack overflow
    pass
```

### Time Complexity Analysis
- **Tree Traversal:** O(N) for DFS traversal
- **Subtree Calculation:** O(N) for size computation  
- **Optimal Selection:** O(N) for finding maximum
- **Total Complexity:** O(N) - Linear time solution

## Security Implications

### Threat Containment
1. **Rapid Response:** Automated containment reduces incident response time
2. **Selective Isolation:** Minimize business impact while containing threats
3. **Intelligence Preservation:** Maintain honeypots for ongoing threat intelligence
4. **Network Resilience:** Strategic segmentation improves overall security posture

### Honeypot Integration
1. **Deception Continuity:** Keep honeypots accessible to attackers
2. **Intelligence Gathering:** Maintain ability to monitor threat actor behavior
3. **Attribution Support:** Preserve evidence collection capabilities
4. **Operational Security:** Balance containment with intelligence requirements

## Tools and Techniques Used

- **Graph Theory:** Tree algorithms for network topology analysis
- **Dynamic Programming:** Optimal containment strategy calculation
- **Python Implementation:** Efficient algorithms for large-scale networks
- **Complexity Analysis:** Performance optimization for real-world deployment

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)