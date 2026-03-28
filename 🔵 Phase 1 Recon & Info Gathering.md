## 🗺️ RECON FLOW

```
Passive OSINT → DNS Enumeration → Zone Transfer Attempt → Web Fingerprinting → Google Dorks
```

---

## 1️⃣ PASSIVE OSINT

### `whois`

```bash
whois [TARGET_DOMAIN]
whois [TARGET_IP]
```

### `theHarvester` — Heavy Hitter ⭐

```bash
# Email, subdomain, host harvesting
theHarvester -d [TARGET_DOMAIN] -b all
theHarvester -d [TARGET_DOMAIN] -b google,bing,linkedin,dnsdumpster
theHarvester -d [TARGET_DOMAIN] -b google -l 500

# Output to file
theHarvester -d [TARGET_DOMAIN] -b all -f recon_output
```

### `maltego` — Workflow

```
1. New Graph → Entity: Domain → [TARGET_DOMAIN]
2. Right-click → Run Transforms → All Transforms
3. Focus: To IP / To DNS / To Email Address / To Person
```

---

## 2️⃣ DNS ENUMERATION

### `dig` — Heavy Hitter ⭐

```bash
# Basic records
dig [TARGET_DOMAIN]
dig [TARGET_DOMAIN] A
dig [TARGET_DOMAIN] MX
dig [TARGET_DOMAIN] NS
dig [TARGET_DOMAIN] TXT
dig [TARGET_DOMAIN] CNAME
dig [TARGET_DOMAIN] ANY

# Reverse lookup
dig -x [TARGET_IP]

# Query specific nameserver
dig @[NAMESERVER_IP] [TARGET_DOMAIN] ANY

# Zone transfer attempt
dig axfr [TARGET_DOMAIN] @[NAMESERVER_IP]
dig axfr @[TARGET_IP] [TARGET_DOMAIN]
```

### `dnsenum` — Heavy Hitter ⭐

```bash
# Full enum + brute force subdomains
dnsenum [TARGET_DOMAIN]
dnsenum --enum [TARGET_DOMAIN]
dnsenum --dnsserver [NAMESERVER_IP] [TARGET_DOMAIN]

# With subdomain wordlist
dnsenum --enum -f /usr/share/wordlists/dnsmap.txt [TARGET_DOMAIN]
dnsenum --threads 20 -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt [TARGET_DOMAIN]

# Output to XML
dnsenum --enum -o output.xml [TARGET_DOMAIN]
```

### `dnsrecon`

```bash
# Standard enum
dnsrecon -d [TARGET_DOMAIN]
dnsrecon -d [TARGET_DOMAIN] -t std

# Zone transfer
dnsrecon -d [TARGET_DOMAIN] -t axfr

# Brute force subdomains
dnsrecon -d [TARGET_DOMAIN] -t brt -D /usr/share/wordlists/dnsmap.txt
dnsrecon -d [TARGET_DOMAIN] -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t brt

# Reverse range
dnsrecon -r [TARGET_IP]/24 -n [NAMESERVER_IP]

# Output
dnsrecon -d [TARGET_DOMAIN] -t std --csv /tmp/dns.csv
```

### `fierce`

```bash
fierce --domain [TARGET_DOMAIN]
fierce --domain [TARGET_DOMAIN] --subdomains admin mail vpn dev staging
fierce --domain [TARGET_DOMAIN] --wordlist /usr/share/wordlists/dnsmap.txt
```

---

## 3️⃣ ZONE TRANSFER ATTEMPTS

### Full Zone Transfer Workflow

```bash
# Step 1 — Get nameservers
dig NS [TARGET_DOMAIN]
or
host -t ns [TARGET_DOMAIN]

# Step 2 — Attempt zone transfer against each NS
dig axfr [TARGET_DOMAIN] @[NAMESERVER_IP]
or
host -l [TARGET_DOMAIN] [NAMESERVER_IP]
or
fierce --domain [TARGET_DOMAIN]       # auto-attempts AXFR

# Step 3 — nmap script
nmap -p 53 --script dns-zone-transfer --script-args dns-zone-transfer.domain=[TARGET_DOMAIN] [TARGET_IP]
```

---

## 4️⃣ WEB FINGERPRINTING

### `whatweb`

```bash
whatweb [TARGET_IP]
whatweb http://[TARGET_IP]
whatweb -v http://[TARGET_DOMAIN]      # verbose
whatweb -a 3 http://[TARGET_DOMAIN]   # aggression level 3
whatweb -a 3 http://[TARGET_DOMAIN] --log-verbose=whatweb.txt
```

### `wafw00f`

```bash
wafw00f http://[TARGET_DOMAIN]
wafw00f -a http://[TARGET_DOMAIN]     # test all WAFs
```

### `curl` Headers

```bash
curl -I http://[TARGET_IP]
curl -I -L http://[TARGET_IP]                    # follow redirects
curl -s http://[TARGET_IP]/robots.txt
curl -s http://[TARGET_IP]/sitemap.xml
```

---

## 5️⃣ GOOGLE DORKS

```
site:[TARGET_DOMAIN]                              # index all pages
site:[*.target.com] -site:[www.target.com] # enum subdomains ('-' removes from results)
site:[TARGET_DOMAIN] filetype:pdf
site:[TARGET_DOMAIN] filetype:xls OR filetype:xlsx
site:[TARGET_DOMAIN] filetype:doc OR filetype:docx
site:[TARGET_DOMAIN] inurl:admin
site:[TARGET_DOMAIN] inurl:login
site:[TARGET_DOMAIN] inurl:wp-admin
site:[TARGET_DOMAIN] inurl:config
site:[TARGET_DOMAIN] intitle:"index of"
site:[TARGET_DOMAIN] intitle:"index of" password
site:[TARGET_DOMAIN] intext:"sql syntax near"
site:[TARGET_DOMAIN] intext:"Warning: mysql"
site:[TARGET_DOMAIN] ext:php inurl:?id=
cache:[TARGET_DOMAIN]
related:[TARGET_DOMAIN]
"[TARGET_DOMAIN]" "username" OR "password"
```

---

## 7️⃣ NETCRAFT

```
# Browser only
https://sitereport.netcraft.com/?url=[TARGET_DOMAIN]
https://searchdns.netcraft.com/?restriction=site+contains&host=[TARGET_DOMAIN]

# Key intel to grab:
# → OS / Web Server / Hosting Provider
# → IP history (pivoting to old IPs)
# → Netblock owner
```

---

## 8️⃣ LINKEDIN (Manual OSINT)

```
# Search strings
site:linkedin.com "[TARGET_ORG]"
site:linkedin.com "[TARGET_ORG]" "IT" OR "sysadmin" OR "network"
site:linkedin.com "[TARGET_ORG]" "engineer" OR "developer"

# Intel to extract:
# → Usernames → build wordlist → brute force
# → Tech stack from job postings
# → Email format: first.last@domain / f.last@domain
```

### Email Format Validator

```bash
# Once you guess format, verify with theHarvester or hunter.io
theHarvester -d [TARGET_DOMAIN] -b linkedin
```

---

## 9️⃣ HOST DISCOVERY (Pre-Scan Recon)

```bash
# Live host sweep (run before port scanning)
nmap -sn [TARGET_IP]/24
nmap -sn [TARGET_IP]/24 -oG sweep.txt | grep "Up"
netdiscover -r [TARGET_IP]/24
arp-scan --localnet
fping -a -g [TARGET_IP]/24 2>/dev/null
```

---

## 🔟 `host` — Quick DNS Swiss Army Knife

```bash
host [TARGET_DOMAIN]
host -t mx [TARGET_DOMAIN]
host -t ns [TARGET_DOMAIN]
host -t txt [TARGET_DOMAIN]
host -l [TARGET_DOMAIN] [NAMESERVER_IP]    # zone transfer
host [TARGET_IP]                           # reverse lookup
```

---

## ⚡ QUICK REFERENCE — Tool → Output

|Tool|Primary Output|eJPT Use|
|---|---|---|
|`theHarvester`|Emails, subdomains, IPs|Username wordlists|
|`dnsenum`|Subdomains + zone xfer|Find hidden hosts|
|`dnsrecon`|Full DNS records|Confirm subdomains|
|`dig axfr`|Full zone dump|Internal hostnames|
|`fierce`|Subdomain brute|Quick recon|
|`whois`|Org, registrar, ASN|Netblock range|
|`shodan`|Open ports, banners|Pre-scan intel|
|`whatweb`|Web stack, version|CVE targeting|
|`crt.sh`|Subdomains via certs|Passive host discovery|
|`recon-ng`|Aggregated OSINT|Full org profiling|

---

## 🧠 EXAM DECISION TREE

```
Got a domain?
├── Run theHarvester → emails, subdomains
├── Run dnsenum → try zone transfer
├── Check crt.sh → more subdomains
├── Dig NS records → attempt AXFR on each NS
└── whatweb + wafw00f → fingerprint web stack

Got an IP range?
├── nmap -sn → live hosts
├── whois [TARGET_IP] → org / netblock
└── shodan host [TARGET_IP] → cached banner info

Got a company name?
├── LinkedIn → employees → username format
├── theHarvester -b linkedin → email harvesting
├── Google dorks → exposed files, login pages
└── Netcraft → hosting history + tech stack
```

---
---
