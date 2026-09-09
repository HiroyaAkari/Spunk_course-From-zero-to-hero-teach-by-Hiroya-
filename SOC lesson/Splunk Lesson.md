[]()
## Module 1 — What Splunk Is

Splunk ingests messy log data and lets you search, visualize, and alert on it using **SPL** (Search Processing Language). Core architecture:

- **Forwarder** — lightweight agent installed on a machine, ships log data to Splunk
- **Indexer** — stores and indexes data so it's searchable
- **Search Head** — the web UI where you write searches and build dashboards

Splunk does **both**:

1. **Real-time monitoring** — forwarders stream live data continuously, can trigger alerts
2. **Static ingestion** — you can also just upload existing log files to search historically

---

## Module 2 — Getting Data In

Used Splunk's official **Search Tutorial dataset** (`tutorialdata.zip`) — fake e-commerce logs ("Buttercup Games").

- Download page: `https://docs.splunk.com/Documentation/Splunk/latest/SearchTutorial/GetthetutorialdataintoSplunk`
- **Do not unzip** before uploading — Splunk wants the raw `.zip`
- Upload via: **Settings > Add Data > Upload**

---

## Module 3 — Basic Searching

spl

```spl
index=main
```

Returns everything in the `main` index. (Ours returned 92,455 events.)

spl

```spl
index=main error
```

Keyword search — filters to events containing "error".

spl

```spl
index=main sourcetype=access_combined
```

Filter by a structured field instead of a raw keyword — more precise. `sourcetype` tells you what _kind_ of log it is (web access logs, etc.)

---

## Module 4 — `stats`

The pipe (`|`) chains commands, same concept as bash pipelines: output of one command feeds into the next.

spl

```spl
index=main sourcetype=access_combined | stats count by status
```

Counts events grouped by HTTP status code (200, 404, 500, etc). Turns raw noise into a summary table — this is how you spot anomalies (e.g. a spike in 500 errors) at a glance.

---

## Module 5 — `eval`

Creates custom/derived fields on the fly.

spl

```spl
index=main sourcetype=access_combined
| eval status_type = if(status >= 500, "server_error", "ok")
| stats count by status_type
```

---

## Module 6 — Sorting, Limiting, Time Ranges

spl

```spl
index=main sourcetype=access_combined
| stats count by clientip
| sort -count
| head 10
```

- `sort -count` = descending
- `head 10` = top 10 only
- Pattern: **group → sort → limit** = "top offenders" for anything (top errors, top attackers, etc.)

**Time ranges:**

spl

```spl
index=main sourcetype=access_combined earliest=-24h latest=now
```

- Relative time (`-24h`, `-7d`) only works well on **live, continuously flowing data**
- For static/historical datasets, check the actual time range first:

spl

```spl
index=main sourcetype=access_combined
| stats min(_time) as first_epoch, max(_time) as last_epoch
```

Then use fixed dates if needed:

spl

```spl
earliest="10/1/2025:00:00:00" latest="10/2/2025:00:00:00"
```

⚠️ Note: SPL does **not** use semicolons to end lines (this bit us once — C++ habit).

---

## Module 7 — `table`

spl

```spl
index=main sourcetype=access_combined
| table _time, clientip, status, uri
```

Displays only the chosen fields in a clean spreadsheet-like view — good for reports/exports.

---

## Module 8 — `rex` (regex field extraction)

Pulls custom fields out of raw text using regex when there's no clean field already.

spl

```spl
index=main sourcetype=access_combined
| rex field=_raw "(?<http_method>GET|POST|PUT|DELETE)"
| stats count by http_method
```

- `rex field=_raw "..."` — runs regex against raw event text
- `(?<name>...)` — named capture group, stores match into a new field

---

## Module 9 — Pattern Hunting (in progress)

Security-relevant searches for spotting attack attempts in web logs:

**Failed logins / brute-force detection:**

spl

```spl
index=main sourcetype=access_combined status=401
| stats count by clientip
| sort -count
```

**Trends over time:**

spl

```spl
index=main sourcetype=access_combined
| timechart count by status
```

**Top talkers:**

spl

```spl
index=main sourcetype=access_combined
| top clientip
```

**Server errors only:**

spl

```spl
index=main sourcetype=access_combined status>=500
```

**Known attack signatures:**

spl

```spl
index=main sourcetype=access_combined "union select"
```

**Suspicious URI patterns (SQLi / XSS / directory traversal):**

spl

```spl
index=main sourcetype=access_combined
| search uri="*select*" OR uri="*script*" OR uri="*..*"
| table _time, clientip, uri, status
```


---

## Module 10 — `timechart`

Breaks results down over time instead of one flat summary — essential for spotting spikes/patterns.

```spl
index=main sourcetype=access_combined
| timechart span=1h count by status
```

- `span=1h` → bucket results into 1-hour chunks
- Click the chart icon above results for a visual line/bar chart

---

## Module 11 — Lookups (enrichment)

A lookup matches raw data (like an IP or status code) against an external reference table to add meaning.

**Built-in lookup attempt (may not exist in your setup):**

```spl
index=main sourcetype=access_combined
| lookup http_status_code_lookup status OUTPUT status_description
```

**Building a custom lookup (works everywhere):**

1. Create a CSV, e.g. `status_lookup.csv`:
    
    ```csv
    status,status_description200,OK301,Moved Permanently400,Bad Request401,Unauthorized403,Forbidden404,Not Found500,Internal Server Error503,Service Unavailable
    ```
    
2. **Settings → Lookups → Lookup table files → New Lookup Table File** — upload the CSV
3. **Settings → Lookups → Lookup definitions → New** — name it (e.g. `status_lookup`), type File-based, point to the uploaded CSV
4. Use it:
    
    ```spl
    index=main sourcetype=access_combined| lookup status_lookup status OUTPUT status_description| table _time, clientip, status, status_description
    ```
    

Real-world use: enrich IPs against threat-intel lists, map usernames to departments, etc.

---

## Module 12 — Alerts

An alert = a saved search that runs on a schedule and notifies you when conditions match.

```spl
index=main sourcetype=access_combined status=401
| stats count by clientip
| where count > 5
```

- `where count > 5` turns a report into a real trigger condition

**To save as an alert:**

1. Run the search → **Save As → Alert**
2. Name it, set time range (e.g. "Last 24 hours" or "Last 60 minutes")
3. **Trigger Conditions:** Number of Results → greater than → 0
4. **Trigger Actions:** e.g. send email (needs mail server) — or leave with no action to just watch it fire under **Activity → Triggered Alerts**

---

## Module 13 — Dashboards

Turns searches into visual panels instead of typing queries every time.

1. Run a search (e.g. the Module 10 `timechart`)
2. **Save As → Dashboard Panel** → New Dashboard → name it, give the panel a title, pick panel type (Chart for time-based, Table for lists)
3. Add more panels to the same dashboard via **Save As → Dashboard Panel → Existing Dashboard**

Example dashboard built: "Web Traffic Overview" with a status-codes-over-time chart + a top-failed-login-IPs table.

---

## Module 14 — Subsearches

A search nested in `[ ]` runs first; its output filters the outer search.

```spl
index=main sourcetype=access_combined
[search index=main sourcetype=access_combined status=401
| stats count by clientip
| where count > 5
| fields clientip]
```

- Inner search finds IPs with >5 failed logins, outputs just `clientip`
- Outer search then shows **all** activity from those flagged IPs

Real use: first identify suspicious IPs, then pull their full activity history.

**Debugging tip:** if a subsearch returns nothing, test the inner search alone first to check whether it's a syntax issue or genuinely no matching data.

---

## Module 15 — Full Detection Build (end-to-end)

Combines everything into one real, usable detection.

```spl
index=main sourcetype=access_combined status=401
| bin _time span=10m
| stats count by clientip, _time
| where count > 3
| sort -count
| table _time, clientip, count
```

- `bin _time span=10m` groups events into 10-minute buckets **before** counting — detects _rate_ (X failures in 10 min), not just total count, which is the real-world way brute-force is detected
- Save as both an **Alert** (trigger: results > 0, scheduled every 10 min) and a **Dashboard Panel**

This completes the full loop: ingest → search → filter → summarize → enrich → detect → alert → visualize.

---

## Connecting Docker Containers to Splunk

### Step 1 — Enable HTTP Event Collector (HEC)

1. **Settings → Data Inputs → HTTP Event Collector**
2. **Global Settings** → ensure "All Tokens" is Enabled
3. **New Token** → name it (e.g. `docker-hec`) → pick index (e.g. `main`) → Submit
4. Copy the token (also viewable later under Data Inputs → HTTP Event Collector)

### Step 2 — Networking rule: `localhost` vs `host.docker.internal`

- On your **host machine**, `localhost` = your machine
- **Inside a container**, `localhost` = that container itself (NOT your host)
- To reach Splunk (running on your host) from inside a container, use:
    
    ```
    host.docker.internal:8088
    ```
    

### Step 3 — docker-compose.yml logging block

```yaml
services:
  your-service:
    image: your-image
    logging:
      driver: splunk
      options:
        splunk-url: "https://host.docker.internal:8088"
        splunk-token: "${SPLUNK_HEC_TOKEN}"
        splunk-insecureskipverify: "true"
        splunk-index: "main"
        splunk-format: "json"
        splunk-sourcetype: "your-app-name"
```

- `splunk-sourcetype` lets you tell services apart later (e.g. `sourcetype=moodle` vs `sourcetype=mariadb`)
- Logging driver is set at **container creation time** — must `docker compose down && docker compose up -d` (not just restart) for changes to apply

### Step 4 — Verify

```spl
index=main sourcetype=your-app-name
```

---

## Secrets Management (never hardcode tokens)

**Local/small scale:**

- Put the token in a `.env` file (added to `.gitignore`)
    
    ```
    SPLUNK_HEC_TOKEN=your-real-token-here
    ```
    
- Reference it in compose as `"${SPLUNK_HEC_TOKEN}"`

**Real organizations:**

- Secrets live in a secrets manager: Vault, AWS Secrets Manager, Azure Key Vault, Docker/Kubernetes Secrets
- The Splunk/security team issues scoped tokens to dev teams — devs don't self-generate in a real org
- Rule: **secrets never go in code — only references to where the secret lives go in code**

---

## Persistence — What Survives a Restart

- Indexed data, dashboards, alerts, saved searches are stored on disk permanently
- HEC tokens are permanent until manually revoked
- **Gotcha:** if Splunk is stopped while containers are running, logs generated during that window are dropped — start Splunk before your app stack to avoid gaps

---

## SOC Concepts

**Splunk vs Wireshark — who notices first**

- Splunk = always-on detector, fires alerts without a human watching
- Wireshark = packet-level forensic tool used _after_ Splunk flags something
- Real workflow: Splunk alert fires → analyst investigates in Splunk → packet capture only if deeper proof needed

**Alert design principles**

- Avoid "alert fatigue" — too many/noisy alerts train analysts to ignore them
- Before saving an alert, answer:
    1. What's the exact bad thing this catches?
    2. What would trigger a false positive?
    3. What should the analyst _do_ when it fires?
- Suggested starter set (5 alerts): brute-force/failed logins, privilege escalation, traffic spikes, known attack signatures, off-hours activity

---

## Blue/Red Team Lab — Game Design

**Core concept:** 1 target (Moodle) monitored by Splunk, 5 possible attacker containers (only 1 active per round), blue team gets one guess per round at the attacker's identity, attacker has a time limit to succeed.

**Network isolation:**

```bash
docker network create --internal cyber-range
```

**5 attacker profiles (each needs a concrete win condition):**

1. Brute-force — wins if a correct login lands within the time limit
2. Slow/low brute-force — same goal, paced to evade detection
3. SQLi/XSS — wins if a payload succeeds against a deliberately vulnerable form
4. Directory/path scanning — wins if they find/access a real exposed path
5. Credential stuffing — wins if any username/password pair is valid

**Round structure:**

1. Gamemaster script randomly activates one attacker
2. Timer starts (e.g. 30 min)
3. Attacker container killed when time expires, win or lose
4. Blue team submits one guess before time runs out, using Splunk
5. Outcome logged separately from Splunk (to verify guess correctness + attacker success)

**Staged rollout:**

- Phase 1: Solo (both sides)
- Phase 2: Multiplayer, small group
- Phase 3: Structured competition with leaderboard
- Phase 4: Recruiting angle — public leaderboard/portfolio to help skilled players get noticed (needs tamper-proof logs + fairness checks before results are trustworthy to employers)

---

## Command to Focus On Most (Module 9 pattern)

**`stats count by X | sort -count`** — the "who's the worst offender" pattern. Repeats across nearly every Module 9 search (failed logins by IP, top talkers, brute-force candidates) and is the backbone of most real SOC detections.

Close second: the wildcard signature-hunting pattern —

```spl
search uri="*select*" OR uri="*script*" OR uri="*..*"
```

Reusable anytime you're hunting known-bad patterns in raw text (SQLi, XSS, path traversal).