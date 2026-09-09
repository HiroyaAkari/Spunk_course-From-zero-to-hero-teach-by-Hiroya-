«
### 1. Name Servers (Hosting Infrastructure)

- **Nameservers**: `ns1.bluehost.com`, `ns2.bluehost.com` (External commercial DNS hosting provider).
- **Target IP**: `103.67.60.38` (A record for `smis.cs.fs.rupp.edu.kh`).
- **IP Ownership**: Registered directly to **Royal University of Phnom Penh (RUOPP-KH)** under APNIC (`103.67.60.0 - 103.67.61.255`), indicating direct campus/institution hosting for the target service rather than third-party cloud hosting.

### 2. MX Records (Email Provider)

- `rupp-edu-kh.mail.protection.outlook.com`
- **Analysis**: The institution utilizes Microsoft 365 / Exchange Online for email security and handling.

### 3. TXT Records & Verification Tokens

- `v=spf1 include:spf.protection.outlook.com -all` (Strict SPF policy enforcing Microsoft 365 mail dispatch).
- `projectdiscovery-verification=5999bb8774` (Indicates prior reconnaissance or scanning activity associated with ProjectDiscovery tools).

### 4. DNS Security & Zone Transfers

- **DNS Zone Transfer (AXFR)**: Tested against `ns1.bluehost.com` and `ns2.bluehost.com`. As expected, zone transfers are properly restricted and rejected.
- **DNSSEC**: Not enabled on the domain (`rupp.edu.kh`).

### 5. CNAME Chains

- `smis.cs.fs.rupp.edu.kh` resolves directly via an A record (`103.67.60.38`) rather than a CNAME chain, leaving no exposure to subdomain takeover via dangling CNAMEs.
  
  
  ### 1. Discovered Subdomains (HTTPS-enabled via CT logs)

A total of **50 active subdomains** were enumerated across the institutional infrastructure. Notable assets include:

- **Target of Interest**: `smis.cs.fs.rupp.edu.kh`, `api.smis.cs.fs.rupp.edu.kh`
- **Academic & Learning Platforms**: `elearning.rupp.edu.kh`, `lms.rupp.edu.kh`, `docs.rupp.edu.kh`, `ecard.rupp.edu.kh`
- **Infrastructure & DevOps**: `gitlab.rupp.edu.kh`, `minio.itc.rupp.edu.kh`, `cluster-dashboard.itc.rupp.edu.kh`, `ceph-dashboard.itc.rupp.edu.kh`
- **Administrative & Support**: `helpdesk.itc.rupp.edu.kh`, `cpanel.rupp.edu.kh`, `phpmyadmin.itc.rupp.edu.kh`

### 2. Certificate Issuers & Automation

- **Issuers**: The certificates utilized across the institution's subdomains are primarily issued by automated CAs (such as **Let's Encrypt**), indicating automated certificate lifecycle management (ACME protocol / certbot) rather than manual enterprise commercial CAs (like DigiCert or Sectigo).
- **Validity Periods**: Standard 90-day validity periods characteristic of Let's Encrypt automation.

### 3. Wildcard Certificates & Takeover Posture

- **Wildcards**: No institutional-wide wildcard certificates (`*.rupp.edu.kh`) were observed in active issuance logs, reducing broad multi-tenant risk.
- **Takeover Risk**: CNAME chains and subdomain records were evaluated; active subdomains map to legitimate internal or managed infrastructure (Bluehost, Microsoft 365, campus IPs), minimizing dangling CNAME subdomain takeover vectors.

### 1.3 Search Engine Dorking (Google, Bing, DuckDuckGo)

### 1. Exposed Documents & Course Materials

- **Path**: `admin.fe.rupp.edu.kh/uploads/`
- **Findings**: Numerous academic syllabi, engineering course materials, and technical drawing PDFs are publicly indexed (e.g., `TEED_Y2_S1_Circuit_Theory_I...pdf`, `TEED_Y3_S2_Network_programming...pdf`, `TEED_Y1_S2_Mathematics_II...pdf`).
- **Academic Repository**: `cjbar.rupp.edu.kh` indexes institutional journal articles and papers, some of which require password prompts to open.

### 2. Login Panels & Portals

- **MIS Portal**: `misv1.rupp.edu.kh` (Management Information System login page).
- **GitLab Instance**: `gitlab.rupp.edu.kh` (Self-hosted GitLab Community Edition login).
- **Research Management**: `rms.rupp.edu.kh` (Research Management System portal).
- **Virtual Scientific Lab LMS**: `vsl-lm.rupp.edu.kh` (LMS password reset and login portal).

### 3. API, Backups, and Source Code

- No exposed database dumps (`.sql`), raw configuration backups (`.bak`, `.old`), or `.git` repositories were discovered via public search indexing for the target domain.

### 4. Cloud Asset Exposure

- Searches targeting cloud storage buckets (`s3.amazonaws.com`, `storage.googleapis.com`, `blob.core.windows.net`) matching `rupp.edu.kh` did not yield misconfigured internal buckets belonging to the university (results were limited to external entities or unrelated third parties sharing the name "Rupp").

1.4 Web Archives & Historical Data

### 1. Overview of Historical Discovery

- **Total Unique Historical URLs**: **9,102 URLs** successfully extracted and deduplicated from archive sources (Wayback Machine, Common Crawl, etc.).
- **Historical Files Identified**: **849 files** matching document and data extensions (`.pdf`, `.doc`, `.docx`, `.xls`, `.xlsx`, `.xml`, `.json`).
- **Sensitive / Administrative Patterns**: **177 endpoints** containing administrative, portal, or master program directories.

### 2. Key Categories of Discovered Historical Assets

- **Academic & Departmental Documents**: Extensive archives of journal publications (`CJBAR`, `CJNH`), conference programs, and Master's program application forms/handbooks (e.g., Development Studies handbooks, MDS application forms in Word and PDF format).
- **Legacy Portals & Directories**: Historical folder structures dating back to earlier deployments (e.g., `/master/development_studies/`, `/center/it_center/`), reflecting the evolution of the institution's web architecture over the years.
- **Exposed Application Files**: Publicly archived PDF reports, student registration documentation, and program brochures. No critical database dumps (`.sql`), raw config backups (`.bak`), or exposed `.git` repositories were found within the historical archive dataset.

Github recon; 

### 1. GitHub Organization & Repository Analysis

- **Official Organizations**: No official enterprise or institutional GitHub organizations publishing production source code for the main university systems were indexed.
- **Student Repositories**: Discovered individual student repositories (e.g., student course collections and personal profiles mentioning RUPP Department of Computer Science), but these contain only academic coursework and personal projects.

### 2. Secret Scanning & Exposed Credentials

- **API Keys & Tokens**: No exposed AWS access keys, Stripe tokens, SendGrid keys, or third-party API secrets were detected in connection with the target domains.
- **Configuration & Environment Files**: No public `.env`, `config.json`, or database connection strings (`.sql`, connection URIs) were found indexed on GitHub for the target systems.


### 1. Email Naming Conventions

- **Primary Format**: `firstname.lastname@rupp.edu.kh` (accounting for ~68% of identified institutional emails, e.g., `sok.soth@rupp.edu.kh`, `sokha.chan@rupp.edu.kh`).
- **Departmental / Generic Emails**: `info@rupp.edu.kh`, `fe.info@rupp.edu.kh`, `cjbar@rupp.edu.kh`, `editorialassistant@rupp.edu.kh`.

### 2. Employee Roles & Infrastructure Insights

- **Key Personnel**: Academic deans, department heads, and administrative staff are publicly listed across faculty pages (e.g., Faculty of Education, Faculty of Engineering).
- **Technical Staff & Engineering**: Profiles associated with IT Engineering, network security engineering, and student systems indicate internal management of campus network infrastructure, LMS platforms, and research repositories.

### 3. Breach & Public Leak Posture

- **Breach Databases / Leaks**: No institutional email credentials or critical employee password dumps associated with `rupp.edu.kh` appear in public credential leak databases or recent major breaches. Public mentions of data breaches in search indexes relate to academic papers and policy briefs discussing cybersecurity trends rather than active institutional compromise.


### 1. Discovery Summary

- **Total Passive Subdomains Identified**: 133 unique subdomains.
- **Active Resolved Subdomains (`dnsx`)**: 94 live, resolving subdomains.
- **Final Consolidated Active Subdomains**: Saved in `all_discovered_subs.txt` (totaling 94 active endpoints).

### 2. Key Subdomains & Assets Discovered

- **Target & Systems**: `smis.cs.fs.rupp.edu.kh`, `api.smis.cs.fs.rupp.edu.kh`, `mis.rupp.edu.kh`, `mis-api.rupp.edu.kh`, `misv2.rupp.edu.kh`.
- **Learning & Academic Platforms**: `elearning.rupp.edu.kh`, `lms.rupp.edu.kh`, `lms.hsl.rupp.edu.kh`, `vsl-lm.rupp.edu.kh`.
- **Infrastructure & DevOps**: `gitlab.rupp.edu.kh`, `docker.rupp.edu.kh`, `ceph-dashboard.itc.rupp.edu.kh`, `cluster-dashboard.itc.rupp.edu.kh`, `minio.itc.rupp.edu.kh`.
- **Administrative & Portals**: `rms.rupp.edu.kh`, `staff-system.rupp.edu.kh`, `cpanel.rupp.edu.kh`, `phpmyadmin.itc.rupp.edu.kh`.

### 1. Live Host Probing & HTTP Enumeration (`httpx` )

- **Total Live Hosts**: **80 live HTTP/HTTPS services** identified out of the 94 resolved subdomains.
- **Key Tech Stacks & Platforms**:
    - **CMS / Frameworks**: Next.js (`register.rupp.edu.kh`, `ceph-dashboard.itc.rupp.edu.kh`), Laravel/Livewire (`cpanel.rupp.edu.kh`, `test-rms.rupp.edu.kh`), WordPress/Elementor (`nicc.rupp.edu.kh`, `fsshv2.rupp.edu.kh`), Koha LMS (`lms.hsl.rupp.edu.kh`), Strapi (`admin.fe.rupp.edu.kh`), and MinIO (`minio.itc.rupp.edu.kh`).
    - **Admin Panels & Portals**: Discovered operational cPanel interfaces (`cpanel.rupp.edu.kh`), phpMyAdmin (`phpmyadmin.itc.rupp.edu.kh`), MinIO console (`minio.itc.rupp.edu.kh`), and Research Management logins (`rms.rupp.edu.kh`, `test-rms.rupp.edu.kh`).

### 2. Port Scanning (`naabu`)

- Scanned discovered subdomains across top ports, revealing a wide range of open management, database, and custom application ports.

### 3. Email Security Posture (SPF, DKIM, DMARC)

- **SPF Record**: `v=spf1 include:spf.protection.outlook.com -all` (Enforces strict Microsoft 365 dispatch policy).
- **DMARC Record**: `v=DMARC1; p=quarantine; rua=mailto:dmarc@rupp.edu.kh; ruf=mailto:dmarc@rupp.edu.kh; sp=quarantine; aspf=s; adkim=s` (Configured with a **quarantine** policy for unverified emails, reducing spoofing risk).
- **DKIM**: Validated via Microsoft 365 selector (`selector1-rupp-edu-kh._domainkey.ruppedukh.onmicrosoft.com`).

### 1. JavaScript Asset Collection

- **Downloaded Chunks**: Successfully downloaded **18 compiled JavaScript bundles** (`.js`) corresponding to the Next.js application structure.

### 2. Endpoint & Routing Disclosures

- **Internal App Routes**: Client-side routing references include:
    - `/login` (Authentication entry point)
    - `/verify-otp` (Multi-factor / OTP verification flow)
    - `/dashboard` (Post-authentication landing page)
- **Authentication & Fingerprinting Logic**: The application implements client-side browser fingerprinting (via user-agent parsing libraries tracking OS, browser, device type, and component hashes) before submitting credentials to the authentication backend.

### 3. Secret & API Key Audit

- **Hardcoded Secrets**: Deep-grep analysis across all JavaScript chunks (`js_secrets.txt`) revealed **no hardcoded API keys, bearer tokens, AWS credentials, or database secrets**.
- **Security Posture**: The client-side code correctly adheres to security best practices by keeping sensitive API keys, database connection strings, and backend logic server-side (Next.js server/API routes).

### 1. Hidden Parameter Discovery (`Arjun`)

- **Target Scanned**: `https://smis.cs.fs.rupp.edu.kh/login`
- **Discovered Parameter**: Arjun successfully identified the hidden parameter **`signing`** associated with the authentication route.
- **Form Parameters**: The form inputs observed in the login bundle consist of standard credential fields (`email`, `password` ) used by the Next.js frontend to interact with authentication endpoints.

### 2. Parameter Extraction (`unfurl`)

- **Crawled Parameters (`katana_params.txt`)**: Extracted parameters from crawled endpoints across the domain footprint.
- **Analysis**: No sensitive admin override parameters (`?admin=true`), debug flags (`?debug=1`), or obvious IDOR parameters (`?user_id=...`) were exposed in public query strings, reflecting a clean routing design under Next.js.
### 1. Directory & File Brute-Forcing (`ffuf`)

- **Target Scanned**: `https://smis.cs.fs.rupp.edu.kh/FUZZ` using `common.txt`.
- **Discovered Paths**:
    - `/login` (HTTP 200 OK ): Primary authentication gateway.
    - `/_next/` (HTTP 200/404 handling): Next.js application framework asset directories.
- **Sensitive Files & Backups**: No exposed database dumps (`.sql`), raw configuration backups (`.bak`), `.git` repositories, or exposed debug files (`phpinfo.php`) were uncovered during the wordlist scan.

### 2. API Documentation & GraphQL Discovery

- **Endpoints Tested**: `/swagger.json`, `/openapi.json`, `/api-docs`, `/graphql`, `/gql`, and `/service?wsdl`.
- **Results**: All common API documentation paths return standard 404/307 redirects or fall back to the Next.js login SPA, indicating that public API definition files are not exposed.

### 3. WebSocket Discovery

- **JS Bundle Analysis**: Scanned all downloaded JavaScript chunks for `ws://`, `wss://`, and `WebSocket` references.
- **Results**: No active WebSocket endpoints or streaming protocols were detected in the client-side code; communication relies entirely on standard HTTPS REST / JSON API calls.