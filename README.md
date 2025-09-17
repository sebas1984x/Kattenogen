# Kattenoog – Gebruiksaanwijzing

Dit project bestaat uit twee Raspberry Pi’s en een Siemens S7-1215 PLC.  
Samen sturen ze de ogen en de kaak van een animatronische kat aan.

---

## 1. Overzicht Architectuur
- **Linker Pi (`kattenoog-left`)**
  - Stuurt het linkeroog
  - UDP-poort: 5005
  - Draait service: `eye-left.service`

- **Rechter Pi (`kattenoog-right`)**
  - Stuurt het rechteroog
  - UDP-poort: 5005
  - Draait service: `eye.service`
  - Stuurt de kaakservo via Dynamixel
  - UDP-poort: 5006
  - Draait service: `jaw.service`

- **PLC (Siemens S7-1215)**
  - Levert realtime waarden via Profinet/UDP
  - Houdt een Data Block (DB) bij met waarden voor ogen en kaak
  - DB moet **non-optimized** zijn (anders kloppen de byte-offsets niet)

---

## 2. PLC Data Structuur

### Linkeroog (standaard offsets in DB1)
- `DB1.DBD0`   → look_x  (horizontaal, -1 = uiterste links, +1 = uiterste rechts)
- `DB1.DBD4`   → look_y  (verticaal, -1 = boven, +1 = onder)
- `DB1.DBD8`   → pupil   (0 = klein, 1 = groot)
- `DB1.DBD12`  → lid     (0 = open, 1 = dicht)

### Rechteroog (standaard offsets in DB1)
- `DB1.DBD100` → look_x
- `DB1.DBD104` → look_y
- `DB1.DBD108` → pupil
- `DB1.DBD112` → lid

### Kaak
- Eén byte (0–255), verstuurd naar UDP-poort 5006 van de rechter Pi
- Wordt in software vertaald naar graden tussen `min_deg` en `max_deg` (standaard 200°–245°)

### Iris-intensiteit (optioneel)
Naast de standaard 8 waarden kan de PLC ook een uitgebreid pakket van 10 bytes sturen.
- Byte 9 = `Liris` (linker iris)
- Byte 10 = `Riris` (rechter iris)

Waarde 0..255 bepaalt de sterkte van de radiale iris-gradient:
- 0   = vlak/zwak contrast
- 255 = sterke gradient

Als deze waarden niet worden meegestuurd, gebruiken de ogen standaard een middenwaarde (≈ 0.5).

### Belangrijk
- Waarden mogen als **REAL** (-1..1 of 0..1) of als **INT/byte** (0..255) worden gestuurd.  
  De Raspberry Pi’s schalen dit automatisch.
- Zorg dat de DB in TIA Portal op **non-optimized** staat.

---

## 3. PLC Voorbeelden
- **Knipoog** → zet `lid` van één oog op 1.0 (dicht) en laat de ander op 0.0 (open).  
- **Pupil groter/kleiner** → schrijf 0..1 (REAL) of 0..255 (byte) naar `pupil`.  
- **Oogbeweging** → stuur -1..+1 (REAL) of 0..255 (byte, 128 = midden) naar `look_x` en `look_y`.  
- **Kaakbeweging** → schrijf een waarde 0..255 naar de variabele die via UDP naar poort 5006 van de rechter Pi wordt gestuurd.  
- **Iris laten pulseren** → stuur sinus (0..255) naar `Liris` en `Riris`.

---

## 4. Raspberry Pi Software

### 4.1 Scripts
Op beide Pi’s staan de scripts in `/home/cat/kattenoog/`:
- `kattenoog_plc_udp_oneeye.py` → oogweergave
- `jaw_udp_dynamixel.py` → kaakservo (alleen rechter Pi)
- `eyes_send.py` en `jaw_send.py` → testtools

### 4.2 Systemd Services
- Linker Pi → `eye-left.service`  
- Rechter Pi → `eye.service` en `jaw.service`  

Commando’s:  
```
systemctl status eye.service
systemctl status jaw.service
sudo systemctl restart eye.service
sudo systemctl restart jaw.service
journalctl -u eye.service -f
journalctl -u jaw.service -f
```

### 4.3 Benodigde pakketten

#### Beide Pi’s (ogen)
- python3
- python3-pip
- pygame
- git
- python-snap7 (optioneel, alleen nodig als je direct vanuit de Pi met de PLC wilt testen)

#### Rechter Pi (kaak)
- Alles hierboven
- dynamixel-sdk (voor aansturing Dynamixel servo’s via USB/RS485)

#### Installatie
```bash
sudo apt update
sudo apt install -y python3 python3-pip git
pip3 install pygame
pip3 install dynamixel-sdk   # alleen rechter Pi
pip3 install python-snap7    # optioneel
```

#### Handigheid: requirements.txt
In de repo kan een `requirements.txt` staan, die installeer je met:
```bash
pip3 install -r requirements.txt
```

---

## 5. Installatie & Deployment

### Eerste keer op een Pi
```bash
git clone git@github.com:sebas1984x/kattenoog-left.git /home/cat/kattenoog   # linker Pi
git clone git@github.com:sebas1984x/kattenoog-right.git /home/cat/kattenoog  # rechter Pi

sudo cp services/*.service /etc/systemd/system/
sudo systemctl daemon-reload

sudo systemctl enable --now eye.service
sudo systemctl enable --now jaw.service   # alleen rechter Pi
```

### Updates
```bash
cd /home/cat/kattenoog
git pull
sudo systemctl restart eye.service
sudo systemctl restart jaw.service   # alleen rechter Pi
```

---

## 6. Netwerk & Poorten
- Ogen luisteren op UDP-poort **5005**  
- Kaak luistert op UDP-poort **5006**  
- Controleer met:  
  ```bash
  sudo netstat -anu | grep 500
  ```

### 6.1 Vast IP-adres instellen (aanbevolen)
```bash
sudo nano /etc/dhcpcd.conf

interface eth0
static ip_address=192.168.0.101/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.1
# voorbeeld: linker Pi = 192.168.0.101, rechter Pi = 192.168.0.102

sudo reboot
ip addr show eth0
ping <PLC-IP>
```

---

## 7. Troubleshooting
- **Geen beeld van oog** → check `eye.service`, logs bekijken.  
- **Kaak beweegt niet** → check `jaw.service`, USB en logs.  
- **PLC stuurt maar geen reactie** → DB non-optimized, datatypes consistent.  

### Netwerk (UTP / eth0)
- `NO-CARRIER` betekent: geen fysieke link.  
- Controle: kabel goed ingeplugd, link-ledjes, andere kabel/poort testen.  
- Handmatig IP instellen als er geen DHCP is:  
  ```bash
  sudo ip addr add 192.168.0.101/24 dev eth0
  sudo ip link set eth0 up
  ```

---

## 8. Extra aandachtspunten
- Services pas starten na netwerk:  
  ```
  After=network-online.target
  Wants=network-online.target
  ```
- Firewall → poorten 5005 en 5006 moeten open.  
- Dynamixel USB → standaard `/dev/ttyUSB0`, evt. vaste naam via udev-rule.  
- Backups → alles staat in GitHub. Nieuwe Pi = `git clone` + services kopiëren.

---

## 9. Dynamixel XM430 limieten instellen

De kaak wordt aangedreven door een Dynamixel XM430-servo.  
Standaard kan deze 0–360° draaien, maar voor de kaak wordt een kleinere veilige range gebruikt (bijv. 200–245°).

### Via Dynamixel Wizard (aanbevolen)
1. Sluit de servo aan met U2D2 of USB2Dynamixel.
2. Start **Dynamixel Wizard 2.0**.
3. Scan motor (ID=1, baud=57600).
4. Operating Mode = Position Control.
5. Stel in:  
   - Min Position Limit ≈ 2275 ticks (200°)  
   - Max Position Limit ≈ 2788 ticks (245°)  
6. Opslaan naar EEPROM.

### Via Python (dynamixel-sdk)
```python
from dynamixel_sdk import *

ADDR_MIN_POS_LIMIT = 52
ADDR_MAX_POS_LIMIT = 48
DXL_ID = 1
BAUDRATE = 57600

ph = PortHandler('/dev/ttyUSB0')
ph.openPort(); ph.setBaudRate(BAUDRATE)
pk = PacketHandler(2.0)

def deg_to_tick(deg): 
    return int(round((deg % 360) * 4096 / 360))

min_tick = deg_to_tick(200)
max_tick = deg_to_tick(245)

pk.write4ByteTxRx(ph, DXL_ID, ADDR_MIN_POS_LIMIT, min_tick)
pk.write4ByteTxRx(ph, DXL_ID, ADDR_MAX_POS_LIMIT, max_tick)

print(f"Set limits: {min_tick}–{max_tick} ticks")
```

---

## 10. Contact & Onderhoud
- Repos:  
  - [kattenoog-left](https://github.com/sebas1984x/kattenoog-left)  
  - [kattenoog-right](https://github.com/sebas1984x/kattenoog-right)
- Services staan in `services/` map.  
- Problemen met GitHub → controleer SSH key op de Pi.

---
