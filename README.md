<img width="861" height="525" alt="Gt7udpmulti" src="https://github.com/user-attachments/assets/f6a729d8-fdbf-4cfb-a688-097f93dccbca" />
<br>
<br>

## GT7 UDP Multiplexer  
**Gran Turismo 7 (PS5) Telemetry Splitter for SimTools, SimHub & GT7 Proxy**
Receives telemetry broadcasts from PS5 GT7 and forwards them to multiple local applications (SimTools, SimHub, etc.), enabling multiple simulation applications requiring udp streams from GT7 to work simultaneously.

## What is this?

**GT7 UDP Multiplexer** is a small, dependency-free Python script that allows **Gran Turismo 7 on PS5** to send telemetry data to **multiple PC applications on the same pc at the same time**. A hardened executable (*.exe) version is currently in the works, along with the ability to add additional applications and modify IP addresses and port numbers via a config file. 

Normally, GT7 telemetry can only be received by **one application per UDP port**, which causes conflicts when using applications such as:

- SimTools using GT7 Proxy
- SimHub (dashboards, bass shakers, LEDs)

This script solves that by acting as a **UDP middleman**.

With this script, you can have multiple applications receiving UDP telemetry data at the same time. 

---

## How it works

1. **GT7 on PS5 broadcasts telemetry** over UDP (port `33740`)
2. This script listens on that port
3. Every telemetry packet is **copied and re-sent** to multiple predefined UDP ports on the local machine
4. Each application listens on its **own UDP port**
5. No conflicts, no data loss

```
PS5 (GT7)
   │  UDP 33740
   ▼
[ GT7 UDP Multiplexer ]
   ├── SimHub  (20777)
   └── GT7 Proxy (20888)> SimTools
```

---

## What this script does NOT do

- Does NOT modify telemetry data  
- Does NOT decode GT7 packets  
- Does NOT replace SimHub or SimTools   

This application acts purely as a UDP splitter/forwarder. It forwards GT7 telemetry packets unchanged. While it does not spoof the origin IP address, it does change what the receiving apps see as the source:

**Incoming packet**
-Source IP: PS5 (i.e. in my setup I have a static IP of 192.168.0.153)
-Source port: GT7 port
-Destination: your PC (:33740)

**Outgoing packet (forwarded)**
-Source IP: your PC’s IP (e.g. loopback 127.0.0.1 or LAN IP)
-Source port: 33740
-Destination: SimHub / GT7Proxy

The payload is unchanged, but the UDP header is rebuilt. This is different from SimHub’s UDP forwarder, which modifies the telemetry payload before re-broadcasting it. That modification can break applications that expect the original GT7 packet structure, such as GT7 Proxy, resulting in errors like:
```cmd
Exception: unpack requires a buffer of 316 bytes 
```
By forwarding the packets unchanged it prevents those issues from occurring. 

---

## Requirements

- Python **3.9+**
- PS5 & PC on the **same LAN** (preferably wired to reduce latency)
- GT7 running on PS5
- One or more of:
  - SimTools v3 (XSim) with GT7 Proxy 
  - SimHub v9+

---

## Usage Instructions (Python Script)

**Step 1 – Install Python**

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

**Step 2 – Create the script**

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
    ("127.0.0.1", 20777),  # SimHub
    ("127.0.0.1", 20888),  # GT7 Proxy
    #add others here
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

**Step 3 – Configure the script**

### DESTINATIONS

Each destination is:

```
(IP address, UDP port)
```

Typical setup:

| Software | Port |
|--------|------|
| SimHub | 20777 |
| GT7 Proxy | 20888 |

 Each application must listen on a **unique port**.

---

### PS5 IP Filtering (Recommended)
This should be set to the same IP as you4 PS5. It is recommended to set your PS5's IP as a static IP address either on your PS5 or as a reserved address in your modem/router. 

```python
PS5_IP = "192.168.0.153" 
```

Disable filtering:

```python
PS5_IP = None
```

---

## Step 4 – Run the script. You can either do this by double clicking the python script in Windows, or via Command Prompt (press Win+R buttons on your keyboard, then type 'cmd' into the run box). In the black window that opens (command prompt), run the python script using: 

```cmd
python gt7_multiplexer.py
```
Note: you will need to either navigate to the directory where the gt7_multiplexer.py is located first, or include the full path in the command (i.e. python c:\gt7multiplexer\gt7_multiplexer.py)

Once the script is running, you may receive a popup from Windows Firewall requesting authorisation for the script. Click allow to continue. 

Leave the window open.

---

## Step 5 – Configure receiving software

**Always start in this order:**
1️⃣ Python multiplexer
Copy code
2️⃣ GT7Proxy
3️⃣ SimTools
4️⃣ SimHub
5️⃣ Start GT7 on PS5

**GT7 Proxy**
- Use port **20888**
- Do NOT bind directly to 33740
- Run the exe using the following command (changing the ps5-ip to the IP address of your PS5):
```cmd
gt7proxy.exe --listen-port 20888 --ps5-ip 192.168.0.153
```

**SimTools**
- GT7 plugin enabled
  
**SimHub**
- Game: GT7
- UDP port: **20777**
- Live mode enabled
- Telemetry forwarding disabled
- 
---

## Known Issues

### SimHub status flickers between 'Ready' and 'Waiting for Data'
This is normal with PS5 GT7 telemetry and UDP timing. I am currently working on a solution for this- although it may be an issue inherent with SimHub.   


---

## Firewall

Allow inbound UDP on:
- 33740
- All destination ports

---
