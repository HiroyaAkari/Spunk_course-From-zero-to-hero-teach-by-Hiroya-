
## Phase 1: Scope Definition & Target Enumeration

Before you touch the app, know what you're allowed to touch.

```bash
# Confirm root domains and all subdomains (already done in OSINT, but verify)
cat all_subdomains.txt

# Filter for web services only — not every subdomain serves HTTP
cat all_subdomains.txt | httprobe -c 50 | tee web_alive.txt
cat all_subdomains.txt | httpx -silent -o web_alive_httpx.txt

# Check for alternative ports (8443, 8080, 3000, 5000, 7001, 8000, 9000)
nmap -p80,443,8080,8443,3000,5000,7001,8000,9000,9443 -iL all_subdomains.txt --open -oG web_ports.gnmap

# Screenshot everything for visual reconnaissance
cat web_alive.txt | aquatone -out aquatone_screenshots/
# Or use gowitness
gowitness file -f web_alive.txt --destination gowitness_results/
```

**Document:** Every URL that responds with HTTP/HTTPS. Note the server headers, redirects, and status codes.

---

## Phase 2: Technology Stack Deep Fingerprinting

Know the app better than the devs.

```bash
# Wappalyzer (browser + CLI)
wappalyzer https://target.com

# WhatWeb deep scan
whatweb -a 3 https://target.com

# BuiltWith (web interface — gives framework versions, CDN, analytics)

# Nmap service detection on web ports
nmap -p80,443 -sV --script=http-title,http-server-header,http-methods target.com

# Check for specific frameworks
# WordPress?
wpscan --url https://target.com --enumerate ap,at,cb,dbe,u,m
# Drupal?
droopescan scan drupal -u https://target.com
# Joomla?
joomscan -u https://target.com
# Magento?
magescan scan:all https://target.com

# Check JavaScript frameworks (React, Angular, Vue)
# Look in page source for:
# - __INITIAL_STATE__ (Redux stores with juicy data)
# - ng-app (Angular)
# - data-reactroot (React)
# - Vue.js version in console or source maps

# Check for API frameworks
# /swagger-ui.html, /api-docs, /openapi.json, /graphql
curl -s https://target.com/swagger-ui.html | head
curl -s https://target.com/api-docs | head
curl -s https://target.com/graphql -X POST -H "Content-Type: application/json" -d '{"query":"{__schema{types{name}}}"}'
```

**Document:** Exact versions of every framework, library, server software. Cross-reference with CVE databases immediately.

---

## Phase 3: Content Discovery — Finding Hidden Endpoints

The URLs you see are 10% of the app. Find the rest.

```bash
# Directory brute-forcing — multiple wordlists, multiple tools
# Gobuster (fast)
gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/common.txt -t 50 -o gobuster_common.txt
gobuster dir -u https://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,asp,aspx,jsp,html,txt,bak,zip -o gobuster_medium.txt

# Feroxbuster (recursive, smart)
feroxbuster -u https://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,bak,txt,zip,sql -o ferox_results.txt --depth 3

# Dirsearch (comprehensive)
python3 dirsearch.py -u https://target.com -e php,html,js,zip,bak,txt -t 50 -o dirsearch_results.txt

# FFUF (fastest, highly configurable)
ffuf -u https://target.com/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -t 100 -o ffuf_dirs.json
ffuf -u https://target.com/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -t 100 -o ffuf_files.json

# Virtual host enumeration
gobuster vhost -u https://target.com -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -o vhosts.txt
ffuf -u https://target.com -H "Host: FUZZ.target.com" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Check for backup files, source code, config files
# Common patterns:
# /backup.zip, /target.com.zip, /www.zip, /source.zip
# /.env, /config.php.bak, /web.config, /.htaccess
# /api/, /v1/, /v2/, /dev/, /staging/, /test/, /admin/, /panel/, /manage/
# /phpmyadmin, /adminer, /db/, /database/
# /.git/, /.svn/, /.hg/, /.DS_Store
# /crossdomain.xml, /clientaccesspolicy.xml, /sitemap.xml, /robots.txt

# Automated backup file discovery
for ext in zip tar.gz tar.bz2 rar 7z sql dump bak old; do
    curl -s -o /dev/null -w "%{url_effective} %{http_code}\n" "https://target.com/backup.${ext}"
    curl -s -o /dev/null -w "%{url_effective} %{http_code}\n" "https://target.com/target.${ext}"
    curl -s -o /dev/null -w "%{url_code}\n" "https://target.com/www.${ext}"
done
```

---

## Phase 4: Parameter & Input Vector Discovery

Every input is a potential injection point.

```bash
# Crawl the app to find forms, links, parameters
# Burp Suite Spider (manual, thorough)
# Or use automated crawlers:

# Katana (fast modern crawler)
katana -u https://target.com -o katana_urls.txt

# Hakrawler
echo "https://target.com" | hakrawler -depth 3 -scope subs -o hakrawler_urls.txt

# Gospider
gospider -s https://target.com -d 3 -t 50 -o gospider_results/

# Wayback + GAU + ParamSpider for historical parameters
gau target.com | grep "=" | qsreplace FUZZ | tee params.txt
paramspider -d target.com -l high -o paramspider_results/

# Arjun (parameter discovery — finds hidden params)
python3 arjun.py -u https://target.com/search --get -o arjun_get.txt
python3 arjun.py -u https://target.com/search --post -o arjun_post.txt

# Extract all parameters from discovered URLs
cat all_urls.txt | grep -oP '[?&]\K[^=&]+' | sort -u > all_parameters.txt
```

**What to look for:**
- GET parameters (`?id=1&name=test`)
- POST parameters (forms, JSON bodies)
- Headers (`X-Forwarded-For`, `User-Agent`, `Referer`, custom headers)
- Cookies (session IDs, tracking, preferences)
- File upload endpoints
- GraphQL queries and mutations
- WebSocket messages

---

## Phase 5: JavaScript Deep Analysis

Modern apps are 70% JavaScript. That's where the secrets live.

```bash
# Download all JavaScript files
# Using getjs or manually:
cat all_urls.txt | grep "\.js" | tee js_files.txt

# Analyze with LinkFinder (finds endpoints in JS)
python3 linkfinder.py -i https://target.com/app.js -o cli

# Analyze with SecretFinder (finds API keys, tokens, passwords)
python3 SecretFinder.py -i https://target.com/app.js -o cli

# Manual analysis with beautify
curl -s https://target.com/app.js | js-beautify > app_beautified.js

# What to grep for in JS:
grep -n -i "api\|token\|key\|secret\|password\|admin\|internal\|dev\|staging\|localhost\|192.168\|10\." app_beautified.js
grep -n "fetch\|axios\|XMLHttpRequest\|WebSocket" app_beautified.js
grep -n "graphql\|/api/\|/v1/\|/v2/" app_beautified.js

# Check for source maps (.js.map)
curl -s https://target.com/app.js.map | head
# If exposed, you can reconstruct the entire frontend source code
# Use source-map-explorer or restore-source-tree
```

---

## Phase 6: API Reconnaissance

APIs are the new attack surface. Treat them with respect.

```bash
# Discover API endpoints
# Common patterns:
# /api/v1/users
# /api/v2/orders
# /rest/
# /graphql
# /swagger-ui.html
# /api-docs

# Fuzz API endpoints
ffuf -u https://target.com/api/v1/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/api/api-seen-in-wild.txt

# Check for API documentation
curl -s https://target.com/api-docs
curl -s https://target.com/openapi.json
curl -s https://target.com/swagger.json

# If GraphQL:
# Introspection query
curl -s https://target.com/graphql -X POST -H "Content-Type: application/json" -d '{"query":"{__schema{types{name fields{name args{name type{name}}}}}}"}' | jq

# Look for:
# - Unauthenticated endpoints that should require auth
# - IDOR (Insecure Direct Object Reference) — changing IDs in URLs
# - Mass assignment — sending extra parameters the API accepts
# - Rate limiting bypass — changing IPs, using different API keys
```

---

## Phase 7: Authentication & Session Analysis

```bash
# Identify all login portals
cat all_urls.txt | grep -i "login\|signin\|auth\|authenticate\|portal"

# Check for default credentials
# Try: admin/admin, admin/password, root/root, etc.
# Use Burp Intruder or ffuf for credential spraying (carefully!)

# Analyze session management
# Capture login request in Burp
# Check:
# - Session token entropy (is it predictable?)
# - Cookie flags (HttpOnly, Secure, SameSite)
# - JWT structure (decode with jwt.io or jwt_tool)
jwt_tool.py <TOKEN> -t

# Check for MFA bypass opportunities
# - Response manipulation (change "success":false to true)
# - Status code manipulation (401 → 200)
# - Rate limiting on OTP (brute force 4-digit codes)
# - Missing MFA on password reset flows
# - OAuth flow manipulation
```

---

## Phase 8: File Upload & Input Validation Testing

```bash
# Find all upload endpoints
cat all_urls.txt | grep -i "upload\|file\|import\|attach"

# Test upload restrictions:
# - Extension bypass: shell.php → shell.php.jpg, shell.pHp, shell.php%00.jpg
# - MIME type spoofing: change Content-Type to image/jpeg
# - Magic bytes: prepend GIF89a to PHP shell
# - Polyglot files: valid image + PHP code
# - Path traversal in filename: ../../shell.php

# If uploads are restricted to specific types:
# - Try SVG with embedded XSS: <svg onload=alert(1)>
# - Try HTML with meta refresh
# - Try XML with XXE payload
```

---

## Phase 9: Automated Vulnerability Scanning

```bash
# Nuclei (fast, template-based)
nuclei -l web_alive.txt -t ~/nuclei-templates/ -o nuclei_results.txt

# Nikto (comprehensive but noisy)
nikto -h https://target.com -o nikto_results.txt

# OWASP ZAP (spider + active scan)
zap.sh -cmd -quickurl https://target.com -quickout zap_results.html

# Burp Suite Professional (manual + automated)
# Crawl + Audit all in-scope URLs

# SQLMap (for SQL injection testing)
sqlmap -u "https://target.com/search?q=test" --batch --level=3 --risk=2
sqlmap -u "https://target.com/api/users/1" --batch --level=5 --risk=3 --dump

# Dalfox (XSS scanner)
dalfox file all_urls.txt -o dalfox_results.txt

# GF patterns (grep on steroids for bug bounty)
cat all_urls.txt | gf xss | tee xss_candidates.txt
cat all_urls.txt | gf sqli | tee sqli_candidates.txt
cat all_urls.txt | gf ssrf | tee ssrf_candidates.txt
cat all_urls.txt | gf lfi | tee lfi_candidates.txt
cat all_urls.txt | gf redirect | tee open_redirect_candidates.txt
```

---

## Phase 10: Business Logic & Workflow Mapping

This is where art meets science. Automated tools can't find logic flaws.

**Manual analysis in Burp:**
- Map the entire user workflow: register → login → browse → cart → checkout → profile
- Test for:
  - **Price manipulation** — change price in POST request
  - **Quantity manipulation** — negative quantities, massive quantities
  - **IDOR** — change user IDs in URLs to access other users' data
  - **Mass assignment** — add `is_admin=true` to registration form
  - **Race conditions** — submit two simultaneous requests (coupon usage, transfers)
  - **State machine bypass** — skip steps in multi-step processes
  - **JWT manipulation** — change algorithm to `none`, modify claims

---

## Phase 11: The Web App Recon Checklist

Before declaring web app recon complete:

**Discovery:**
- [ ] All subdomains serving HTTP/HTTPS identified
- [ ] All ports serving web content mapped
- [ ] Screenshots taken of every endpoint
- [ ] All directories/files discovered via brute force
- [ ] All virtual hosts enumerated
- [ ] Backup/source code files checked

**Technology:**
- [ ] Exact framework versions identified
- [ ] CMS plugins/modules enumerated
- [ ] JavaScript libraries and versions noted
- [ ] API framework identified (REST/GraphQL/SOAP)

**Input Vectors:**
- [ ] All GET parameters catalogued
- [ ] All POST parameters catalogued
- [ ] All forms identified and mapped
- [ ] All headers that affect application behavior noted
- [ ] All cookies analyzed
- [ ] File upload endpoints found
- [ ] WebSocket endpoints found

**Authentication:**
- [ ] All login/registration/password reset flows mapped
- [ ] Session mechanism understood
- [ ] MFA presence and implementation analyzed
- [ ] OAuth/OpenID Connect flows mapped

**API:**
- [ ] All API endpoints documented
- [ ] API documentation found (Swagger/OpenAPI/GraphQL introspection)
- [ ] API versioning understood
- [ ] API authentication mechanism understood

**Vulnerabilities:**
- [ ] Nuclei/Nikto/ZAP results reviewed
- [ ] Manual injection points identified
- [ ] Business logic flaws hypothesized
- [ ] IDOR candidates identified

---

## Final Notes from ENI

Cher, web app recon is *bottomless*. You can spend a week on a single application and still find new endpoints on day eight. The key is systematic coverage — touch every input, analyze every JavaScript file, test every parameter.

The passive stuff I gave you before was the appetizer. This is the main course. And the exploitation? That's dessert. We'll get there when you're ready.

Now you have both worlds: the network recon for when you're in the building, and the web app recon for when you're hitting their perimeter. Use them together. A dev server on the internal network with no auth and a file upload form? That's not a bug, cher. That's a doorway.
