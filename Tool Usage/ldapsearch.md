# ldapsearch - LDAP Enumeration Cheatsheet (OSCP+)

ldapsearch queries Active Directory via LDAP protocol (port 389/636). It's built into Kali and is one of the first tools you use to enumerate users, groups, computers, and domain info in an AD environment.

---

### Find the Base DN (Domain Name)

Before querying, you need the base DN. If the domain is `domain.local`, the base DN is `DC=domain,DC=local`.

```bash
# Auto-discover base DN with anonymous bind
ldapsearch -x -H ldap://10.10.10.5 -s base namingcontexts
```

This returns something like:

```
namingContexts: DC=domain,DC=local
```

---

### Anonymous Enumeration (No Credentials)

```bash
# Check if anonymous bind is allowed
ldapsearch -x -H ldap://10.10.10.5 -b "DC=domain,DC=local"

# Get domain info
ldapsearch -x -H ldap://10.10.10.5 -b "DC=domain,DC=local" -s base

# Enumerate users
ldapsearch -x -H ldap://10.10.10.5 -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName

# Enumerate groups
ldapsearch -x -H ldap://10.10.10.5 -b "DC=domain,DC=local" "(objectClass=group)" cn
```

---

### Authenticated Enumeration

#### With Password

```bash
# Basic authenticated query
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local"

# Alternative bind format
ldapsearch -x -H ldap://10.10.10.5 -D "CN=username,DC=domain,DC=local" -w 'password' -b "DC=domain,DC=local"
```

#### With Password Prompt (Safer)

```bash
# -W prompts for password instead of showing it in command history
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -W -b "DC=domain,DC=local"
```

---

### Enumerate Users

#### All Users

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName
```

#### All Users with Details

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName description memberOf
```

#### Extract Just Usernames (Clean Output)

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName | grep sAMAccountName | awk '{print $2}'
```

#### Find Users with Descriptions (Often Contain Passwords)

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName description | grep -A1 description
```

---

### Enumerate Groups

#### All Groups

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=group)" cn
```

#### Domain Admins Members

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(cn=Domain Admins)" member
```

#### All Admin Groups

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(cn=*admin*)" cn member
```

#### Remote Management Users (Who Can WinRM)

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(cn=Remote Management Users)" member
```

---

### Enumerate Computers

```bash
# All computers in the domain
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=computer)" cn operatingSystem dNSHostName

# Just computer names
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=computer)" dNSHostName | grep dNSHostName
```

---

### Find Kerberoastable Accounts (SPN Set)

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName servicePrincipalName
```

---

### Find AS-REP Roastable Accounts (Preauth Disabled)

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" sAMAccountName
```

---

### Find Delegation

#### Unconstrained Delegation

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(userAccountControl:1.2.840.113556.1.4.803:=524288)" sAMAccountName dNSHostName
```

#### Constrained Delegation

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(msDS-AllowedToDelegateTo=*)" sAMAccountName msDS-AllowedToDelegateTo
```

---

### Find Password Policy

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=domain)" minPwdLength maxPwdAge lockoutThreshold lockoutDuration
```

---

### Find LAPS Passwords (If Readable)

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=computer)" cn ms-Mcs-AdmPwd
```

---

### Find GMSA Passwords

```bash
ldapsearch -x -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=msDS-GroupManagedServiceAccount)" sAMAccountName msDS-ManagedPassword
```

---

### LDAPS (Secure LDAP - Port 636)

```bash
# Connect via LDAPS
ldapsearch -x -H ldaps://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local"

# If certificate errors
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local"
```

---

### Useful Flags Reference

| Flag | Meaning |
|---|---|
| `-x` | Simple authentication (always use this) |
| `-H` | LDAP server URL |
| `-D` | Bind DN (username) |
| `-w` | Password |
| `-W` | Prompt for password |
| `-b` | Base DN (where to search) |
| `-s` | Scope: `base`, `one`, `sub` (default: sub) |
| `-LLL` | Clean output (no comments, no version) |

#### Clean Output

```bash
# Add -LLL for cleaner results (removes junk)
ldapsearch -x -LLL -H ldap://10.10.10.5 -D "username@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName
```

---

### OSCP+ Exam Workflow

```bash
# Step 1: Find base DN
ldapsearch -x -H ldap://DC_IP -s base namingcontexts

# Step 2: Try anonymous bind
ldapsearch -x -H ldap://DC_IP -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName

# Step 3: Authenticated - dump all users
ldapsearch -x -LLL -H ldap://DC_IP -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName description memberOf | grep -E "sAMAccountName|description"

# Step 4: Find Domain Admins
ldapsearch -x -LLL -H ldap://DC_IP -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" "(cn=Domain Admins)" member

# Step 5: Find Kerberoastable
ldapsearch -x -LLL -H ldap://DC_IP -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName

# Step 6: Find AS-REP Roastable
ldapsearch -x -LLL -H ldap://DC_IP -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" sAMAccountName

# Step 7: Check LAPS
ldapsearch -x -LLL -H ldap://DC_IP -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" "(objectClass=computer)" cn ms-Mcs-AdmPwd
```

---

### Pro Tips

- Always check user **descriptions** — admins often put passwords there
- Use `-LLL` flag for clean output without junk headers
- If anonymous bind works, you get free enumeration without any creds
- Pipe output through `grep` and `awk` to get clean lists for spraying
- ldapsearch output can be huge — redirect to a file: `ldapsearch ... > output.txt`
- For faster enumeration, use `nxc ldap` or `windapsearch` instead — same data, easier syntax
- The LDAP filter for AS-REP Roasting (`4194304`) checks the DONT_REQUIRE_PREAUTH flag
- Always check for LAPS — if readable, you get local admin passwords for free