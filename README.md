# rtb-workshop

2-hour red team / blue team detection lab.

- [Part 1 — Web Attacks + Elastic Rules](part1-web-attacks.md)
- [Part 2 — Windows Attacks + Wazuh Rules](part2-windows-wazuh.md)

## Setup

```bash
git clone https://github.com/Ahmadd018/logacademy
cd logacademy
cp .env.example .env
docker compose up -d
```

Add to `/etc/hosts`:
```
<SERVER_IP>  logacademy.local logportal.local
```

Kibana: `http://<SERVER_IP>:5601`
