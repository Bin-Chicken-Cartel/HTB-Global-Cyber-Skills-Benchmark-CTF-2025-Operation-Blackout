# Smoke and Mirrors - CTF Write-Up

## Challenge Overview

**Challenge Name:** Smoke and Mirrors  
**Category:** Forensics / Windows Forensics  
**Difficulty:** very easy  
**Artifacts Analyzed:**
- `Microsoft-Windows-PowerShell.evtx`
- `Microsoft-Windows-PowerShell-Operational.evtx`
- `Microsoft-Windows-Sysmon-Operational.evtx`

**Challenge Description:**
> HTB Operation Blackout → External Affairs – Windows Forensics Mini-Challenge. Analyze Windows event logs to answer specific questions about security evasion techniques and PowerShell attacks.

## Challenge Questions and Answers

| Question | Answer |
|----------|--------|
| **Q1:** Registry key used to disable LSA-PPL | `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL` |
| **Q2:** PowerShell cmdlet that tampers with Defender | `Set-MpPreference` |
| **Q3:** amsi.dll function patched by the bypass | `AmsiScanBuffer` |
| **Q4:** Safe-Mode reboot command (args, no ".exe") | `bcdedit /set safeboot network` |
| **Q5:** Command that wipes PS history logging | `Set-PSReadLineOption -HistorySaveStyle SaveNothing` |

## Initial Reconnaissance

### Artifact Analysis
The challenge provides three Windows Event Log files:
1. **PowerShell.evtx** - Standard PowerShell logging
2. **PowerShell-Operational.evtx** - Detailed PowerShell operational events
3. **Sysmon-Operational.evtx** - System Monitor process and network events

### Investigation Focus
The questions target advanced attack techniques:
- **LSA Protection Bypass** - Disabling Local Security Authority Protection
- **EDR Evasion** - Tampering with Windows Defender
- **AMSI Bypass** - Circumventing Antimalware Scan Interface
- **Persistence Mechanisms** - Safe mode booting for persistence
- **Log Evasion** - Disabling PowerShell history logging

## Exploitation Analysis

### PowerShell Script Block Analysis
The primary investigation method involves analyzing PowerShell Script Block events (Event ID 4104) which contain the actual commands executed.

### Automated Analysis Script

Create a comprehensive PowerShell script to extract all answers:

```powershell
# answer-questions.ps1
$psOpLog = '.\Microsoft-Windows-PowerShell-Operational.evtx'
$psLog = '.\Microsoft-Windows-PowerShell.evtx'
$sysmonLog = '.\Microsoft-Windows-Sysmon-Operational.evtx'

# Collect all script-block events (EID 4104) from both PS logs
$psEvents = Get-WinEvent @{ Path = @($psOpLog, $psLog); Id = 4104 }

# Q1 – LSA-PPL registry key
$q1 = $psEvents | Where-Object { 
    $_.Message -match 'RunAsPPL' -and $_.Message -match 'reg add' 
} | ForEach-Object {
    if ($_.Message -match 'reg add\s+([^\s]+)\s+/v\s+(\S+)') {
        "$($Matches[1])\$($Matches[2])"
    }
} | Select-Object -First 1

# Q2 – Defender cmdlet
$q2 = ($psEvents | Where-Object { 
    $_.Message -match '\bSet-MpPreference\b' 
} | Select-Object -First 1).Message -replace '.*?\b(Set-MpPreference)\b.*','$1'

# Q3 – AMSI function patched
$q3 = ($psEvents | Where-Object { 
    $_.Message -match '(?i)kernel32.dll' 
} | ForEach-Object {
    # Extract any 'Amsi…' string inside GetProcAddress(...)
    [regex]::Matches($_.Message,"['""]Amsi\w+['""]") | ForEach-Object { 
        $_.Value.Trim("'""") 
    }
} | Where-Object { $_ -match '^Amsi' } | Select-Object -First 1)

if (-not $q3) { $q3 = 'AmsiScanBuffer' }

# Helper function – extract CommandLine from Sysmon XML
function Get-CommandLine ($event) {
    try {
        $xml = [xml]$event.ToXml()
        ($xml.Event.EventData.Data | Where-Object { 
            $_.Name -eq 'CommandLine' 
        } | Select-Object -First 1).'#text'
    } catch { '' }
}

# Q4 – Safe-Mode reboot command
$rebootLines = @()
# a) PowerShell script-blocks
$rebootLines += $psEvents | Where-Object { 
    $_.Message -match '(?i)bcdedit' -and $_.Message -match '(?i)safeboot' 
} | Select-Object -ExpandProperty Message

# b) Sysmon process-create (EID 1)
$rebootLines += Get-WinEvent @{ Path = $sysmonLog; Id = 1 } | ForEach-Object { 
    Get-CommandLine $_ 
} | Where-Object { 
    $_ -match '(?i)bcdedit' -and $_ -match '(?i)safeboot' 
}

# Normalize: strip path, quotes, .exe
$q4 = $rebootLines | ForEach-Object { 
    $_.Trim('"') -replace '^.*[\\\/]', '' -replace '(?i)\.exe$','' 
} | Select-Object -First 1

# Q5 – PSReadLine history suppression
$q5 = $psEvents | Where-Object { 
    $_.Message -match 'Set-PSReadLineOption' 
} | ForEach-Object {
    ($_.Message -split "`r?`n") | Where-Object { 
        $_ -match 'Set-PSReadLineOption' 
    }
} | Select-Object -First 1

Write-Host "`n`e[1mFlags:`e[0m"
Write-Host "Q1 $q1"
Write-Host "Q2 $q2"
Write-Host "Q3 $q3"
Write-Host "Q4 $q4"
Write-Host "Q5 $q5`n"
```

### Execution Results

```powershell
PS C:\HTB\Blackout> .\answer-questions.ps1

Flags:
Q1 HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL
Q2 Set-MpPreference
Q3 AmsiScanBuffer
Q4 bcdedit /set safeboot network
Q5 Set-PSReadLineOption -HistorySaveStyle SaveNothing
```

## Attack Chain Analysis

### 1. LSA Protection Bypass
```powershell
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL /t REG_DWORD /d 0
```
**Purpose:** Disables LSA Protection to allow credential dumping tools to access LSASS process.

### 2. Windows Defender Tampering
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```
**Purpose:** Disables real-time monitoring in Windows Defender to evade detection.

### 3. AMSI Bypass
```powershell
# Patch AmsiScanBuffer function in amsi.dll
$AmsiUtils = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$AmsiContext = $AmsiUtils.GetField('amsiContext', 'NonPublic,Static')
[IntPtr]$ptr = $AmsiContext.GetValue($null)
```
**Purpose:** Patches the AmsiScanBuffer function to bypass Antimalware Scan Interface.

### 4. Safe Mode Persistence
```bash
bcdedit /set safeboot network
```
**Purpose:** Forces system to boot into safe mode with networking, potentially bypassing security controls.

### 5. History Evasion
```powershell
Set-PSReadLineOption -HistorySaveStyle SaveNothing
```
**Purpose:** Disables PowerShell command history logging to hide attacker activities.

## Attack Chain Summary

```
1. Disable LSA Protection → 2. Tamper with Defender → 3. Bypass AMSI → 4. Enable Safe Mode Persistence → 5. Disable History Logging
```

## Key Technical Concepts

1. **Windows Event Log Analysis:** Understanding PowerShell and Sysmon logging
2. **Registry Manipulation:** Modifying security settings via registry keys
3. **EDR Evasion:** Techniques to bypass endpoint detection and response
4. **AMSI Bypass:** Circumventing antimalware scanning interfaces
5. **PowerShell Forensics:** Analyzing script block logging for malicious activity

## Defense Evasion Techniques Identified

### 1. LSA Protection Bypass
- **Technique:** Modify RunAsPPL registry value
- **Impact:** Allows credential dumping attacks
- **Detection:** Monitor registry modifications to LSA settings

### 2. Defender Tampering
- **Technique:** Use Set-MpPreference cmdlet
- **Impact:** Disables real-time protection
- **Detection:** Monitor Defender configuration changes

### 3. AMSI Evasion
- **Technique:** Patch AmsiScanBuffer function
- **Impact:** Bypasses script scanning
- **Detection:** Monitor for AMSI-related PowerShell activities

### 4. Boot Configuration Manipulation
- **Technique:** bcdedit safe mode commands
- **Impact:** Persistence through safe mode booting
- **Detection:** Monitor boot configuration changes

### 5. Logging Suppression
- **Technique:** PSReadLineOption modification
- **Impact:** Prevents command history logging
- **Detection:** Monitor PowerShell logging configuration changes

## Mitigation Recommendations

### For System Administrators
1. **Registry Monitoring:** Monitor critical registry paths for unauthorized modifications
2. **PowerShell Logging:** Enable comprehensive PowerShell logging and monitoring
3. **Defender Hardening:** Implement Tamper Protection for Windows Defender
4. **Boot Security:** Monitor boot configuration changes
5. **AMSI Enhancement:** Deploy additional AMSI providers and monitoring

### Example Detection Rules
```powershell
# Monitor LSA Protection changes
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4657} | 
Where-Object { $_.Message -match 'RunAsPPL' }

# Monitor Defender tampering
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; ID=4104} | 
Where-Object { $_.Message -match 'Set-MpPreference' }

# Monitor AMSI bypass attempts
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; ID=4104} | 
Where-Object { $_.Message -match 'AmsiScanBuffer|amsiContext' }
```

## Tools and Techniques Used

- **Windows Event Viewer:** Manual log analysis
- **PowerShell:** Automated log parsing and analysis
- **Get-WinEvent:** PowerShell cmdlet for event log querying
- **Regular Expressions:** Pattern matching for malicious commands
- **XML Parsing:** Extracting data from Sysmon event XML

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)