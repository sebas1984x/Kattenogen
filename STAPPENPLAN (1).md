# Stappenplan Kattenkaak & Ogen

## 1. Oude kaak verwijderen
- Stop en disable de oude service:
  ```bash
  sudo systemctl stop jaw_udp_dynamixel.service
  sudo systemctl disable jaw_udp_dynamixel.service
  ```
- Verwijder het oude script:
  ```bash
  rm -f /home/cat/kattenoog/jaw_udp_dynamixel.py
  ```

## 2. Oog-bridge schoonmaken
- Pas `plc_to_udp_bridge.py` aan en zet:
  ```python
  "Jaw": None
  ```
- Zorg dat de service `plc_to_udp_bridge.service` draait zodat de ogen via UDP (poort 5005) gevoed blijven.

## 3. Nieuwe kaak installeren
- Plaats `jaw_snap7_dynamixel.py` in:
  ```
  /home/cat/kattenoog/
  ```
- Maak systemd-unit `/etc/systemd/system/kattenkaak.service` aan:

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

- Activeer de service:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable --now kattenkaak.service
  ```

## 4. PLC aanpassen
- Maak twee niet-geoptimaliseerde DB’s:
  - **DB_Command**: setpoints (positie, snelheid, acceleratie, invert, enable, cmd_seq).
  - **DB_Status**: feedback (positie, snelheid, ack_seq, hw_ok, sw_ok, timestamp).
- Zorg dat de PLC bij elk nieuw commando `cmd_seq` ophoogt.

## 5. Testen
- Reboot de Pi.
- Controleer services:
  ```bash
  systemctl status kattenkaak.service
  systemctl status plc_to_udp_bridge.service
  ```
- In de PLC checken dat:
  - `ack_seq` volgt op `cmd_seq`.
  - `ts_ms` verandert.
  - De kaak beweegt volgens setpoints.
