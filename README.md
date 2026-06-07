# Red Team vs Blue Team: Detection Lab

2-hour hands-on workshop. Red team (mentor) launches real attacks; blue team (students) watches logs and writes detection rules.

## Lab Overview

| Component | Role |
|---|---|
| Kali Linux (mentor) | Attacker machine |
| logacademy server | Vulnerable PHP web app + Elasticsearch/Kibana |
| Wazuh server | SIEM for Windows events |
| Windows VM 1 & 2 | Wazuh agents / attack targets |

## Agenda

| Time | Activity |
|---|---|
| 0:00 – 0:05 | Lab check: everyone can reach logacademy and Kibana |
| 0:05 – 0:35 | **Part 1** — Web attacks + Elastic detection rules |
| 0:35 – 0:55 | Wazuh install + Windows agent setup |
| 0:55 – 1:50 | **Part 2** — Windows attacks + Wazuh detection rules |
| 1:50 – 2:00 | Debrief |

## Parts

- [Part 1 — Web Application Attacks & Elastic Rules](part1-web-attacks.md)
- [Part 2 — Windows Attacks & Wazuh Rules](part2-windows-wazuh.md)

## Quick Setup Check

**logacademy** — follow the repo README to bring it up with Docker Compose:
```bash
git clone https://github.com/Ahmadd018/logacademy
cd logacademy
cp .env.example .env
docker compose up -d
```

Add to `/etc/hosts` on every machine:
```
<SERVER_IP>  logacademy.local logportal.local
```

Kibana → `http://<SERVER_IP>:5601`

> The mentor performs attacks from Kali. Students watch Kibana/Wazuh and write rules. After each attack section the class reviews what fired and why.
