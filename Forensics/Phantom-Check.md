# Phantom Check - CTF Write-Up

## Challenge Overview

**Challenge Name:** Phantom Check  
**Category:** Forensics / Windows Event Log Analysis  
**Difficulty:** very easy

**Challenge Description:**
> Talion suspects that the threat actor carried out anti-virtualization checks to avoid detection in sandboxed environments. Our goal was to parse the Windows Event Logs (.evtx) and extract evidence of the registry checks, WMI queries, functions, and output used by the attacker's VM-detection script.

## Initial Reconnaissance

### Investigation Objectives
The challenge requires analyzing Windows Event Logs to answer 6 specific questions about VM detection techniques:
1. **Q1:** WMI class for model/manufacturer detection
2. **Q2:** Exact WMI query for current temperature
3. **Q3:** Function name of the VM-detection script
4. **Q4:** Registry path for service information
5. **Q5:** VirtualBox processes checked
6. **Q6:** Detected virtualization platforms

### Tools Required
- **PowerShell** for event log analysis
- **Windows Event Viewer** for manual verification
- **Regular Expressions** for pattern extraction
- **Event Log Analysis** techniques

## Vulnerability Analysis

### Anti-Virtualization Techniques
The malware implements multiple evasion techniques:
1. **WMI Queries:** Hardware manufacturer and model detection
2. **Temperature Sensors:** Physical hardware presence verification
3. **Registry Analysis:** Virtualization service detection
4. **Process Enumeration:** Hypervisor-specific process identification
5. **Environmental Checks:** Multiple validation layers

### Detection Evasion Strategy
- **Sandbox Evasion:** Avoid executing malicious payload in analysis environments
- **Dynamic Analysis Bypass:** Prevent behavior observation in VMs
- **Multi-Layer Validation:** Combine multiple detection methods for reliability
- **Automated Decision Making:** Script-based environment classification

## Exploitation Analysis

### Complete PowerShell Analysis Script

```powershell
<#
.SYNOPSIS
Extract VM-detection answers (Q1–Q6) from all .evtx in the current folder,
including pipeline (4103) and script-block (4104) events for complete analysis.
#>

# 1. Gather all .evtx files in current directory
$evtxFiles = Get-ChildItem -Filter '*.evtx' -File

# 2. Extract both pipeline (ID 4103) and script-block (ID 4104) events
$events = foreach ($f in $evtxFiles) {
    Get-WinEvent -Path $f.FullName |
      Where-Object { $_.Id -in 4103,4104 } |
      Select-Object Id, TimeCreated, Message
}

# 3. Q1: WMI class for model/manufacturer detection
$q1 = ($events |
    Where-Object { $_.Id -eq 4104 } |
    Select-String -Pattern '-Class\s+([A-Za-z0-9_]+)' -AllMatches |
    ForEach-Object { $_.Matches | ForEach-Object { $_.Groups[1].Value } } |
    Where-Object { $_ -like 'Win32_*' } |
    Select-Object -First 1)

if (-not $q1) {
    $q1 = ($events |
        Where-Object { $_.Id -eq 4104 } |
        Select-String -Pattern 'FROM\s+(Win32_[A-Za-z0-9_]+)' |
        ForEach-Object { $_.Matches[0].Groups[1].Value } |
        Select-Object -First 1)
}

# 4. Q2: Exact WMI query for current temperature
$q2 = ($events |
    Where-Object { $_.Id -eq 4104 } |
    Select-String -Pattern '-Query\s+"([^\"]*MSAcpi_ThermalZoneTemperature[^\"]*)"' |
    ForEach-Object { $_.Matches[0].Groups[1].Value } |
    Select-Object -First 1)

# 5. Q3: Function name of the VM-detection script
$q3 = ($events |
    Where-Object { $_.Id -eq 4104 } |
    Select-String -Pattern 'function\s+([A-Za-z0-9\-_]+)' -AllMatches |
    ForEach-Object { $_.Matches | ForEach-Object { $_.Groups[1].Value } } |
    Where-Object { $_ -match 'Check|Detect|Test' } |
    Select-Object -First 1)

# 6. Q4: Registry path for service information
$q4 = ($events |
    Where-Object { $_.Id -eq 4104 } |
    Select-String -Pattern 'HKLM:\\SYSTEM[^"\s]+' -AllMatches |
    ForEach-Object { $_.Matches | ForEach-Object { $_.Groups[0].Value } } |
    Select-Object -Unique)

# 7. Q5: VirtualBox processes checked
$q5 = ($events |
    Where-Object { $_.Id -eq 4104 } |
    Select-String -Pattern '(vbox[a-z]*\.exe)' -AllMatches |
    ForEach-Object { $_.Matches | ForEach-Object { $_.Groups[1].Value } } |
    Select-Object -Unique |
    Select-Object -First 2) -join ':'

# 8. Q6: Platforms detected via "This is a ..." output
$q6 = ($events |
    Where-Object { $_.Id -eq 4103 } |
    Select-String -Pattern 'This is a\s+"?([A-Za-z0-9\-_]+)(?:\s+machine)?"?' -AllMatches |
    ForEach-Object { foreach ($m in $_.Matches) { $m.Groups[1].Value } } |
    Select-Object -Unique |
    Select-Object -First 2) -join ':'

# 9. Display results
Write-Host "`n==== VM-DETECTION SCRIPT ANSWERS ====`n"
Write-Host "Q1: $q1"
Write-Host "Q2: $q2"  
Write-Host "Q3: $q3"
Write-Host "Q4: $($q4 -join ', ')"
Write-Host "Q5: $q5"
Write-Host "Q6: $q6"
Write-Host "`n===================================== `n"
```

### Investigation Results

| Question | Answer | Evidence Source |
|----------|--------|-----------------|
| **Q1** | Win32_ComputerSystem | Script-block logging (Event ID 4104) |
| **Q2** | SELECT * FROM MSAcpi_ThermalZoneTemperature | WMI query pattern matching |
| **Q3** | Check-VM | Function definition extraction |
| **Q4** | HKLM:\SYSTEM\ControlSet001\Services | Registry path analysis |
| **Q5** | vboxservice.exe:vboxtray.exe | Process name pattern matching |
| **Q6** | Hyper-V:VMWare | Pipeline output analysis (Event ID 4103) |

## Attack Chain Analysis

### 1. Hardware Manufacturer Detection (Q1 & Q2)
```powershell
# WMI class used for hardware detection
Get-WmiObject -Class Win32_ComputerSystem

# Temperature sensor query to detect physical hardware
Get-WmiObject -Query "SELECT * FROM MSAcpi_ThermalZoneTemperature"
```

**Purpose:** Determine if system is running on physical hardware vs. virtual environment

### 2. VM Detection Function (Q3)
```powershell
function Check-VM {
    # Primary VM detection logic
    # Combines multiple detection methods
}
```

**Purpose:** Centralized function for comprehensive virtualization detection

### 3. Service Registry Analysis (Q4)
```powershell
# Check for virtualization services in registry
Get-ChildItem -Path "HKLM:\SYSTEM\ControlSet001\Services"
```

**Purpose:** Detect hypervisor-related services and drivers in Windows registry

### 4. VirtualBox Process Detection (Q5)
```powershell
# Check for VirtualBox-specific processes
Get-Process -Name "vboxservice","vboxtray" -ErrorAction SilentlyContinue
```

**Purpose:** Identify running VirtualBox guest addition processes

### 5. Environment Classification (Q6)
```powershell
# Output actual detection results
Write-Host "This is a Hyper-V machine."
Write-Host "This is a VMWare machine."
```

**Purpose:** Report detected virtualization platform for conditional execution

## Attack Chain Summary

```
1. Function Initialization → 2. Hardware Query → 3. Temperature Check → 4. Registry Analysis → 5. Process Enumeration → 6. Platform Classification → 7. Conditional Execution
```

## Key Technical Concepts

1. **Windows Event Log Analysis:** PowerShell event log processing techniques
2. **Anti-Virtualization Techniques:** Methods to detect sandbox environments
3. **WMI (Windows Management Instrumentation):** System information query interface
4. **Registry Forensics:** Analyzing Windows registry for security artifacts
5. **Process Analysis:** Identifying hypervisor-specific processes

## VM Detection Techniques Explained

### 1. Hardware Manufacturer Detection
```powershell
# Detect virtualization through manufacturer strings
$computer = Get-WmiObject -Class Win32_ComputerSystem
if ($computer.Manufacturer -match "VMware|VirtualBox|Microsoft Corporation") {
    # Virtual environment detected
}
```

### 2. Temperature Sensor Analysis
```powershell
# Physical hardware typically has thermal sensors
$temp = Get-WmiObject -Query "SELECT * FROM MSAcpi_ThermalZoneTemperature"
if (-not $temp) {
    # Likely virtual environment (no thermal sensors)
}
```

### 3. Service Registry Inspection
```powershell
# Check for virtualization services
$services = Get-ChildItem "HKLM:\SYSTEM\ControlSet001\Services"
$vmServices = $services | Where-Object { $_.Name -match "vbox|vmware|hyper" }
```

### 4. Process-Based Detection
```powershell
# Look for hypervisor-specific processes
$vboxProcesses = @("vboxservice.exe", "vboxtray.exe")
foreach ($process in $vboxProcesses) {
    if (Get-Process -Name $process -ErrorAction SilentlyContinue) {
        # VirtualBox detected
    }
}
```

## Forensic Analysis Methodology

### Event Log Processing
1. **Event ID 4103:** Pipeline execution events (command output)
2. **Event ID 4104:** Script block logging (PowerShell code execution)
3. **Pattern Matching:** Regular expressions for data extraction
4. **Cross-Validation:** Multiple data sources for accuracy

### Evidence Correlation
- **Script Blocks:** Actual PowerShell code execution
- **Pipeline Events:** Command output and results
- **Timing Analysis:** Understanding execution sequence
- **Context Analysis:** Correlating events across log files

## Detection and Mitigation

### For Security Analysts
```powershell
# Monitor for VM detection scripts
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-PowerShell/Operational'
    ID=4104
} | Where-Object { 
    $_.Message -match 'Win32_ComputerSystem|MSAcpi_ThermalZone|vbox|vmware'
}
```

### For Sandbox Operators
1. **Hardware Emulation:** Implement realistic hardware profiles
2. **Service Masking:** Hide virtualization-specific services
3. **Process Injection:** Simulate physical hardware processes
4. **Registry Manipulation:** Modify registry to appear physical
5. **Thermal Simulation:** Implement fake temperature sensors

### Example Sandbox Evasion Detection
```powershell
function Detect-SandboxEvasion {
    param($EventLogs)
    
    $evasionIndicators = @(
        'Win32_ComputerSystem',
        'MSAcpi_ThermalZoneTemperature',
        'vboxservice.exe',
        'HKLM:\\SYSTEM.*Services'
    )
    
    foreach ($indicator in $evasionIndicators) {
        $matches = $EventLogs | Select-String -Pattern $indicator
        if ($matches) {
            Write-Warning "Sandbox evasion technique detected: $indicator"
        }
    }
}
```

## Tools and Techniques Used

- **PowerShell Event Log Analysis:** Get-WinEvent cmdlet for log processing
- **Regular Expression Matching:** Pattern extraction from log messages
- **Cross-Event Correlation:** Linking script blocks with pipeline output
- **Automated Analysis:** Batch processing of multiple event log files

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069) & ## Write-Up Credit: [SR20DET](https://ctf.hackthebox.com/user/profile/605510)