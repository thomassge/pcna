# VLAN-Konfiguration Schritt für Schritt (mit Erklärungen)

Dieses Dokument führt Schritt für Schritt durch die Konfiguration eines VLAN-Setups mit 4 Hosts und einem Router. Die Befehle gelten für Debian-basierte Linux-Systeme.

---

## 🧰 Voraussetzungen

- Terminalzugang zu jedem Gerät
- VLAN-fähige Netzwerkkarten
- Root- oder sudo-Zugriff

---

## 🔹 Vorbereitung: Netzwerk-Interface prüfen

```bash
ip link
```

👉 Zeigt alle Netzwerkschnittstellen an (z. B. `eth0`). Wir benötigen den Namen der physischen Schnittstelle, um darauf VLANs aufzubauen.

---

## 🔹 Host 1 – VLAN 10 – IP: 192.168.10.10

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
```
➡️ Erstellt ein virtuelles VLAN-Interface (`eth0.10`) auf Basis von `eth0` mit der VLAN-ID 10.

```bash
sudo ip addr add 192.168.10.10/24 dev eth0.10
```
➡️ Weist diesem Interface die IP-Adresse 192.168.10.10 mit Subnetzmaske 255.255.255.0 zu.

```bash
sudo ip link set dev eth0.10 up
```
➡️ Aktiviert das VLAN-Interface.

---

## 🔹 Host 2 – VLAN 10 – IP: 192.168.10.11

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 192.168.10.11/24 dev eth0.10
sudo ip link set dev eth0.10 up
```

---

## 🔹 Host 3 – VLAN 20 – IP: 192.168.20.10

```bash
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 192.168.20.10/24 dev eth0.20
sudo ip link set dev eth0.20 up
```

---

## 🔹 Host 4 – VLAN 20 – IP: 192.168.20.11

```bash
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 192.168.20.11/24 dev eth0.20
sudo ip link set dev eth0.20 up
```

---

## 🔹 Router 1 – beide VLANs anbinden + Routing vorbereiten

### VLAN 10 auf Router einrichten

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 192.168.10.1/24 dev eth0.10
sudo ip link set dev eth0.10 up
```

### VLAN 20 auf Router einrichten

```bash
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 192.168.20.1/24 dev eth0.20
sudo ip link set dev eth0.20 up
```

### IP-Forwarding aktivieren (damit der Router Pakete weiterleitet)

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

➡️ Schaltet das Weiterleiten von IP-Paketen zwischen Interfaces ein.

---

## ✅ Tests

### Ping innerhalb eines VLANs
```bash
ping 192.168.10.11  # Host 1 → Host 2
ping 192.168.20.11  # Host 3 → Host 4
```

### Ping Router von Host
```bash
ping 192.168.10.1   # Host 1 → Router
ping 192.168.20.1   # Host 3 → Router
```

### Ping zwischen VLANs (nur wenn Routing aktiv)
```bash
ping 192.168.20.10  # Host 1 → Host 3
```

---

## 🔍 Wireshark-Tipp

Filter: `vlan`

➡️ Damit sieht man, ob Pakete korrekt getaggt sind (802.1Q Header).

---

**Ende des Setups – Netzwerk ist jetzt logisch getrennt & bereit für Routing**


---

## 🧭 Optional: DHCP-Server einrichten (für automatische IP-Vergabe)

Statt die IPs statisch zu setzen, kannst du einen DHCP-Server am Router konfigurieren. Die Clients erhalten dann ihre IP-Adresse automatisch – das ist praxisnäher.

### DHCP-Server installieren (auf dem Router)
```bash
sudo apt update
sudo apt install isc-dhcp-server
```

### DHCP-Konfiguration bearbeiten
```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Inhalt (für beide VLANs):
```conf
subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.100 192.168.10.200;
  option routers 192.168.10.1;
  option domain-name-servers 8.8.8.8;
}

subnet 192.168.20.0 netmask 255.255.255.0 {
  range 192.168.20.100 192.168.20.200;
  option routers 192.168.20.1;
  option domain-name-servers 8.8.8.8;
}
```

### Interface-Bindung konfigurieren
```bash
sudo nano /etc/default/isc-dhcp-server
```
Zeile anpassen:
```bash
INTERFACESv4="eth0.10 eth0.20"
```

### DHCP-Server starten
```bash
sudo systemctl restart isc-dhcp-server
```

---

### Clients (z. B. Host 1) – DHCP-Adresse anfordern
```bash
sudo dhclient -v eth0.10
```

Damit wird automatisch eine passende Adresse bezogen, wenn der Server richtig läuft.

---

