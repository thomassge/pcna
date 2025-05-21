# PCNA Lab Exercise 1 – Team 4

**Kurs:** Practical Computer Networks and Applications (SoSe 2025)**Teamnummer:** 4**Mitglieder:** (bitte Namen und Matrikelnummern eintragen)

---

## ✅ 1. Kommandozeilen-Tools für Netzwerkanalyse

Wir machen uns mit folgenden CLI-Tools vertraut:

- `ip`: Netzwerkinterfaces anzeigen & konfigurieren
- `ping`: ICMP-Echo zur Erreichbarkeit
- `traceroute`: Pfad zu einem Host über Routern anzeigen
- `netstat` / `ss`: Netzwerksockets anzeigen
- `arp`: ARP-Tabelle anzeigen
- `nc`: Netcat (Verbindung zu Ports testen / Strings senden)
- `nmap`: Netzwerk-/Portscanner
- `ethtool`: Interface-Infos wie Geschwindigkeit, Status
- `dig` / `nslookup`: DNS-Namen auf IP auflösen
- `sysctl`: Systemparameter anzeigen/setzen (z. B. IP-Forwarding)

**Notizen zu getesteten Befehlen:**

```bash
ip addr
ping 8.8.8.8
ss -tuln
arp -n
netstat -rn
```

---

## ✅ 2. Netzwerk aufbauen

### a) Topologie & IP-Konfiguration (Gruppe 4)

#### IP-Adressen:

- Host 1: 192.168.4.10/24
- Host 2: 192.168.4.20/24
- Host 3: 192.168.4.30/24
- Router 1 (eth0): 192.168.4.1/24
- Router 1 (eth1): 10.16.0.4/16
- Router 2: 10.16.0.200/16

#### Wichtig:

Interface `eno2` deaktivieren (für Lab-Setup):

```bash
sudo ip link set eno2 down
```

### b) Konfiguration per CLI

- **IP-Adressen setzen**: `ip addr add ...`
- **Interface aktivieren**: `ip link set ... up`
- **Standard-Gateway setzen**: `ip route add default via ...`
- **IP-Forwarding auf Router 1 aktivieren**:

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### c) Netzwerk testen

- ICMP-Test:

```bash
ping 192.168.4.1          # von Host zu Router 1
ping 10.16.0.200          # von Host zu Router 2
ping google.de            # DNS & Internettest
```

- Routing sichtbar machen:

```bash
traceroute -I ict-ffm.de
```

- DNS-Test:

```bash
nslookup medusa.fh-frankfurt.de
dig google.de
```

**Dokumentation aller Schritte inkl. Terminalausgabe!**

---

## ✅ 3. Netzwerkadministration testen

Hier testen wir Tools aus Kapitel 1 im eigenen Netzwerk. Wichtig ist, jeweils Befehl + Output zu dokumentieren!

### Aufgaben:

1. IP-Adresse von Host 1 anzeigen:

```bash
ip addr show enp1s0f0
```

2. Routing-Tabelle auf Host 1 anzeigen:

```bash
ip route show
```

3. ARP-Tabelle auf Router 1 anzeigen:

```bash
arp -n
```

4. Echo-Request mit `netcat` an Router 2 schicken:

```bash
curl -d "Hello!" 10.16.0.200
```

5. Antwort dokumentieren (bei Erfolg Ausgabe sichtbar).
6. Offene TCP-Ports auf Host 1 anzeigen:

```bash
ss -tuln
```

7. UDP-Ports anzeigen:

```bash
ss -uln
```

8. Ports 1–1023 auf Router 1 scannen:

```bash
nmap -p 1-1023 192.168.4.1
```

9. Alle Dienste im privaten Netz scannen:

```bash
nmap -sV 192.168.4.0/24
```

10. Scan des externen Netzes:

```bash
nmap -sV 10.16.0.0/16
```

---

## ✅ 4. ICMP überwachen mit Wireshark

### Vorbereitung:

- Wireshark auf folgenden Interfaces starten:
  - Junction 1: Host 1 (eth0)
  - Junction 2: Router 1 (eth0)
  - Junction 3: Router 1 (eth1)

### Filter setzen:

```bash
icmp
```

### ICMP analysieren:

- Ping von Host 1 zu Router 2 starten:

```bash
ping 10.16.0.200
```

- Analyse:
  - MAC-Adresse (Layer 2)
  - IP-Adresse (Layer 3)
  - TTL, ICMP-Typ & Code

### Zusätzliche Tests:

```bash
ping -s 65507 192.168.4.1
ping -M do -s 1473 192.168.4.1
```

- Beobachten, was bei Fragmentierung passiert!

---

## ✅ 5. Protokolle (TCP, HTTP, ARP) beobachten

### a) Vorbereitung:

```bash
sudo ip neigh flush all       # ARP-Cache leeren
```

### b) HTTP-Traffic erzeugen:

```bash
curl ict-ffm.de
```

### c) Wireshark-Filter:

```bash
arp
tcp
http
```

### Analysepunkte:

- ARP-Request: Wer hat IP XY?
- TCP 3-Way Handshake: SYN, SYN-ACK, ACK
- HTTP GET Request + Antwort
- Reihenfolge der Protokolle

### d) Message Sequence Chart (MSC)

Beispielstruktur (bitte IPs & Ports aus eurem Lab nehmen):

```
192.168.4.10:12345 --SYN--> 10.16.0.200:80
192.168.4.10:12345 <--SYN+ACK-- 10.16.0.200:80
192.168.4.10:12345 --ACK--> 10.16.0.200:80
```

### e) OSI-Stack + Overhead

Füllt die folgenden Informationen für ein HTTP-Paket:

- Ethernet Header: 14 Byte
- IP Header: 20 Byte
- TCP Header: 20 Byte
- HTTP Payload: z. B. 512 Byte
- Ethernet Trailer (FCS): 4 Byte

Berechnung:

- **Overhead (Bytes)** = Header + Trailer = 14 + 20 + 20 + 4 = 58
- **Overhead-Ratio (%)** = (58 / (58 + 512)) * 100 ≈ 10.2 %

---

## 🧾 Übersicht wichtiger Befehle

| Befehl | Beschreibung |
|--------|--------------|
| `ip addr show` | Zeigt die Netzwerkkonfiguration eines Interfaces |
| `ip route show` | Zeigt die aktuelle Routing-Tabelle |
| `ip link set eno2 down` | Deaktiviert das Lab-Interface `eno2` |
| `ping <IP>` | Prüft, ob ein Host erreichbar ist |
| `traceroute -I <domain>` | Zeigt die Route zu einem Ziel (mit ICMP) |
| `ss -tuln` / `netstat -tuln` | Zeigt aktive Ports (TCP/UDP) |
| `arp -n` | Zeigt die ARP-Tabelle ohne Namensauflösung |
| `nc <IP> <Port>` | Netcat – sendet Daten an einen Port |
| `nmap -p 1-1023 <IP>` | Scannt Ports im Bereich 1–1023 |
| `nmap -sV <Netz>` | Scannt alle Dienste im angegebenen Netz |
| `dig <domain>` | DNS-Abfrage mit vielen Details |
| `nslookup <domain>` | Einfache DNS-Abfrage |
| `curl <URL>` | Sendet HTTP-Anfrage an eine Webseite |
| `ip neigh flush all` | Leert den ARP-Cache |
| `echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward` | Aktiviert IP-Weiterleitung auf dem Router |

---

## 🔹 Anhang / Screenshots / Terminalausgaben

(Screenshots von Wireshark & Terminalausgaben hier einfügen oder verlinken)

---

> ✉ Tipp: Speichert eure Ergebnisse frühzeitig in einem gemeinsamen Git-Repo oder Cloud-Ordner, damit ihr alles bei der Demonstration griffbereit habt!