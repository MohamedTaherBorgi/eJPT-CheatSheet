## 🗺️ SCAN FLOW

```
Host Discovery → Port Scan (Fast) → Full Port Scan → Service/Version → NSE Scripts → Service-Specific Enum
```

---

## 1️⃣ HOST DISCOVERY

```bash
# ARP (LAN — most reliable)
netdiscover -r [TARGET_IP]/24
netdiscover -i eth0 -r [TARGET_IP]/24
arp-scan --localnet
arp-scan -I eth0 [TARGET_IP]/24

# ICMP sweep
nmap -sn [TARGET_IP]/24
nmap -sn [TARGET_IP]/24 -oG - | grep "Up" | awk '{print $2}'
fping -a -g [TARGET_IP]/24 2>/dev/null

# No ping (when ICMP blocked)
nmap -Pn [TARGET_IP]
```

---

## 2️⃣ NMAP — Heavy Hitter ⭐⭐⭐

### Fast Initial Scan

```bash
# Top 1000 ports — start here
nmap -sV -sC [TARGET_IP]
nmap -sV -sC [TARGET_IP] -oN initial.txt

# Top 1000 + OS detection
nmap -A [TARGET_IP] -oN initial.txt
```

### Full Port Scan

```bash
# All 65535 ports
nmap -p- [TARGET_IP]
nmap -p- --min-rate 5000 [TARGET_IP] -oN full.txt

# Full + service detection (after finding open ports)
nmap -p [PORT1],[PORT2],[PORT3] -sV -sC [TARGET_IP] -oN targeted.txt

# Aggressive all-ports
nmap -A -p- --min-rate 3000 [TARGET_IP] -oN full_aggressive.txt
```

### Targeted / Specific Port Scans

```bash
nmap -p 21,22,23,25,53,80,110,135,139,143,443,445,3306,3389,8080,8443 -sV -sC [TARGET_IP]
nmap -p 80,443,8080,8443,8888 -sV -sC [TARGET_IP]    # web ports only
nmap -p 445,139 -sV -sC [TARGET_IP]                   # SMB only
nmap -p 3306 -sV -sC [TARGET_IP]                      # MySQL only
nmap -p 3389 -sV -sC [TARGET_IP]                      # RDP only
```

### UDP Scan

```bash
nmap -sU [TARGET_IP]
nmap -sU -p 53,67,68,69,111,123,137,138,161,162,500 [TARGET_IP]
nmap -sU --top-ports 20 [TARGET_IP]
```

### NSE Scripts — Heavy Hitter ⭐

```bash
# Run all default + safe scripts
nmap -sC [TARGET_IP]

# Vuln scan
nmap --script vuln [TARGET_IP]
nmap --script vuln -p [PORT] [TARGET_IP]

# Auth brute
nmap --script brute [TARGET_IP]

# SMB scripts
nmap --script smb-vuln* -p 445 [TARGET_IP]
nmap --script smb-vuln-ms17-010 -p 445 [TARGET_IP]
nmap --script smb-enum-shares,smb-enum-users -p 445 [TARGET_IP]
nmap --script smb-os-discovery -p 445 [TARGET_IP]

# HTTP scripts
nmap --script http-title,http-headers,http-methods [TARGET_IP]
nmap --script http-enum -p 80,443,8080 [TARGET_IP]
nmap --script http-shellshock --script-args uri=/cgi-bin/test.cgi [TARGET_IP]

# FTP scripts
nmap --script ftp-anon,ftp-brute,ftp-vsftpd-backdoor -p 21 [TARGET_IP]

# DNS zone transfer
nmap -p 53 --script dns-zone-transfer --script-args dns-zone-transfer.domain=[DOMAIN] [TARGET_IP]

# MySQL
nmap --script mysql-info,mysql-empty-password,mysql-databases -p 3306 [TARGET_IP]

# SSH
nmap --script ssh-auth-methods,ssh-brute -p 22 [TARGET_IP]
```

### Output Formats

```bash
nmap [TARGET_IP] -oN output.txt      # normal
nmap [TARGET_IP] -oX output.xml      # XML
nmap [TARGET_IP] -oG output.gnmap    # grepable
nmap [TARGET_IP] -oA output          # ALL three formats
```

### Firewall / IDS Evasion

```bash
nmap -f [TARGET_IP]                          # fragment packets
nmap -D RND:10 [TARGET_IP]                   # decoy scan
nmap --source-port 53 [TARGET_IP]            # spoof source port
nmap -sS [TARGET_IP]                         # SYN scan (stealth)
nmap --scan-delay 1s [TARGET_IP]             # slow scan
nmap -Pn --disable-arp-ping [TARGET_IP]      # skip ping probes
```

---

## 3️⃣ FAST SCANNERS

### `masscan`

```bash
# Full port range — very fast
masscan -p1-65535 [TARGET_IP] --rate=1000
masscan -p1-65535 [TARGET_IP]/24 --rate=5000 -oL masscan.txt

# Specific ports
masscan -p 80,443,8080,445,22,3389 [TARGET_IP]/24 --rate=1000

# Output then feed to nmap
masscan -p1-65535 [TARGET_IP] --rate=1000 -oL ports.txt
# Extract ports:
grep "open" ports.txt | awk '{print $3}' | tr '\n' ',' | sed 's/,$//'
nmap -p [PORTS_FROM_MASSCAN] -sV -sC [TARGET_IP]
```

### `rustscan`

```bash
rustscan -a [TARGET_IP]
rustscan -a [TARGET_IP] -- -sV -sC                     # pipe to nmap
rustscan -a [TARGET_IP] -p 1-65535 -- -sV -sC -oN scan.txt
rustscan -a [TARGET_IP]/24                             # subnet
rustscan -a [TARGET_IP] --ulimit 5000                  # increase speed
```

---

## 4️⃣ SMB ENUMERATION (Port 139/445)

### `enum4linux` — Heavy Hitter ⭐

```bash
# Full enum (null session)
enum4linux -a [TARGET_IP]

# Targeted
enum4linux -U [TARGET_IP]      # users
enum4linux -S [TARGET_IP]      # shares
enum4linux -G [TARGET_IP]      # groups
enum4linux -P [TARGET_IP]      # password policy
enum4linux -o [TARGET_IP]      # OS info
enum4linux -n [TARGET_IP]      # NetBIOS names

# With credentials
enum4linux -a -u [USER] -p [PASS] [TARGET_IP]

# enum4linux-ng (modern version)
enum4linux-ng [TARGET_IP] -A
enum4linux-ng [TARGET_IP] -A -u [USER] -p [PASS]
```

### `smbclient`

```bash
# List shares (null session)
smbclient -L //[TARGET_IP]
smbclient -L //[TARGET_IP] -N              # no password
smbclient -L //[TARGET_IP] -U [USER]

# Connect to share
smbclient //[TARGET_IP]/[SHARE]
smbclient //[TARGET_IP]/[SHARE] -N         # no password
smbclient //[TARGET_IP]/[SHARE] -U [USER]%[PASS]

# Inside smbclient shell:
ls
get [FILE]
put [FILE]
mget *
recurse ON
prompt OFF
mget *
```

### `crackmapexec` (SMB) — Heavy Hitter ⭐

```bash
# Host info
crackmapexec smb [TARGET_IP]
crackmapexec smb [TARGET_IP]/24

# Null / Guest enum
crackmapexec smb [TARGET_IP] --shares
crackmapexec smb [TARGET_IP] -u '' -p '' --shares
crackmapexec smb [TARGET_IP] -u 'guest' -p '' --shares

# Authenticated enum
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] --shares
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] --users
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] --groups
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] --sessions
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] --loggedon-users

# Spray passwords across subnet
crackmapexec smb [TARGET_IP]/24 -u [USER] -p [PASS]

# Check for MS17-010
crackmapexec smb [TARGET_IP] -u '' -p '' -M ms17-010

# Execute command
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] -x "whoami"
crackmapexec smb [TARGET_IP] -u [USER] -p [PASS] -X "Get-ComputerInfo"
```

### `smbmap` Permission enumeration

```bash
smbmap -H [TARGET_IP]
smbmap -H [TARGET_IP] -u [USER] -p [PASS]
smbmap -H [TARGET_IP] -u [USER] -p [PASS] -R [SHARE]   # recursive
smbmap -H [TARGET_IP] -u '' -p ''                       # null session
```

---

## 5️⃣ SNMP ENUMERATION (Port 161 UDP)

**SNMP (Simple Network Management Protocol)** is the "standard language" used by network devices (routers, switches, servers, printers, etc.) to communicate status information to a central management system.
### The Attack Flow

1. **Discovery:** Use `nmap -sU -p 161 [IP]` to see if the SNMP port (UDP 161) is open.
    
2. **Brute-Force:** Use `onesixtyone` with a wordlist to find the "Community String."
    
3. **Enumeration:** Use `snmp-check` or `snmpwalk` to dump the entire system configuration.
### `onesixtyone`

```bash
onesixtyone [TARGET_IP] public
onesixtyone [TARGET_IP] private
onesixtyone [TARGET_IP] manager
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt [TARGET_IP]
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt -i hosts.txt
```

| **Version** | **Security Level** | **Red Team Context**                                                                                                                           |
| ----------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **v1**      | **None**           | Uses "Community Strings" (passwords) sent in **cleartext**. Extremely vulnerable to sniffing.                                                  |
| **v2c**     | **None**           | Same as v1 but adds "Bulk" transfers. Still uses cleartext Community Strings. **This is the most common version you will find misconfigured.** |
| **v3**      | **High**           | Adds encryption and actual Username/Password authentication. Much harder to exploit.                                                           |
### `snmpwalk`

```bash
# Full walk
snmpwalk -c public -v1 [TARGET_IP]
snmpwalk -c public -v2c [TARGET_IP]

# Targeted OIDs
snmpwalk -c public -v1 [TARGET_IP] 1.3.6.1.2.1.25.4.2.1.2   # running processes
snmpwalk -c public -v1 [TARGET_IP] 1.3.6.1.2.1.25.6.3.1.2   # installed software
snmpwalk -c public -v1 [TARGET_IP] 1.3.6.1.4.1.77.1.2.25    # user accounts
snmpwalk -c public -v1 [TARGET_IP] 1.3.6.1.2.1.6.13.1.3     # open TCP ports
snmpwalk -c public -v1 [TARGET_IP] 1.3.6.1.2.1.25.1.6.0     # system processes
```

## OR
### `snmp-check`

```bash
snmp-check [TARGET_IP]
snmp-check -c public [TARGET_IP]
snmp-check -c private [TARGET_IP] -v 2c
```

---

## 6️⃣ FTP ENUMERATION (Port 21)

```bash
# nmap first
nmap --script ftp-anon,ftp-bounce,ftp-syst,ftp-vsftpd-backdoor -p 21 [TARGET_IP]

# Manual anonymous login
ftp [TARGET_IP]
# Username: anonymous  Password: (blank or anything@email.com)

# Check anonymous
curl -v ftp://[TARGET_IP]/
curl ftp://[TARGET_IP]/ --user anonymous:
curl ftp://[TARGET_IP]/[FILE] -o file.txt

# Brute force
hydra -l [USER] -P /usr/share/wordlists/rockyou.txt ftp://[TARGET_IP]
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://[TARGET_IP]
medusa -h [TARGET_IP] -u [USER] -P /usr/share/wordlists/rockyou.txt -M ftp
```

---

## 7️⃣ SSH ENUMERATION (Port 22)

```bash
# nmap
nmap --script ssh-auth-methods -p 22 [TARGET_IP]
nmap --script ssh-hostkey -p 22 [TARGET_IP]

# Manual banner grab
nc -nv [TARGET_IP] 22
ssh [USER]@[TARGET_IP]

# Brute force
hydra -l [USER] -P /usr/share/wordlists/rockyou.txt ssh://[TARGET_IP]
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://[TARGET_IP] -t 4
crackmapexec ssh [TARGET_IP] -u [USER] -p /usr/share/wordlists/rockyou.txt
medusa -h [TARGET_IP] -u [USER] -P /usr/share/wordlists/rockyou.txt -M ssh
```

---

## 8️⃣ RDP ENUMERATION (Port 3389)

```bash
# nmap
nmap --script rdp-enum-encryption,rdp-vuln-ms12-020 -p 3389 [TARGET_IP]

# Check if open
nmap -p 3389 --open [TARGET_IP]/24

# Brute force
hydra -l [USER] -P /usr/share/wordlists/rockyou.txt rdp://[TARGET_IP]
crackmapexec rdp [TARGET_IP] -u [USER] -p [PASS]
crackmapexec rdp [TARGET_IP] -u [USER] -p /usr/share/wordlists/rockyou.txt

# Connect
xfreerdp /u:[USER] /p:[PASS] /v:[TARGET_IP]
xfreerdp /u:[USER] /p:[PASS] /v:[TARGET_IP] /cert-ignore
rdesktop -u [USER] -p [PASS] [TARGET_IP]
```

---

## 9️⃣ SMTP ENUMERATION (Port 25)


```bash
# nmap
nmap --script smtp-enum-users,smtp-commands,smtp-vuln* -p 25 [TARGET_IP]

# Manual
nc -nv [TARGET_IP] 25
EHLO test
VRFY [USER]         # verify user exists
EXPN [USER]         # expand mailing list

# smtp-user-enum
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t [TARGET_IP]
smtp-user-enum -M RCPT -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t [TARGET_IP]
smtp-user-enum -M EXPN -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t [TARGET_IP]
```

#### SMTP is for **sending**, POP3/IMAP are for **reading**.

---

## 🔟 MYSQL ENUMERATION (Port 3306)

```bash
# nmap
nmap --script mysql-info,mysql-empty-password,mysql-auth-bypass,mysql-databases,mysql-users -p 3306 [TARGET_IP]

# Manual login
mysql -h [TARGET_IP] -u root
mysql -h [TARGET_IP] -u root -p
mysql -h [TARGET_IP] -u [USER] -p[PASS]          # no space before pass

# Inside MySQL:
show databases;
use [DATABASE];
show tables;
select * from [TABLE];
select user,password from mysql.user;
select @@version;
select @@datadir;
```

---

## 1️⃣1️⃣ WEB ENUMERATION

### `gobuster` — Heavy Hitter ⭐

```bash
# Directory brute force
gobuster dir -u http://[TARGET_IP] -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
gobuster dir -u http://[TARGET_IP] -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,zip
gobuster dir -u http://[TARGET_IP] -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,asp,aspx,txt -o gobuster.txt

# HTTPS (ignore cert)
gobuster dir -u https://[TARGET_IP] -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k

# DNS subdomain brute
gobuster dns -d [TARGET_DOMAIN] -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# VHost brute
gobuster vhost -u http://[TARGET_DOMAIN] -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### `ffuf` — Heavy Hitter ⭐

```bash
# Directory
ffuf -u http://[TARGET_IP]/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
ffuf -u http://[TARGET_IP]/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.txt,.html,.bak

# Filter by size / status
ffuf -u http://[TARGET_IP]/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fc 404
ffuf -u http://[TARGET_IP]/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs [SIZE]

# Subdomain / vhost
ffuf -u http://[TARGET_DOMAIN] -H "Host: FUZZ.[TARGET_DOMAIN]" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 302

# GET param fuzzing
ffuf -u http://[TARGET_IP]/page.php?FUZZ=value -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# POST body fuzzing
ffuf -u http://[TARGET_IP]/login.php -X POST -d "username=FUZZ&password=test" -w /usr/share/seclists/Usernames/top-usernames-shortlist.txt -fc 200

# Output
ffuf -u http://[TARGET_IP]/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ffuf.json -of json
```

### `dirsearch`

```bash
dirsearch -u http://[TARGET_IP]
dirsearch -u http://[TARGET_IP] -e php,asp,aspx,txt,bak,zip
dirsearch -u http://[TARGET_IP] -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
dirsearch -u http://[TARGET_IP] -e php,txt -o dirsearch.txt
```

### `nikto`

```bash
nikto -h http://[TARGET_IP]
nikto -h http://[TARGET_IP] -p 8080
nikto -h http://[TARGET_IP] -ssl                      # HTTPS
nikto -h http://[TARGET_IP] -o nikto.txt -Format txt
nikto -h http://[TARGET_IP] -Tuning 9                 # SQL injection checks
```

### `wfuzz`

```bash
# Directory
wfuzz -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hc 404 http://[TARGET_IP]/FUZZ

# Extension brute
wfuzz -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -z list,php-txt-html --hc 404 http://[TARGET_IP]/FUZZ.FUZ2Z

# POST login brute
wfuzz -c -w /usr/share/wordlists/rockyou.txt -d "username=[USER]&password=FUZZ" --hc 200 http://[TARGET_IP]/login.php

# Header fuzz
wfuzz -c -w wordlist.txt -H "Authorization: FUZZ" http://[TARGET_IP]/api
```

---

## 1️⃣2️⃣ BANNER GRABBING

```bash
# on NON encrypted ports
nc -nv [TARGET_IP] [PORT] # BEST
telnet [TARGET_IP] [PORT]
curl -sv telnet://[TARGET_IP]:[PORT]

# on encrypted ports
openssl s_client -connect [TARGET_IP]:[PORT] -quiet

# Multiple ports
for port in 21 22 25 80 110 143 443 445 3306 3389 8080; do
  echo "=== Port $port ===" && echo "" | nc -nv -w1 [TARGET_IP] $port 2>&1 | head -5
done
```

---

## ⚡ QUICK REFERENCE — Service → Tool Chain

|Port|Service|Tool Chain|
|---|---|---|
|21|FTP|`nmap --script ftp-anon` → `ftp` anon login → `hydra`|
|22|SSH|`nmap --script ssh-auth-methods` → `hydra`|
|25|SMTP|`nmap --script smtp-enum-users` → `smtp-user-enum`|
|80/443|HTTP/S|`nikto` → `gobuster` → `ffuf` → `whatweb`|
|139/445|SMB|`enum4linux` → `smbclient` → `crackmapexec`|
|161|SNMP|`onesixtyone` → `snmpwalk`|
|3306|MySQL|`nmap --script mysql-*` → `mysql -h`|
|3389|RDP|`nmap --script rdp-*` → `hydra` → `xfreerdp`|
|8080|Alt HTTP|same as 80 chain|

---

## 🧠 EXAM DECISION TREE

```
Open port found?
│
├── 21 (FTP)
│   ├── Try anonymous login first
│   ├── Check nmap ftp-anon script result
│   └── If creds needed → hydra brute
│
├── 22 (SSH)
│   ├── Grab banner → note version (check exploits)
│   └── If creds → hydra / crackmapexec ssh
│
├── 80/443/8080 (Web)
│   ├── nikto → quick vuln check
│   ├── gobuster/ffuf → directory discovery
│   ├── whatweb → fingerprint stack
│   └── Manual browse → login pages, upload forms
│
├── 139/445 (SMB)
│   ├── nmap smb-vuln-ms17-010 first (EternalBlue!)
│   ├── enum4linux -a → null session enum
│   ├── smbclient -L → list shares
│   └── crackmapexec smb → spray creds
│
├── 161 UDP (SNMP)
│   ├── onesixtyone → community string brute
│   └── snmpwalk → dump users, processes
│
└── 3306 (MySQL)
    ├── Try root with no password
    └── nmap mysql-empty-password script
```

---

## 📋 WORDLISTS — Quick Reference

```bash
# General directories
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/big.txt

# DNS / Subdomains
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# Passwords
/usr/share/wordlists/rockyou.txt

# Usernames
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
/usr/share/seclists/Usernames/Names/names.txt

# SNMP community strings
/usr/share/seclists/Discovery/SNMP/snmp.txt
```

---
---
