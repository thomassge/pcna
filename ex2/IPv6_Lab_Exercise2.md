
# IPv6 Lab Exercise Sheet 2 - Solution & Documentation

## Systemvoraussetzungen & Hinweis
Alle Befehle sind für **Linux** (getestet unter Ubuntu) geeignet und benötigen **root-Rechte** (z. B. via `sudo`). Für Wireshark wird zusätzlich GUI-Zugriff oder ein Terminal-Zugriff auf `tshark` benötigt.

---

## 1. Link-local IPv6 addresses

### a) Netzwerk-Topologie
- Hosts: Host1, Host2, Host3
- Router: Router1 (verbindet Netzwerk1 und Netzwerk2)

### b) IPv6 aktivieren (sysctl)
Ziel: IPv6-Unterstützung aktivieren, da diese standardmäßig deaktiviert sein kann.
```bash
# IPv6 temporär aktivieren (bis zum Neustart)
sysctl -w net.ipv6.conf.all.disable_ipv6=0
sysctl -w net.ipv6.conf.default.disable_ipv6=0

# Permanent aktivieren (in /etc/sysctl.conf oder /etc/sysctl.d/*.conf)
echo 'net.ipv6.conf.all.disable_ipv6 = 0' >> /etc/sysctl.d/99-ipv6.conf
echo 'net.ipv6.conf.default.disable_ipv6 = 0' >> /etc/sysctl.d/99-ipv6.conf
sysctl -p
```

### Unicast & Multicast Adressen pro Interface anzeigen
Ziel: alle automatisch und manuell konfigurierten IPv6-Adressen der Schnittstellen anzeigen und überprüfen.
```bash
ip -6 addr show
ip -6 route show
ip -6 neigh show
```

### Weitere Link-local Adresse hinzufügen
Ziel: eine zusätzliche (zweite) Link-local-Adresse für Testzwecke hinzufügen.
```bash
ip -6 addr add fe80::1234/64 dev eth0
```

### Beobachtung mit Wireshark
Ziel: die automatischen NS/NA-Nachrichten der Duplicate Address Detection (DAD) nach Adresshinzufügung analysieren.
- Filter: `icmpv6`

### c) Netzwerk testen
Ziel: Kommunikation via Multicast testen, sowie direkte Link-local-Verbindung mit expliziter Interface-Angabe.
```bash
# Alle lokalen Interfaces via Multicast ermitteln
ping6 ff02::1%eth0

# Ping zwischen Hosts (nur mit Angabe des Interface, da Link-local)
ping6 fe80::1a2b:3c4d:5e6f:7g8h%eth0
```

### Demonstration Exercise 1
Fragen & Antworten:

1. Design und Zweck von Link-local IPv6-Adressen:
Link-local-Adressen (Bereich `fe80::/10`) sind zwingend erforderlich für jede IPv6-Schnittstelle und ermöglichen lokale Kommunikation ohne Router. Sie werden automatisch generiert und si`nd nicht über Router hinaus gültig. Verwendet z. B. für Neighbor Discovery und Routing-Protokolle.

2. Design und Zweck der Solicited-Node Multicast-Adresse:
Diese speziellen Multicast-Adressen (Bereich `ff02::1:ffXX:XXXX`) werden aus der letzten Hälfte der unicast-Adresse gebildet. Sie ermöglichen eine effiziente Erreichbarkeit einzelner Nodes für Neighbor Discovery, ohne den gesamten Link zu belasten.

3. Eigenschaften und Einschränkungen des Link-local-Scopes:

- Nur gültig innerhalb desselben Netzsegmentes (kein Routing)
- Muss für jede Schnittstelle eindeutig sein (mit Zonenkennung z. B. `%eth0`)
- Unerlässlich für grundlegende IPv6-Kommunikation (z. B. Router Discovery)

4. Funktionsweise der Duplicate Address Detection (DAD):
Vor Verwendung einer neuen IPv6-Adresse sendet das Gerät ein Neighbor Solicitation (NS) an die eigene solicited-node multicast-Adresse. Antwortet jemand (NA), ist die Adresse bereits vergeben und darf nicht genutzt werden. Ansonsten wird sie übernommen.

---

## 2. SLAAC (Stateless Address Autoconfiguration)

### a) Setup gemäß RFC 4862
Ziel: SLAAC einrichten – Hosts sollen sich selbstständig konfigurieren, basierend auf Router Advertisements.
- **Prefix**: `2001:67C:2414:1i::/64`
- **Router1 eth0**: `2001:67C:2414:1i::1`

#### `radvd` konfigurieren
Router-Advertisements für SLAAC aktivieren.
```bash
apt install radvd
```
Konfiguration `/etc/radvd.conf`:
```conf
interface eth0 {
  AdvSendAdvert on;
  prefix 2001:67C:2414:1i::/64 {
    AdvOnLink on;
    AdvAutonomous on;
  };
};
```
Start:
```bash
systemctl restart radvd
```

### Clients konfigurieren
Hosts müssen RAs akzeptieren, um ihre Adresse selbst zu konfigurieren.
```bash
sysctl -w net.ipv6.conf.eth0.accept_ra=1
```

### Standardroute anzeigen
Ziel: Überprüfung, ob der Router automatisch als Standardgateway übernommen wurde.
```bash
ip -6 route show default
```

### b) DNS via RA (RFC 8106)
Ziel: DNS-Server-Adressen via Router Advertisement mitteilen.
```conf
 RDNSS 2001:1608:10:25::1c04:b12f 2001:1608:10:25::9249:d69b {
   AdvRDNSSLifetime 60;
 };
```

### c) Verbindung testen
Ziel: Sicherstellen, dass Router und Hosts sich gegenseitig und andere Netzwerke erreichen können.
```bash
ping6 2001:67C:2414:1i::1          # Router von Host aus
ping6 <Adresse von Host1>          # Host1 von Router aus
ping6 2001:67C:2414:10::ffff       # Verbindung zu Router 2
```

### Demonstration Exercise 2
Fragen & Antworten:

1. Wie funktioniert SLAAC?
Router senden sogenannte Router Advertisements (RA) mit Prefix-Informationen. Hosts erzeugen daraus ihre eigene IPv6-Adresse (meist mit EUI-64-Verfahren) und konfigurieren sich selbstständig – ohne DHCP.

2. Was ist der Unterschied zu DHCPv6?
SLAAC ist zustandslos (keine Adressvergabe durch Server). DHCPv6 ist zustandsbehaftet und zentral verwaltet die Adressen.

3. Welche Adresse wird als Standardgateway genutzt?
Die Adresse des Routers, der die RAs sendet – z. B. `2001:67C:2414:1i::1` – wird automatisch als Default Gateway übernommen.

4. Welche Rolle spielt RFC 8106 in dieser Konfiguration?
Er erlaubt die Verteilung von DNS-Servern via RA. Der Router kann DNS-Adressen in RAs integrieren, was SLAAC weiter ergänzt.

---

## 3. Neighbor Discovery Protocol (NDP) analysieren

### Filter in Wireshark
```plaintext
icmpv6
```

### a) Neighbor Solicitation
Ziel: Ermitteln, welche Ziel-MAC-Adresse zu einer IPv6-Adresse gehört (ähnlich wie ARP).
```bash
apt install ndisc6
ndisc6 fe80::abcd%eth0 eth0
```
Beobachtungen:
- Absender: Host (z. B. `fe80::1`)
- Ziel: **solicited-node multicast** (z. B. `ff02::1:ffxx:xxxx`)

### b) Neighbor Advertisement
Ziel: Antwort auf NS – MAC-Adresse wird übermittelt.
- Absender: Zielhost (z. B. Router)
- Ziel: ursprünglicher Anfragesteller

### c) Router Solicitation
Ziel: Aufforderung an Router, eine RA zu senden.
```bash
apt install rdisc6
rdisc6 eth0
```
- Quelle: Host
- Ziel: `ff02::2`

### d) Router Advertisement
Ziel: Verteilen von Netzwerk-Informationen und Prefixes durch Router.
- Quelle: Router
- Ziel: `ff02::1` (alle Nodes)

### e) Diagramm: Nachrichtenfluss NDP
1. **Router Solicitation (RS)**
2. **Router Advertisement (RA)**
3. **Neighbor Solicitation (NS)**
4. **Neighbor Advertisement (NA)**

### Demonstration Exercise 3
Fragen & Antworten:

1. Was macht der Neighbor Discovery Protocol (NDP)?
Er ersetzt ARP in IPv6 und dient zur Erkennung von Nachbarn, zur Adressauflösung und zur Router-Entdeckung.

2. Welche Nachrichten gehören zu NDP?

- RS: Router Solicitation
- RA: Router Advertisement
- NS: Neighbor Solicitation
- NA: Neighbor Advertisement
- Redirect: für alternative Routen

3. Was ist eine solicited-node multicast address?
Eine spezielle Multicast-Adresse, die aus einer Unicast-Adresse berechnet wird – Zieladresse für NS-Pakete.

4. Wie erkennt man DAD im Wireshark?
Ein Host sendet ein NS mit der eigenen Adresse als Ziel. Keine Antwort bedeutet: Adresse ist nutzbar.

---

## 4. DHCPv6 (Stateful Autoconfiguration)

### a) Router als DHCPv6-Client (Prefix Delegation)
Ziel: Router soll Prefix dynamisch via DHCPv6 erhalten und an Hosts weitergeben.
```bash
apt install wide-dhcpv6-client
```
Konfiguration `/etc/wide-dhcpv6/dhcp6c.conf`:
```conf
interface eth1 {
  send ia-na 1;
  send ia-pd 1;
  request domain-name-servers;
  script "/etc/wide-dhcpv6/dhcp6c-script";
};
id-assoc na 1 {};
id-assoc pd 1 {
  prefix-interface eth0 {
    sla-id 0;
    sla-len 8;
  };
};
```
Start:
```bash
systemctl restart wide-dhcpv6-client
```

### b) Hosts konfigurieren
Ziel: Hosts erhalten IPv6-Adresse aus delegiertem Prefix.
```bash
sysctl -w net.ipv6.conf.eth0.accept_ra=1
```

### Konnektivität testen
Ziel: Sicherstellen, dass Kommunikation intern, zu Router2 und ins Internet funktioniert.
```bash
ping6 <Adresse von Router1>
ping6 <Adresse von Router2>
ping6 ipv6.google.com
```

### Demonstration Exercise 4
Fragen & Antworten:

1. Was ist DHCPv6?
Ein Protokoll zur zentralen Zuweisung von IPv6-Adressen, DNS-Servern und Netzwerkpräfixen durch einen Server.

2. Was sind IA_NA und IA_PD?
- IA_NA (Non-temporary Address): reguläre IPv6-Adresse für Hosts
- IA_PD (Prefix Delegation): ein Prefix wird einem Router übergeben, der es an sein LAN weiterleitet

3. Welche Rolle spielt der delegating router?
Er gibt das Prefix an den requesting router weiter. Dieser konfiguriert sich selbst und sein LAN mit diesem Prefix.

4. Welche Konfiguration ist notwendig auf Router1?
Konfiguration von `dhcp6c` mit Interface `eth1` als Anfragestelle und `eth0` als Weiterleitungsstelle des delegierten Prefix.
