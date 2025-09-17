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

Per oog zijn 4 variabelen nodig (REAL of INT/byte).

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

### Belangrijk
- Waarden mogen als **REAL** (-1..1 of 0..1) of als **INT/byte** (0..255) worden gestuurd.  
  De Raspberry Pi’s schalen dit automatisch.
- Zorg dat de DB in TIA Portal op **non-optimized** staat.

---

## 3. PLC Voorbeelden

- **Knipoog**  
  Zet `lid` van één oog op 1.0 (dicht) en laat de ander op 0.0 (open).

- **Pupil groter/kleiner**  
  Schrijf 0..1 (REAL) of 0..255 (byte) naar `pupil`.

- **Oogbeweging**  
  Stuur -1..+1 (REAL) of 0..255 (byte, 128 = midden) naar `look_x` en `look_y`.

- **Kaakbeweging**  
  Schrijf een waarde 0..255 naar de variabele die via UDP naar poort 5006 van de rechter Pi wordt gestuurd.  
  0 = minimale hoek (`min_deg`), 255 = maximale hoek (`max_deg`).

---

## 4. Raspberry Pi Software

### Scripts
Op beide Pi’s staan de scripts in `/home/cat/kattenoog/`:
- `kattenoog_plc_udp_oneeye.py` → oogweergave
- `jaw_udp_dynamixel.py` → kaakservo (alleen rechter Pi)
- `eyes_send.py` en `jaw_send.py` → testtools

### Systemd services
De Pi’s starten de scripts automatisch bij boot via systemd.

- Linker Pi:
  - `eye-left.service` (alias: `eye.service`)

- Rechter Pi:
  - `eye.service`
  - `jaw.service`

Status bekijken:
```
systemctl status eye.service
systemctl status jaw.service
```

Herstarten:
```
sudo systemctl restart eye.service
sudo systemctl restart jaw.service
```

Logs volgen:
```
journalctl -u eye.service -f
journalctl -u jaw.service -f
```

---

## 4.1 Benodigde pakketten

Voor een correcte werking moeten de volgende pakketten aanwezig zijn:

### Beide Pi’s (ogen)
- python3
- python3-pip
- pygame
- git
- python-snap7 (optioneel, alleen nodig als je direct vanuit de Pi met de PLC wilt testen)

### Rechter Pi (kaak)
- Alles hierboven
- dynamixel-sdk (voor aansturing Dynamixel servo’s via USB/RS485)

### Installatie
```bash
# basis
sudo apt update
sudo apt install -y python3 python3-pip git

# pygame
pip3 install pygame

# dynamixel (alleen rechter Pi)
pip3 install dynamixel-sdk

# optioneel: snap7
pip3 install python-snap7
```

### Handigheid: requirements.txt
In de repo kan een `requirements.txt` staan, die installeer je met:
```bash
pip3 install -r requirements.txt
```

## 5. Installatie & Deployment

### Eerste keer op een Pi
1. Repos klonen:
   ```
   git clone git@github.com:sebas1984x/kattenoog-left.git /home/cat/kattenoog   # op linker Pi
   git clone git@github.com:sebas1984x/kattenoog-right.git /home/cat/kattenoog  # op rechter Pi
   ```
2. Services kopiëren:
   ```
   sudo cp services/*.service /etc/systemd/system/
   sudo systemctl daemon-reload
   ```
3. Services activeren:
   ```
   sudo systemctl enable --now eye.service
   sudo systemctl enable --now jaw.service   # alleen op rechter Pi
   ```

### Updates
1. Code ophalen:
   ```
   cd /home/cat/kattenoog
   git pull
   ```
2. Services herstarten:
   ```
   sudo systemctl restart eye.service
   sudo systemctl restart jaw.service   # alleen op rechter Pi
   ```

---

## 6. Netwerk & Poorten

- Ogen luisteren op UDP-poort **5005**
- Kaak luistert op UDP-poort **5006**
- Controleer of UDP actief is:
  ```
  sudo netstat -anu | grep 500
  ```

### 6.1 Vast IP-adres instellen (aanbevolen)

Voor stabiele communicatie met de PLC moeten beide Pi’s een vast IP-adres krijgen.

1. Open het configuratiebestand:
   ```
   sudo nano /etc/dhcpcd.conf
   ```

2. Voeg onderaan toe (pas IP’s en gateway aan naar je netwerk):
   ```
   interface eth0
   static ip_address=192.168.0.101/24
   static routers=192.168.0.1
   static domain_name_servers=192.168.0.1

   # voorbeeld: linker Pi = 192.168.0.101, rechter Pi = 192.168.0.102
   ```

3. Opslaan en rebooten:
   ```
   sudo reboot
   ```

4. Controle:
   ```
   ip addr show eth0
   ping <PLC-IP>
   ```

---

## 7. Troubleshooting

- **Geen beeld van oog**
  - Check of service draait: `systemctl status eye.service`
  - Bekijk logs: `journalctl -u eye.service -f`
  - Controleer of PLC waarden stuurt (via TIA monitor)

- **Kaak beweegt niet**
  - Check of service draait: `systemctl status jaw.service`
  - Bekijk logs: `journalctl -u jaw.service -f`
  - Controleer of de waarde in PLC echt naar UDP-poort 5006 gaat
  - Controleer of Dynamixel motor is aangesloten op `/dev/ttyUSB0`

- **PLC stuurt waarden, maar geen beweging**
  - Verzeker je dat DB op **non-optimized** staat
  - Controleer of je REAL of byte gebruikt (beide mogen, maar consistent instellen)

---

## 8. Extra aandachtspunten

- **Netwerkvolgorde**  
  Zet in de `.service` bestanden:
  ```
  After=network-online.target
  Wants=network-online.target
  ```
  Zo starten de scripts pas als het netwerk actief is.

- **Firewall**  
  Als later een firewall wordt gebruikt: poorten 5005 (ogen) en 5006 (kaak) moeten open blijven.

- **USB device voor Dynamixel**  
  Controleer dat de servo altijd op `/dev/ttyUSB0` zit. Een udev-rule voor een vaste naam kan handig zijn.

- **Backups**  
  Alle services en scripts staan in GitHub. Nieuwe Pi opzetten = `git clone` en services kopiëren.

---

## 9. Contact & Onderhoud

- Repos:  
  - [kattenoog-left](https://github.com/sebas1984x/kattenoog-left)  
  - [kattenoog-right](https://github.com/sebas1984x/kattenoog-right)

- Alle systemd servicebestanden staan in de `services/` map van de repos.  
- Bij problemen met GitHub toegang: controleer SSH key op de Pi.

---


- Alle systemd servicebestanden staan in de `services/` map van de repos.  
- Bij problemen met GitHub toegang: controleer SSH key op de Pi.

---
