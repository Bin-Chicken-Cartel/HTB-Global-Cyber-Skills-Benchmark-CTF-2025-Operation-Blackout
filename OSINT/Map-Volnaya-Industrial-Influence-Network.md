# Map Volnaya's Industrial Influence Network - CTF Write-Up

## Challenge Overview

**Challenge Name:** Map Volnaya's Industrial Influence Network  
**Category:** OSINT (Open Source Intelligence)  
**Difficulty:** very easy  
**Flag:** `HTB{GLOBALYNX_INDUSTRIAL_SYSTEMS_FRONT}`

**Challenge Description:**
> In this OSINT CTF challenge, we were provided with four Docker URLs. Each of these URLs hosted a system that contained valuable information to help us uncover the name of a shell company. Alongside these systems, we received a PDF document that provided additional context and the required flag format.

## Initial Reconnaissance

### Challenge Resources
- **Four Docker-hosted systems** containing different databases and search functionalities
- **PDF Document** providing context and flag format requirements
- **Flag Format:** `HTB{company_name_FRONT}` where company name should be appended with `_FRONT`

### System Architecture Analysis
Each Docker system contained specialized databases:
1. **Global Shipping Database** - Shipping manifests and logistics data
2. **Corporate News Vault** - Corporate news and business intelligence
3. **Leaked Documents Platform** - Internal corporate documents
4. **Domain Intelligence Platform** - Domain and network infrastructure data

## Vulnerability Analysis

### Information Disclosure Through Multiple Sources
The challenge demonstrates how shell companies can be uncovered through:
1. **Cross-Reference Analysis** - Linking entities across multiple data sources
2. **Corporate Structure Mapping** - Identifying parent-subsidiary relationships
3. **Shipping Manifest Analysis** - Tracking physical deliveries and recipients
4. **Document Correlation** - Connecting entities through leaked internal documents

## Exploitation Strategy

### Step 1: PDF Document Analysis

Review the provided PDF document for context and requirements:
- **Key Focus:** Shipping manifests and corporate connections
- **Investigation Target:** Shell company identification within Volnaya's network
- **Flag Format:** `HTB{company_name_front}` (case sensitivity noted)

### Step 2: Global Shipping Database Investigation

Access the Global Shipping Database system and search for relevant shipping data:

**Methodology:**
```bash
# Search for shipping manifests containing suspicious companies
# Look for patterns in delivery addresses and recipient organizations
# Focus on industrial equipment and systems deliveries
```

**Critical Discovery:**
- Multiple shipping manifests show deliveries addressed to **"Globalynx Industrial Systems"**
- Pattern of high-value industrial equipment shipments
- Delivery locations consistent with critical infrastructure sites

### Step 3: Corporate News Vault Cross-Reference

Use the Corporate News Vault to validate and expand findings:

**Search Strategy:**
```bash
# Search for "Globalynx" and related shipping data
# Look for corporate announcements and business relationships
# Identify connections to other entities
```

**Key Findings:**
- Documents linking "Globalynx Industrial Systems" to "Technovert Solutions"
- Corporate structure indicating subsidiary relationships
- Business dealings suggesting front company operations

### Step 4: Leaked Documents Platform Verification

Access leaked documents to confirm corporate structure:

**Investigation Focus:**
- Internal corporate communications
- Organizational charts and structure documents
- Financial relationships and ownership data

**Confirmation:**
- "Technovert Solutions" identified as subsidiary of "Volnaya Corporation"
- Internal documents confirm "Globalynx Industrial Systems" as operational entity
- Shell company structure verified through leaked organizational charts

### Step 5: Domain Intelligence Correlation

Use domain platform to verify digital footprint:

**Analysis:**
- Domain registration patterns
- Network infrastructure connections
- Digital certificates and ownership data

**Validation:**
- Digital infrastructure consistent with shell company operations
- Domain registrations support corporate structure findings
- Network connections confirm operational relationships

### Step 6: Entity Relationship Mapping

**Confirmed Corporate Structure:**
```
Volnaya Corporation (Parent)
    └── Technovert Solutions (Subsidiary)
        └── Globalynx Industrial Systems (Shell Company/Front)
```

**Shell Company Profile:**
- **Name:** Globalynx Industrial Systems
- **Function:** Industrial equipment procurement and distribution
- **Purpose:** Obscure Volnaya Corporation's direct involvement in critical infrastructure
- **Evidence:** Multiple shipping manifests, corporate documents, leaked communications

## Attack Chain Summary

```
1. PDF Analysis → 2. Shipping Database Investigation → 3. Corporate News Cross-Reference → 4. Leaked Document Verification → 5. Domain Intelligence Validation → 6. Entity Mapping → 7. Flag Construction
```

## OSINT Methodology Applied

### 1. Multi-Source Intelligence Gathering
- **Primary Sources:** Shipping databases, corporate news
- **Secondary Sources:** Leaked documents, domain intelligence
- **Correlation:** Cross-referencing data across all sources

### 2. Corporate Structure Analysis
- **Hierarchy Mapping:** Identifying parent-subsidiary relationships
- **Operational Analysis:** Understanding business functions and purposes
- **Evidence Validation:** Confirming findings through multiple sources

### 3. Pattern Recognition
- **Shipping Patterns:** Identifying consistent delivery patterns
- **Naming Conventions:** Recognizing corporate branding strategies
- **Digital Footprints:** Analyzing online presence and infrastructure

### 4. Source Reliability Assessment
- **Document Authenticity:** Verifying leaked document credibility
- **Cross-Validation:** Confirming information across multiple platforms
- **Timeline Analysis:** Ensuring consistent chronological information

## Key Technical Concepts

1. **Open Source Intelligence (OSINT):** Gathering intelligence from publicly available sources
2. **Corporate Shell Analysis:** Identifying front companies and their purposes
3. **Cross-Reference Investigation:** Correlating information across multiple databases
4. **Entity Relationship Mapping:** Understanding complex corporate structures
5. **Digital Footprint Analysis:** Investigating online presence and infrastructure

## Shell Company Identification Techniques

### Red Flags Identified
1. **Limited Public Information:** Minimal legitimate business presence
2. **Single Purpose Operations:** Focused only on industrial equipment
3. **Complex Ownership Structure:** Multiple layers of corporate ownership
4. **High-Value Transactions:** Large equipment purchases for infrastructure
5. **Geographic Patterns:** Deliveries to sensitive locations

### Validation Methods
1. **Document Triangulation:** Confirming through multiple document sources
2. **Timeline Correlation:** Ensuring consistent operational timelines
3. **Business Logic Analysis:** Evaluating realistic business operations
4. **Digital Verification:** Checking online presence and registration data

## Real-World Applications

### Corporate Intelligence
- **Due Diligence:** Investigating potential business partners
- **Risk Assessment:** Identifying shell companies in supply chains
- **Compliance Monitoring:** Detecting sanctions evasion attempts
- **Competitive Intelligence:** Understanding competitor corporate structures

### Security Applications
- **Threat Actor Attribution:** Identifying front organizations
- **Supply Chain Security:** Vetting vendors and suppliers
- **Critical Infrastructure Protection:** Monitoring access and control
- **Financial Investigation:** Tracking money laundering operations

## Tools and Techniques Used

- **Database Search Techniques:** Advanced querying across multiple systems
- **Document Analysis:** PDF parsing and content extraction
- **Cross-Reference Correlation:** Linking entities across data sources
- **Corporate Structure Mapping:** Visualizing organizational relationships
- **Pattern Recognition:** Identifying suspicious business patterns

## Mitigation for Shell Company Detection

### For Organizations
1. **Enhanced Due Diligence:** Thorough vetting of business partners
2. **Supply Chain Monitoring:** Regular audits of vendor relationships
3. **Open Source Monitoring:** Continuous OSINT on business associates
4. **Financial Transaction Analysis:** Unusual payment pattern detection

### For Investigators
1. **Multi-Source Analysis:** Never rely on single information sources
2. **Timeline Verification:** Ensure chronological consistency
3. **Business Logic Assessment:** Evaluate operational legitimacy
4. **Digital Footprint Investigation:** Comprehensive online presence analysis

## Flag Construction

Based on the comprehensive investigation across all four systems:

**Identified Shell Company:** Globalynx Industrial Systems
**Required Format:** Company name + _FRONT
**Final Flag:** `HTB{GLOBALYNX_INDUSTRIAL_SYSTEMS_FRONT}`

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)