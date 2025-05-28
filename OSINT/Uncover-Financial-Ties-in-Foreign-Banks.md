# Uncover Financial Ties in Foreign Banks - CTF Write-Up

## Challenge Overview

**Challenge Name:** Uncover Financial Ties in Foreign Banks  
**Category:** OSINT / Financial Intelligence / Cryptocurrency Analysis  
**Difficulty:** easy  
**Flag:** `HTB{SWISS_MERIDIAN_BANK_ACCT_9276514}`

**Challenge Description:**
> Following the identification of a front company for Volnaya Corporation, intelligence indicates they've established a complex financial network to fund Operation Blackout. A specific bank account serves as the central hub for these operations. Your task is to trace the money flow through offshore entities and identify the specific bank account used to finance Volnaya's clandestine activities.

## Initial Reconnaissance

### Challenge Resources
- **Briefing Document:** `Phase 42 Brief Doc.pdf` containing initial intelligence
- **Four Investigation Platforms:**
  - OffshoreLeaks Database
  - BlockTrace Explorer (cryptocurrency analysis)
  - Corporate Ownership Tracker
  - FinDocs Archive

### Key Starting Points
- **Target Organization:** Volnaya Corporation
- **Front Company:** GlobaLynx Industrial Systems
- **Operation:** Operation Blackout
- **Crypto Wallet:** `0xA7F9C32E5B61D48`

## Vulnerability Analysis

### Financial Network Structure
The investigation reveals a complex financial network designed to:
1. **Obscure Fund Origins:** Multiple layers of entities to hide Volnaya's involvement
2. **Facilitate Cross-Border Transfers:** International banking relationships for regulatory evasion
3. **Enable Cryptocurrency Integration:** Digital assets for transaction anonymity
4. **Centralize Fund Distribution:** Single hub account for operational funding

## Exploitation Strategy

### Step 1: Briefing Document Analysis

Extract key intelligence from the provided briefing:
- **Known Front:** GlobaLynx Industrial Systems (previously identified)
- **Primary Target:** Volnaya Corporation as main entity
- **Cryptocurrency Entry Point:** Specific wallet address for transaction tracing
- **Available Tools:** Four specialized investigation platforms

### Step 2: BlockTrace Explorer Investigation

Start with cryptocurrency analysis using the known wallet address:

**Initial Dashboard Metrics:**
- **Total Wallets:** 9 tracked entities
- **Total Transactions:** 11 recorded transfers
- **Peak Transaction:** 25.34 ETH (highest single transfer)

**Key Entities Identified:**
- **GlobaLynx** (Risk Level: 55)
- **Swiss Meridian Bank** (Risk Level: 40)

### Step 3: Transaction Pattern Analysis

Analyze recent transaction flows for patterns and recurring references:

**Critical Transaction Sequence:**
```
Transaction 1:
Sana Petrova → Operation Blackout: 1.2760 ETH
Note: "Operational expense authorization - Reference: SMB-9276514"

Transaction 2:
Volnaya Corporation → Swiss Meridian Bank: 35.4320 ETH
Note: "Additional funding deposit to Swiss Meridian Bank account #9276514"

Transaction 3:
Swiss Meridian Bank → Operation Blackout: 42.5430 ETH
Note: "Major project funding transfer from Swiss Meridian Bank account #9276514"
```

**🔍 Key Discovery:** Account number **#9276514** appears consistently across multiple high-value transactions.

### Step 4: Swiss Meridian Bank Entity Profiling

Investigate the Swiss Meridian Bank entity for comprehensive details:

**Entity Profile:**
- **Institution Type:** Swiss private banking institution
- **Primary Function:** Handling accounts for multinational corporations
- **Associated Account:** #9276514
- **Current Balance:** 73.9284 ETH
- **Transaction Volume:** Highest among all tracked entities

**Complete Transaction History:**
```
1. Volnaya Corporation → Swiss Meridian Bank (35.4320 ETH)
   "Additional funding deposit to Swiss Meridian Bank account #9276514"

2. Swiss Meridian Bank → Operation Blackout (42.5430 ETH)
   "Major project funding transfer from Swiss Meridian Bank account #9276514"

3. Swiss Meridian Bank → TechVentures SG (12.1230 ETH)
   "Investment transfer - Reference: SMB-9276514"

4. Elena Petrova → Swiss Meridian Bank (2.7960 ETH)
   "Transfer authorization for account #9276514"

5. Horizon Capital → Swiss Meridian Bank (18.7250 ETH)
   "Deposit to Swiss Meridian Bank Account #9276514"
```

### Step 5: Network Mapping and Verification

Cross-reference connected entities to confirm central hub status:

**Connected Entity Analysis:**

| Entity | Risk Level | Role | Relationship to SMB-9276514 |
|--------|------------|------|------------------------------|
| Volnaya Corporation | 75 | Primary funding source | Major depositor |
| Horizon Capital | 55 | Investment vehicle | Regular contributor |
| TechVentures SG | 50 | Singapore front | Funding recipient |
| Elena Petrova | 70 | Financial controller | Account operator |
| Operation Blackout | 85 | Primary beneficiary | Main recipient |

### Step 6: Central Hub Confirmation

Evidence supporting Swiss Meridian Bank account #9276514 as the central funding hub:

**Validation Criteria:**
✅ **Multiple Input Sources:** Receives funds from Volnaya Corporation, Horizon Capital, and Elena Petrova  
✅ **Primary Distribution Point:** Distributes funds to Operation Blackout and operational entities  
✅ **Consistent Reference:** Account #9276514 appears in ALL major transactions  
✅ **High-Value Handler:** Manages largest volume of funds (73.9284 ETH balance)  
✅ **Direct Operation Link:** Explicitly funds Operation Blackout activities  
✅ **Network Centrality:** All major entities either deposit to or receive from this account

### Step 7: Flag Construction

Based on comprehensive investigation:
- **Bank Name:** Swiss Meridian Bank
- **Account Number:** 9276514 (formatting: remove # symbol)
- **Flag Format:** `HTB{BANK_NAME_ACCT_NUMBER}`

## Attack Chain Summary

```
1. Briefing Analysis → 2. Cryptocurrency Investigation → 3. Transaction Pattern Analysis → 4. Entity Profiling → 5. Network Mapping → 6. Central Hub Identification → 7. Flag Construction
```

## Key Investigation Techniques

### 1. Pattern Recognition
- **Method:** Identifying recurring account references across transactions
- **Application:** Account #9276514 appeared in all major fund transfers
- **Outcome:** Clear indication of central hub status

### 2. Network Analysis
- **Method:** Mapping connections between financial entities
- **Application:** Tracing fund flows from sources to destinations
- **Outcome:** Visualization of financial network structure

### 3. Transaction Flow Tracing
- **Method:** Following money trails from origin to final destination
- **Application:** Tracking how funds move through the network
- **Outcome:** Understanding operational funding mechanisms

### 4. Entity Relationship Mapping
- **Method:** Understanding connections between organizations
- **Application:** Identifying roles and relationships in the network
- **Outcome:** Complete picture of financial infrastructure

### 5. Financial Intelligence Analysis
- **Method:** Interpreting banking relationships and account structures
- **Application:** Understanding the purpose and function of each entity
- **Outcome:** Strategic intelligence about operational funding

## Technical Concepts

1. **Financial Intelligence (FININT):** Analyzing financial data for intelligence purposes
2. **Cryptocurrency Analysis:** Blockchain transaction investigation techniques
3. **Money Laundering Detection:** Identifying patterns in financial crime
4. **Network Analysis:** Understanding complex financial relationships
5. **OSINT Financial Investigation:** Using open sources for financial intelligence

## Real-World Applications

### Financial Crime Investigation
- **Anti-Money Laundering (AML):** Detecting suspicious financial patterns
- **Sanctions Evasion:** Identifying attempts to circumvent financial restrictions
- **Terrorist Financing:** Tracking funding sources for illicit activities
- **Corporate Fraud:** Investigating complex financial schemes

### Intelligence Operations
- **State Actor Funding:** Understanding nation-state financial operations
- **Criminal Enterprise Mapping:** Identifying financial networks of criminal organizations
- **Economic Espionage:** Investigating foreign financial influence operations
- **Critical Infrastructure Protection:** Understanding threats to financial systems

## Tools and Techniques Used

- **BlockTrace Explorer:** Cryptocurrency transaction analysis platform
- **Entity Relationship Analysis:** Mapping organizational connections
- **Pattern Recognition:** Identifying recurring references and behaviors
- **Financial Flow Analysis:** Tracing money movement through networks
- **Risk Assessment:** Evaluating entity threat levels and relationships

## Mitigation for Financial Networks

### For Investigators
1. **Multi-Source Correlation:** Cross-reference data across multiple platforms
2. **Pattern Analysis:** Look for consistent references and behaviors
3. **Network Visualization:** Map relationships between entities
4. **Transaction Timeline:** Understand chronological flow of funds
5. **Risk Assessment:** Evaluate the significance of each entity

### For Financial Institutions
1. **Enhanced Due Diligence:** Thorough vetting of high-risk accounts
2. **Transaction Monitoring:** Real-time analysis of unusual patterns
3. **Network Analysis:** Understanding customer relationships and connections
4. **Regulatory Compliance:** Adherence to AML and sanctions requirements
5. **Intelligence Sharing:** Collaboration with law enforcement and regulators

## Write-Up Credit: [SR20DET](https://ctf.hackthebox.com/user/profile/605510)