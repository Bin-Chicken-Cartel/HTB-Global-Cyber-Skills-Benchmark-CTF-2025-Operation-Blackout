# Ghost Thread - CTF Write-Up

## Challenge Overview

**Challenge Name:** Ghost Thread  
**Category:** Forensics / Malware Analysis  
**Difficulty:** easy

**Challenge Description:**
> Byte Doctor suspects a stealthy injector used a Thread-Local-Storage-based callback to drop shellcode into a legitimate process before its `main()` ever ran. By reviewing the provided API Monitor dumps (not a live capture) and a quick static check in IDA Free, we'll answer: What injection technique was used? Which APIs enumerate processes, locate and open the target? What is the target process name and its PID? How large is the shellcode? Which API executes it? Which API kills the injector pre-`main()`?

## Initial Reconnaissance

### Provided Artifacts
- **API Monitor v2** (.apmx log file)
- **Binary Sample** for static analysis with IDA Free
- **Investigation Goals:** Answer 7 specific questions about the injection technique

### Tools Required
- **API Monitor v2** for dynamic analysis log review
- **IDA Free** for static analysis and TLS callback examination
- **Hex viewer** for binary data analysis

## Vulnerability Analysis

### Thread Local Storage (TLS) Injection Technique
TLS injection (MITRE T1055.005) is a sophisticated process injection method that:
1. **Executes Before Main:** TLS callbacks run before the main() function
2. **Stealth Operation:** Difficult to detect as it uses legitimate Windows mechanisms
3. **Early Execution:** Gains control of target process immediately upon load
4. **Self-Termination:** Injector can exit before main() execution begins

## Exploitation Analysis

### Question 1: Injection Technique Identification

**Method:** Static Analysis with IDA Free
1. Open the binary in IDA Free
2. Navigate to TLS callback section (`TlsCallback_0`)
3. Examine calls to helper functions like `ConfigThreadLocal`
4. Check `.rdata` section for `TlsCallbacks` references

**Evidence:** TLS callback structure and references confirm the technique.

**Answer:** **Thread Local Storage**

### Question 2: Process Enumeration API

**Method:** API Monitor Log Analysis
```bash
# Search API Monitor dump for process enumeration
Find → CreateToolhelp32Snapshot
```

**API Call Evidence:**
```
API Monitor: CreateToolhelp32Snapshot
CreateToolhelp32Snapshot(
    dwFlags = TH32CS_SNAPPROCESS | TH32CS_SNAPTHREAD,
    th32ProcessID = 0
) → HANDLE = 0x000000000000029C
```

**Answer:** **CreateToolhelp32Snapshot**

### Question 3: Target Process Location API

**Method:** API Monitor Log Analysis
```bash
# Search for process iteration APIs
Find → Process32Next
```

**API Call Evidence:**
```
API Monitor: Process32Next
Process32Next(
    hSnapshot = 0x000000000000029C,
    lppe = 0x000000CEE76FEF88
) → TRUE
```

After `lstrcmpiA` matching `"Notepad.exe"`, the Parameters pane shows the `PROCESSENTRY32` structure with target process information.

**Memory Analysis:**
- Offset 0x1C contains UTF-16LE string: `"Notepad.exe"`

**Answer:** **notepad.exe**

### Question 4: Target Process ID Extraction

**Method:** API Monitor Memory Dump Analysis

**PROCESSENTRY32 Structure Analysis:**
```
PROCESSENTRY32 Memory Dump:
Offset 0x08: 90 3F 00 00  ← th32ProcessID (little-endian)
Offset 0x1C: "Notepad.exe" (UTF-16LE)
```

**Calculation:**
- Bytes: `90 3F 00 00`
- Little-endian conversion: `0x3F90`
- Decimal value: **16224**

**Answer:** **16224**

### Question 5: Shellcode Size Determination

**Method:** API Monitor WriteProcessMemory Analysis
```bash
# Search for memory writing operations
Find → WriteProcessMemory
```

**API Call Evidence:**
```
API Monitor: WriteProcessMemory
WriteProcessMemory(
    hProcess = 0x0000000000000268,
    lpBaseAddress = 0x000000206583d000,
    lpBuffer = 0x000000c3e76f320,
    nSize = 0x000001FF,  ← Shellcode size
    lpNumberOfBytesWritten = NULL
) → TRUE
```

**Size Calculation:**
- Hex value: `0x1FF`
- Decimal conversion: **511 bytes**

**Answer:** **511**

### Question 6: Shellcode Execution API

**Method:** API Monitor Thread Creation Analysis
```bash
# Search for remote thread creation
Find → CreateRemoteThread
```

**API Call Evidence:**
```
API Monitor: CreateRemoteThread
CreateRemoteThread(
    hProcess = 0x0000000000000268,
    lpThreadAttributes = NULL,
    dwStackSize = 0,
    lpStartAddress = 0x000000206583d000,  ← Shellcode entry point
    lpParameter = NULL,
    dwCreationFlags = 0,
    lpThreadId = NULL
) → THREAD HANDLE = 0x00000000000002A4
```

**Answer:** **CreateRemoteThread**

### Question 7: Pre-Main Termination API

**Method:** API Monitor Process Termination Analysis
```bash
# Search for process exit APIs
Find → ExitProcess
```

**API Call Evidence:**
```
API Monitor: ExitProcess
ExitProcess(
    uExitCode = 0
) → NO RETURN
```

This call occurs in the TLS callback, terminating the injector before `main()` execution.

**Answer:** **ExitProcess**

## Attack Chain Summary

```
1. TLS Callback Execution → 2. Process Enumeration → 3. Target Location → 4. Process Opening → 5. Shellcode Injection → 6. Remote Execution → 7. Injector Termination
```

## Complete Investigation Results

| Question | API/Answer | Purpose |
|----------|------------|---------|
| Q1 | Thread Local Storage | Injection technique used |
| Q2 | CreateToolhelp32Snapshot | Process enumeration API |
| Q3 | notepad.exe | Target process name |
| Q4 | 16224 | Target process PID |
| Q5 | 511 | Shellcode size in bytes |
| Q6 | CreateRemoteThread | Shellcode execution API |
| Q7 | ExitProcess | Pre-main termination API |

## Key Technical Concepts

1. **Thread Local Storage (TLS):** Windows mechanism for thread-specific data storage
2. **API Monitoring:** Dynamic analysis technique for tracking system calls
3. **Process Injection:** Technique for executing code in target process memory
4. **Static Analysis:** Examination of binary without execution
5. **Memory Forensics:** Analysis of process memory structures

## TLS Injection Attack Flow

### 1. TLS Callback Registration
```c
// Typical TLS callback structure
PIMAGE_TLS_CALLBACK TlsCallbacks[] = {
    TlsCallback_0,  // Our malicious callback
    NULL
};
```

### 2. Process Enumeration
```c
// Enumerate running processes
HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
PROCESSENTRY32 pe32;
Process32First(hSnapshot, &pe32);
```

### 3. Target Selection
```c
// Find specific target process
do {
    if (lstrcmpiA(pe32.szExeFile, "notepad.exe") == 0) {
        targetPID = pe32.th32ProcessID;
        break;
    }
} while (Process32Next(hSnapshot, &pe32));
```

### 4. Memory Injection
```c
// Inject shellcode into target
HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPID);
LPVOID pRemoteMemory = VirtualAllocEx(hProcess, NULL, shellcodeSize, 
                                     MEM_COMMIT, PAGE_EXECUTE_READWRITE);
WriteProcessMemory(hProcess, pRemoteMemory, shellcode, shellcodeSize, NULL);
```

### 5. Remote Execution
```c
// Execute injected code
HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, 
                                  (LPTHREAD_START_ROUTINE)pRemoteMemory,
                                  NULL, 0, NULL);
```

### 6. Self-Termination
```c
// Exit before main() runs
ExitProcess(0);
```

## Detection and Mitigation

### Detection Strategies
1. **TLS Callback Monitoring:** Monitor for unusual TLS callback registrations
2. **API Sequence Analysis:** Look for process enumeration followed by injection APIs
3. **Memory Pattern Analysis:** Detect shellcode patterns in process memory
4. **Behavioral Analysis:** Monitor for processes exiting before main() execution

### Prevention Techniques
1. **Code Signing:** Verify binary authenticity and integrity
2. **Application Sandboxing:** Isolate applications from system resources
3. **API Hooking:** Monitor and control dangerous API calls
4. **Memory Protection:** Implement DEP/ASLR and other memory protections

### Example Detection Rule
```python
def detect_tls_injection():
    """Detect potential TLS injection patterns"""
    
    api_sequence = [
        "CreateToolhelp32Snapshot",
        "Process32First",
        "Process32Next", 
        "OpenProcess",
        "VirtualAllocEx",
        "WriteProcessMemory",
        "CreateRemoteThread",
        "ExitProcess"
    ]
    
    # Monitor for this API sequence in short timeframe
    if detect_api_sequence(api_sequence, timeframe=5000):  # 5 seconds
        return "TLS Injection Detected"
```

## Tools and Techniques Used

- **API Monitor v2:** Dynamic analysis and API call monitoring
- **IDA Free:** Static binary analysis and reverse engineering
- **Process Memory Analysis:** Understanding Windows process structures
- **TLS Callback Analysis:** Examining thread-local storage mechanisms
- **Hex Data Analysis:** Converting binary data to meaningful information

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069) & ## Write-Up Credit: [SR20DET](https://ctf.hackthebox.com/user/profile/605510)