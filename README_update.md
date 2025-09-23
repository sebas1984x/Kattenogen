# Kattenkaak & Ogen – PLC ↔ Raspberry Pi ↔ Dynamixel

Deze repository bevat de Python-scripts en systemd-services om de **ogen** en **kaak** van de Kat aan te sturen vanuit een Siemens PLC via een Raspberry Pi.

---

## 📦 Overzicht

- **`plc_to_udp_bridge.py`**  
  Leest de oogwaardes uit de PLC (DB) en stuurt ze via UDP naar de eye-daemon.  
  👉 Stuurt *alleen* nog de ogen, **niet** de kaak.

- **`jaw_snap7_dynamixel.py`**  
  Stuurt de Dynamixel-kaak direct via Snap7 en seriële poort.  
  👉 Kaak gaat dus **niet meer via UDP**, maar via DB_Command/DB_Status.

- **Systemd-services**  
  - `kattenkaak.service` → start de kaakcontroller automatisch.  
  - `plc_to_udp_bridge.service` → laat de ogen doorsturen naar UDP.

---

## ⚙️ PLC-DB Layouts

### DB_Command (PLC → Pi → Dynamixel)

```
INT   jaw_pos_sp   // offset 0   | Setpoint positie (0..1000)
INT   jaw_vel_sp   // offset 2   | Gewenste snelheid (0..1000)
INT   jaw_acc_sp   // offset 4   | Gewenste acceleratie (0..1000)
BOOL  jaw_invert   // byte 6.0   | Mapping omkeren
BOOL  enable       // byte 6.1   | Interlock / noodstop
UINT  cmd_seq      // offset 8   | ++ bij nieuw commando
```

### DB_Status (Pi → PLC)

```
INT   jaw_pos_fb   // offset 0   | Feedback positie (0..1000)
INT   jaw_vel_fb   // offset 2   | Feedback snelheid (0..1000)
UINT  ack_seq      // offset 4   | Laatst verwerkt cmd_seq
BOOL  hw_ok        // byte 6.0   | Servo reageert, torque on
BOOL  sw_ok        // byte 6.1   | Script loopt
DWORD ts_ms        // offset 8   | Timestamp (ms) van de Pi
```

---

## 🚀 Installatie op de Pi

### 1. Benodigdheden installeren
```bash
sudo apt update
sudo apt install python3-pip
pip3 install python-snap7 dynamixel-sdk
```

### 2. User toegang geven tot seriële poort
```bash
sudo usermod -a -G dialout cat
```
Daarna uit- en inloggen (of rebooten).

### 3. Scripts plaatsen
Plaats de scripts in `/home/cat/kattenoog/`:
- `jaw_snap7_dynamixel.py`
- `plc_to_udp_bridge.py`

### 4. Systemd-service voor de kaak
Maak `/etc/systemd/system/kattenkaak.service` aan:

```ini
[Unit]
Description=Kattenkaak Servo (snap7→Dynamixel)
After=network-online.target
Wants=network-online.target

[Service]
User=cat
Group=dialout
WorkingDirectory=/home/cat/kattenoog
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 /home/cat/kattenoog/jaw_snap7_dynamixel.py \
  --plc_host 2.100.1.243 --rack 0 --slot 1 --db_cmd 1 --db_sts 2 \
  --dev /dev/ttyUSB0 --id 1 --baud 57600 \
  --min_deg 200 --max_deg 245 \
  --prof_vel_default 50000 --prof_acc_default 30000 \
  --hz 100 --watchdog_20ms 3
Restart=always
RestartSec=1
KillSignal=SIGINT
TimeoutStopSec=5

[Install]
WantedBy=multi-user.target
```

Activeer de service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kattenkaak.service
```

### 5. Systemd-service voor de ogen
Laat `plc_to_udp_bridge.service` draaien, maar met **Jaw = None** in de offset-tabel. Deze service blijft ogen via UDP naar poort 5005 sturen.

---

## ✅ Controleren

- Status van de kaak-service:
  ```bash
  systemctl status kattenkaak.service
  ```
- Live logs bekijken:
  ```bash
  journalctl -u kattenkaak.service -f
  ```
- Zien welke services draaien:
  ```bash
  systemctl list-units --type=service | grep katten
  ```

---

## 🔎 Dataflow

```
PLC
 │
 │  DB_Command + DB_Status
 ▼
Raspberry Pi
 ├─ jaw_snap7_dynamixel.py  → Dynamixel servo (kaak) via /dev/ttyUSB0
 └─ plc_to_udp_bridge.py    → Ogen via UDP naar eye-daemon (poort 5005)
```

---

## 📝 Samenvatting

- **Ogen** → via `plc_to_udp_bridge.py` → UDP (poort 5005).  
- **Kaak** → via `jaw_snap7_dynamixel.py` → direct Snap7 met DB’s.  
- Oude `jaw_udp_dynamixel.py` is niet meer nodig.
