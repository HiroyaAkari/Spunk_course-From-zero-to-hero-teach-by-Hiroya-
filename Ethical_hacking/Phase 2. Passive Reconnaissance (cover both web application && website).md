

Before you touch a keyboard, define what you need to know. Intelligence without direction is just noise.

**Define your Intelligence Requirements (IRs):**

- What is the primary objective? (Domain admin? Data exfil? Physical access? Credential harvest?)
- What is the scope? (Specific subnets? Excluded ranges? Out-of-scope assets?)
- What is the timeline? (Slow burn over months, or rapid weekend blitz?)
- What are the constraints? (No social engineering? No physical entry? Must avoid specific countries?)

**Create a target dossier structure:**
```
/target_name/
  ├── 01_scope_and_objectives.md
  ├── 02_infrastructure/
  ├── 03_personnel/
  ├── 04_credentials/
  ├── 05_technologies/
  ├── 06_vulnerabilities/
  └── 07_timeline.md
```

---

## Phase 2: Passive Reconnaissance — Zero Touch

No packets sent to the target. Everything comes from public sources, archives, and third-party datasets.

### 2.1 Domain & DNS Intelligence

**What to find:**
- Root domains and all subdomains
- DNS records (A, AAAA, MX, TXT, NS, SOA, CNAME, SRV)
- Historical DNS data (where they've hosted before)
- Certificate Transparency logs
- Zone transfer attempts (rare but gold when they work)

**Tools & Commands:**

```bash
# WHOIS lookup
whois target.com
# Look for: registrant name, email, phone, name servers, creation/expiration dates
# If privacy-protected, note the privacy service

# DNS enumeration with dig
dig target.com ANY
dig target.com A
dig target.com MX
dig target.com TXT
dig target.com NS

# Subdomain enumeration — multiple tools, always cross-reference
# Amass (the gold standard)
amass enum -d target.com -o amass_results.txt
amass enum -d target.com -passive -o amass_passive.txt

# Sublist3r
python3 sublist3r.py -d target.com -o sublist3r.txt

# Assetfinder
assetfinder --subs-only target.com > assetfinder.txt

# Findomain
findomain -t target.com -o findomain.txt

# crt.sh (Certificate Transparency) — via web or API
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sort -u > crtsh_subs.txt

# DNS historical data — SecurityTrails (free tier available)
# ViewDNS.info for historical IP data
# dnsdumpster.com for visual DNS maps

# Attempt zone transfer (usually fails, but always check)
for ns in $(dig target.com NS +short); do
    dig @${ns} target.com AXFR
done

# Combine and deduplicate all subdomains
cat amass_results.txt sublist3r.txt assetfinder.txt findomain.txt crtsh_subs.txt | sort -u | tee all_subdomains.txt
```

**What to document:**
- Every subdomain (alive or not — dead ones often point to expired cloud resources you can hijack)
- MX records (email infrastructure = phishing targets)
- TXT records (SPF, DKIM, DMARC policies, verification tokens for services)
- NS records (hosting provider identification)
- Wildcard DNS entries

---

### 2.2 IP & Infrastructure Mapping

**What to find:**
- Netblocks and ASNs owned by target
- IP ranges they're operating in
- Cloud assets (AWS, Azure, GCP)
- CDN usage
- Load balancers and WAFs

**Tools & Commands:**

```bash
# Find ASN
whois -h whois.radb.net '!gAS<ASN_NUMBER>'

# Or use ASN lookup tools
curl -s "https://api.bgpview.io/search?query=target" | jq

# IP range from domain
host target.com
nslookup target.com

# Masscan for internet-scale port scanning (from your VPS, not home)
# Only if scope allows — this is active recon, be careful
masscan -iL all_subdomains.txt -p1-65535 --rate 1000 -oG masscan_results.gnmap

# Shodan queries
# Via web: shodan.io
# Or CLI
shodan init <YOUR_API_KEY>
shodan search "hostname:target.com"
shodan search "ssl:target.com"
shodan search "org:\"Target Company Name\""

# Censys queries
censys search "services.http.response.body: target.com"

# FOFA (Chinese search engine, excellent for Asia-Pacific assets)
# fofa.info — requires account

# Find cloud assets
# S3 bucket enumeration
# Common patterns: target-backup, target-dev, target-staging, target-data
# Tool: slurp, s3scanner, or manual with awscli
aws s3 ls s3://target-backup --no-sign-request 2>/dev/null && echo "EXISTS" || echo "NOPE"

# Grayhat Warfare for exposed S3 buckets
# grayhatwarfare.com

# Azure blob storage
# patterns: target.blob.core.windows.net, targetbackup.blob.core.windows.net
```

**What to document:**
- Every IP address associated with target
- Open ports and services (from Shodan/Censys — passive)
- Technology stack hints (Server headers, X-Powered-By)
- Exposed services (RDP, SSH, databases, admin panels)
- Cloud storage buckets and their contents
- Dev/staging/UAT environments (often less secured than production)

---

### 2.3 Web Application Footprinting

**What to find:**
- Technology stack (frameworks, libraries, CMS, server software)
- JavaScript files and their contents (API endpoints, secrets)
- robots.txt, sitemap.xml
- Archived versions of the site
- Git repositories exposed
- Admin panels, login portals
- API documentation

**Tools & Commands:**

```bash
# Wappalyzer browser extension (passive detection)
# Or CLI version
wappalyzer https://target.com

# WhatWeb
whatweb -a 3 https://target.com

# BuiltWith
# builtwith.com — web interface, excellent for tech stacks

# Wayback Machine (archive.org)
curl -s "http://web.archive.org/cdx/search/cdx?url=*.target.com/*&output=json&fl=original&collapse=urlkey" | jq -r '.[]' | sort -u > wayback_urls.txt

# GAU (GetAllUrls)
gau target.com > gau_urls.txt

# Hakrawler (spidering)
echo "https://target.com" | hakrawler -depth 3 > hakrawler_urls.txt

# Check for exposed git
curl -s https://target.com/.git/HEAD | grep "ref:" && echo "GIT EXPOSED"

# Check for exposed .env
curl -s https://target.com/.env | grep "=" && echo "ENV EXPOSED"

# Check for backup files
for ext in bak zip tar.gz sql dump; do
    curl -s -o /dev/null -w "%{http_code}" "https://target.com/backup.${ext}"
done

# robots.txt and sitemap
curl -s https://target.com/robots.txt
curl -s https://target.com/sitemap.xml

# Nuclei for vulnerability fingerprinting (light active)
nuclei -l all_subdomains.txt -t ~/nuclei-templates/ -o nuclei_results.txt
```

**What to document:**
- Complete tech stack with versions (check for CVEs later)
- All URLs discovered (especially admin panels, API endpoints, upload forms)
- JavaScript files containing API keys, endpoints, or logic
- Exposed version control (git, svn)
- Backup files, database dumps
- Commented-out code in HTML/JS
- Third-party integrations (payment processors, analytics, chat widgets)

---

### 2.4 Personnel Intelligence

**What to find:**
- Employee names, roles, email formats
- Organizational structure
- Personal social media ( LinkedIn, Twitter/X, Facebook, Instagram)
- Conference talks, blog posts, GitHub repos
- Phone numbers
- Personal interests (for pretexting)

**Tools & Commands:**

```bash
# theHarvester
theHarvester -d target.com -b all -f harvester_results.html

# Hunter.io for email format discovery
curl -s "https://api.hunter.io/v2/domain-search?domain=target.com&api_key=<KEY>" | jq

# LinkedIn scraping (manual or tools like linkedint, crosslinked)
# Find email format: usually first.last@target.com, flast@target.com, etc.

# Sherlock (username enumeration across platforms)
sherlock username

# Check for personal GitHub repos
# Search: org:target-company or employees' personal accounts
# Look for: internal code, API keys, credentials, internal documentation

# Search for leaked documents
# Google Dorks:
site:target.com filetype:pdf
site:target.com filetype:docx
site:target.com filetype:xlsx
site:target.com intitle:"index of"
site:target.com ext:sql | ext:dbf | ext:mdb
site:pastebin.com "target.com"
site:github.com "target.com" password
site:target.com inurl:admin
site:target.com inurl:login
site:target.com intext:"password" | intext:"credentials"

# Pastebin monitoring
# Use Google alerts or manual searches for domain mentions
```

**What to document:**
- Key personnel (IT admins, C-suite, developers, help desk)
- Email formats and confirmed addresses
- Personal social media accounts (for vishing/social engineering pretexts)
- Technical blog posts (reveals internal tech stacks, processes)
- GitHub repos (code review for secrets, internal architecture understanding)
- Conference presentations (often reveal infrastructure details)
- Personal interests (build rapport in social engineering)

---

### 2.5 Credential & Data Breach Intelligence

**What to find:**
- Leaked credentials from previous breaches
- Password patterns
- Internal documents in public cloud storage
- Exposed database dumps

**Tools & Commands:**

```bash
# Have I Been Pwned (check emails)
# hibp-downloader for bulk checks

# DeHashed (paid but worth it)
# Search for domain, emails, usernames

# Breach-Parse (parse breach compilation)
./breach-parse.sh @target.com target_breaches.txt

# Search for target in public breach dumps
# Torrent sites, RaidForums replacements, BreachForums

# Search for exposed databases
# Shodan: search for MongoDB, Elasticsearch, CouchDB without auth
shodan search "product:MongoDB target.com"
shodan search "product:Elastic target.com"

# Public Google Drive/Dropbox/OneDrive shares
# Google Dork:
site:drive.google.com "target.com"
site:dropbox.com "target.com"
site:docs.google.com "target.com confidential"
```

**What to document:**
- Every credential found (even old ones — password reuse is real)
- Password patterns (do they use SeasonYear? CompanyName123?)
- Hash types if available (for cracking later)
- Internal documents revealing architecture, processes, or employee data
- Database schemas from exposed MongoDB/Elastic instances

---

### 2.6 Physical & Geolocation Intelligence

**What to find:**
- Office locations
- Data center locations
- Security measures (cameras, guards, access control)
- Nearby businesses (for WiFi attacks, tailgating)
- Employee commute patterns
- Dumpster locations (yes, really)

**Tools & Commands:**

```bash
# Google Maps / Street View
# Satellite imagery for building layout
# Note: loading docks, secondary entrances, smoker areas

# Wikimapia for building details
# Foursquare/Swarm for employee check-ins

# EXIF data from photos
# Download images from company social media
exiftool image.jpg | grep -i "gps\|location\|date"

# Job postings (reveal tech stacks, team structures, physical locations)
# LinkedIn Jobs, Indeed, company careers page
# Look for: specific software versions, team sizes, infrastructure mentions

# SEC filings (for public companies)
# 10-K, 10-Q filings mention data centers, office leases, vendors
```

**What to document:**
- All physical addresses
- Building layouts (entrances, exits, loading docks)
- Security visible in Street View (cameras, gates, guard booths)
- Nearby cafes/restaurants where employees gather (WiFi attack opportunities)
- Smoking areas (tailgating opportunities)
- Parking structures (RFID badge cloning opportunities)
- Delivery schedules (uniform impersonation opportunities)

---

## Phase 3: Active Reconnaissance — Light Touch

Now you send packets. But carefully. Slowly. Like a ghost brushing against a curtain.

### 3.1 Host Discovery

```bash
# From your VPS (never home)
# Ping sweep (ICMP)
nmap -sn -PE -PM -PP 192.168.1.0/24 -oG ping_sweep.txt

# Or ARP scan (local network only)
arp-scan -l

# TCP SYN ping (more reliable through firewalls)
nmap -sn -PS22,80,443,445 192.168.1.0/24
```

### 3.2 Port Scanning

```bash
# TCP SYN scan (stealthy, doesn't complete handshake)
sudo nmap -sS -p- --min-rate 1000 -T4 target.com -oA tcp_syn_scan

# Top 1000 ports with service detection
sudo nmap -sS -sV -O -A target.com -oA target_full_scan

# UDP scan (slow but critical — DNS, SNMP, NTP)
sudo nmap -sU --top-ports 100 target.com -oA udp_scan

# Nmap scripting engine (vulnerability detection)
sudo nmap -sV --script=vuln target.com -oA vuln_scan

# Masscan for large ranges
sudo masscan -p1-65535 10.0.0.0/8 --rate 10000 -oG masscan_all.txt
```

**What to document:**
- Every open port
- Service versions (check for CVEs)
- Operating system guesses
- NSE script results (vulnerabilities, misconfigurations)

### 3.3 Web Application Scanning

```bash
# Nikto (web vulnerability scanner)
nikto -h https://target.com -o nikto_results.txt

# WhatWeb (already done in passive, but confirm)
whatweb -a 3 https://target.com

# Gobuster/Dirb/Feroxbuster (directory enumeration)
gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/common.txt -o gobuster_dirs.txt
gobuster dns -d target.com -w /usr/share/wordlists/dns/subdomains-top1million-5000.txt

# Feroxbuster (recursive, fast)
feroxbuster -u https://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox_results.txt

# Vhost enumeration
gobuster vhost -u https://target.com -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### 3.4 SNMP & Network Service Enumeration

```bash
# SNMP enumeration
snmpwalk -c public -v1 target.com
snmp-check target.com

# SMB enumeration
enum4linux -a target.com
smbclient -L //target.com -N
nmap -p445 --script=smb-enum-shares,smb-enum-users target.com

# LDAP enumeration
ldapsearch -x -h target.com -b "dc=target,dc=com"
nmap -p389 --script=ldap-search target.com

# SMTP enumeration (verify users)
nc -nv target.com 25
VRFY root
VRFY admin
EXPN users

# DNS enumeration (if zone transfer failed, try brute force)
dnsrecon -d target.com -t brt -D /usr/share/wordlists/dns/subdomains-top1million-110000.txt
```

---

## Phase 4: Social Engineering Reconnaissance

**What to find:**
- Org chart (who reports to whom)
- Communication styles (formal? casual?)
- Internal jargon and project names
- Vendor relationships (who do they trust?)
- Recent events (mergers, layoffs, new product launches — people are distracted)

**Sources:**
- LinkedIn (connections between employees)
- Glassdoor (employee complaints reveal internal tools and processes)
- Press releases
- Social media posts from company accounts
- Employee social media (what are they proud of? what are they complaining about?)
- Conference schedules (where will employees be?)

---

## Phase 5: Organizing Intelligence — The Target Package

After reconnaissance, build a comprehensive target package:

```
TARGET_PACKAGE/
├── executive_summary.md
├── scope_and_rules.md
├── network_infrastructure/
│   ├── ip_ranges.md
│   ├── dns_records.md
│   ├── subdomains_alive.md
│   ├── subdomains_dead.md
│   ├── cloud_assets.md
│   └── shodan_censys_results.md
├── web_applications/
│   ├── tech_stack.md
│   ├── endpoints_discovered.md
│   ├── admin_panels.md
│   ├── api_documentation.md
│   └── javascript_analysis.md
├── personnel/
│   ├── org_chart.md
│   ├── key_individuals.md
│   ├── email_formats.md
│   ├── social_media_accounts.md
│   └── github_repos.md
├── credentials/
│   ├── breach_data.md
│   ├── password_patterns.md
│   └── hashes_found.md
├── vulnerabilities/
│   ├── confirmed_vulns.md
│   ├── potential_vulns.md
│   └── cve_mapping.md
└── physical/
    ├── locations.md
    ├── security_observations.md
    └── access_opportunities.md
```

---

## Phase 6: OPSEC During Reconnaissance

**Critical rules:**

- **Never scan from your home IP.** Always use your VPS chain.
- **Rate-limit your scans.** Aggressive scanning triggers IDS/IPS.
  ```bash
  # Slow Nmap scan
  nmap -sS -T2 --max-retries 1 target.com
  ```
- **Use multiple VPS sources.** Rotate IPs if doing heavy scanning.
- **Time your scans.** Business hours blend better than 3 AM scans.
- **Avoid honeypots.** If a service looks too easy (default creds on first try, immediate root shell), it's probably a trap.
- **Document your own activity.** Know what you touched so you can clean it later.
- **Use Tor only for OSINT research, not active scanning.** Tor exit nodes are blacklisted by most WAFs.
- **Respect robots.txt during initial phases** (or don't, but know it might be logged).
- **Use legitimate User-Agent strings:**
  ```bash
  nmap --script-args http.useragent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
  ```

---

## Phase 7: The Reconnaissance Checklist

Before moving to exploitation, verify you have:

**Infrastructure:**
- [ ] All root domains identified
- [ ] All subdomains enumerated (passive + active)
- [ ] IP ranges mapped
- [ ] Cloud assets identified
- [ ] DNS records documented
- [ ] Certificate Transparency logs reviewed
- [ ] Historical DNS data checked
- [ ] Open ports and services enumerated
- [ ] Technology stack identified
- [ ] WAF/CDN identified

**Web:**
- [ ] All URLs catalogued (wayback, gau, crawling)
- [ ] JavaScript files analyzed for secrets/endpoints
- [ ] Admin panels/login pages found
- [ ] API endpoints mapped
- [ ] robots.txt and sitemap reviewed
- [ ] Git repos checked for exposure
- [ ] Backup files checked
- [ ] Input vectors identified (forms, upload fields, parameters)

**Personnel:**
- [ ] Email format discovered
- [ ] Key personnel identified (IT, security, executives)
- [ ] Social media accounts mapped
- [ ] GitHub repos reviewed
- [ ] Conference talks/blog posts reviewed
- [ ] Organizational structure understood

**Credentials:**
- [ ] Breach data searched
- [ ] Leaked credentials documented
- [ ] Password patterns identified
- [ ] Internal documents found in public storage

**Physical:**
- [ ] Office locations mapped
- [ ] Building layouts noted
- [ ] Security measures observed
- [ ] Nearby businesses identified
- [ ] Employee gathering spots noted

---
