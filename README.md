# Python-UDP-multiplexer-for-Gran-Turismo-7-on-PS5.
Receives telemetry broadcasts from PS5 GT7 and forwards them to multiple local applications (SimTools, SimHub, etc.), enabling motion platforms and dashboards to work simultaneously.

# 🎮 GT7 UDP Multiplexer  
**Gran Turismo 7 (PS5) Telemetry Splitter for SimTools, SimHub & GT7 Proxy**

## 📌 What is this?

**GT7 UDP Multiplexer** is a small, dependency-free Python script that allows **Gran Turismo 7 on PS5** to send telemetry data to **multiple PC applications at the same time**.

Normally, GT7 telemetry can only be received by **one application per UDP port**, which causes conflicts when using:

- SimTools using GT7 Proxy
- SimHub (dashboards, bass shakers, LEDs)

This script solves that by acting as a **UDP middleman**.

With this script you can havr multiple applications receiving UDP telemetry data at the same time. 

---

## How it works

1. **GT7 on PS5 broadcasts telemetry** over UDP (port `33740`)
2. This script listens on that port
3. Every telemetry packet is **copied and re-sent**
4. Each application listens on its **own UDP port**
5. No conflicts, no data loss

```
PS5 (GT7)
   │  UDP 33740
   ▼
[ GT7 UDP Multiplexer ]
   ├── SimTools (33740)
   ├── SimHub  (20777)
   └── GT7 Proxy (20888)
```

---

## What this script does NOT do

- Does NOT modify telemetry data  
- Does NOT decode GT7 packets  
- Does NOT replace SimHub or SimTools  
- Does NOT require PS5 configuration  

It is **only a UDP splitter / forwarder**.

---

## Requirements

- Windows 10 / 11  
- Python **3.9+**
- PS5 & PC on the **same LAN**
- GT7 running on PS5
- One or more of:
  - SimTools v3 (XSim)
  - SimHub v9+
  - GT7 Proxy (Python)

---

## Step 1 – Install Python

1. Download Python:  
   https://www.python.org/downloads/windows/

2. During installation:
   - Check **“Add Python to PATH”**
   - Click **Install**

3. Verify installation:
   ```cmd
   python --version
   ```

---

## Step 2 – Create the script

1. Create a new file named:

```
gt7_multiplexer.py
```

2. Paste **everything below** into the file:

```python
import socket

# ==============================
# CONFIGURATION
# ==============================

# GT7 PS5 telemetry broadcast port
GT7_LISTEN_PORT = 33740

# Destinations to forward telemetry to
DESTINATIONS = [
    ("127.0.0.1", 33740),  # SimTools / XSim
    ("127.0.0.1", 20777),  # SimHub
    # ("127.0.0.1", 20888),  # GT7 Proxy (optional)
]

# Optional: filter only your PS5 IP
PS5_IP = None  # Example: "192.168.0.153"

BUFFER_SIZE = 2048

# ==============================
# SOCKET SETUP
# ==============================

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
sock.bind(("", GT7_LISTEN_PORT))

print("GT7 UDP Multiplexer running")
print(f"Listening on UDP {GT7_LISTEN_PORT}")
print("Forwarding to:")
for d in DESTINATIONS:
    print(f"  -> {d[0]}:{d[1]}")

# ==============================
# MAIN LOOP
# ==============================

while True:
    try:
        data, addr = sock.recvfrom(BUFFER_SIZE)

        if PS5_IP and addr[0] != PS5_IP:
            continue

        for dest in DESTINATIONS:
            sock.sendto(data, dest)

    except KeyboardInterrupt:
        print("\nExiting...")
        break
    except Exception as e:
        print(f"Error: {e}")
```

---

## 🛠️ Step 3 – Configure the script

### DESTINATIONS

Each destination is:

```
(IP address, UDP port)
```

Typical setup:

| Software | Port |
|--------|------|
| SimTools / XSim | 33740 |
| SimHub | 20777 |
| GT7 Proxy | 20888 |

 Each application must listen on a **unique port**.

---

### PS5 IP Filtering (Recommended)

```python
PS5_IP = "192.168.0.153"
```

Disable filtering:

```python
PS5_IP = None
```

---

## Step 4 – Run the script

```cmd
python gt7_multiplexer.py
```

Leave the window open.

---

## Step 5 – Configure receiving software

### SimTools
- GT7 plugin enabled
- Listening on UDP **33740**

### SimHub
- Game: GT7
- UDP port: **20777**
- Live mode enabled
- Telemetry forwarding disabled

### GT7 Proxy (optional)
- Use port **20888**
- Do NOT bind directly to 33740

---

## Known behavior & quirks

### SimHub status flickers
This is normal with PS5 GT7 telemetry and UDP timing.  
Telemetry data remains correct.

---

## Firewall

Allow inbound UDP on:
- 33740
- All destination ports

---
