# Netwerkontwerp

## Doel

Alle apparaten hangen achter één firewall en zijn verdeeld over vijf VLAN's. Wat tussen de VLAN's mag, wordt bepaald door het firewallbeleid op de FortiGate.

## Topologie

![Topologie](../diagrams/topologie.png)

Modem naar WAN1 van de FortiGate. De DMZ-poort van de FortiGate is een trunk naar poort 1 van de TL-SG108E. Er komt maar één kabel van de modem de kamer in.

## VLAN-plan

| VLAN | Naam | Netwerk | Gateway | Gebruik |
|---|---|---|---|---|
| 10 | Beheer | 10.0.10.0/24 | 10.0.10.1 | Werklaptop, Proxmox-beheer |
| 20 | Servers | 10.0.20.0/24 | 10.0.20.1 | Active Directory, Linux, Zabbix |
| 30 | Clients | 10.0.30.0/24 | 10.0.30.1 | Windows 11 client-VM's |
| 40 | Gasten | 10.0.40.0/24 | 10.0.40.1 | Alleen internet |
| 50 | Thuis | 10.0.50.0/24 | 10.0.50.1 | Desktop-pc, alleen internet |

Het adresbereik 10.0.x.x is bewust gekozen om niet te botsen met het netwerk achter de modem (meestal 192.168.x.x).

## IP-schema (vaste adressen)

| Apparaat | VLAN | Adres |
|---|---|---|
| Proxmox-server | 10 | 10.0.10.10 |
| Werklaptop | 10 | 10.0.10.20 |
| Windows Server (AD, DNS, DHCP) | 20 | 10.0.20.10 |
| Linux-VM | 20 | 10.0.20.20 |
| Zabbix | 20 | 10.0.20.30 |
| Windows 11-client | 30 | DHCP |
| Desktop-pc | 50 | 10.0.50.10 |

De adressen van de Linux-VM en Zabbix zijn voorstellen en worden vastgelegd zodra de VM's bestaan.

## Poortindeling TL-SG108E

| Poort | Verbonden met | VLAN-instelling | PVID |
|---|---|---|---|
| 1 | FortiGate (DMZ), trunk | Tagged 10, 20, 30, 40, 50 | 1 |
| 2 | Proxmox-server | Untagged 10, tagged 20, 30, 40 | 10 |
| 3 | Werklaptop | Untagged 10 | 10 |
| 4 | Desktop-pc | Untagged 50 | 50 |
| 5, 6, 7 | Niet in gebruik | - | - |
| 8 | Reserve (switchbeheer) | Untagged 1 | 1 |

Poort 8 blijft op VLAN 1, zodat de beheerpagina van de switch bereikbaar blijft als er bij het instellen van VLAN's iets misgaat.

## Trunk en access

| Type | Betekenis | Hier |
|---|---|---|
| Access-poort | Hoort bij een VLAN. Het apparaat weet niets van VLAN's. | Poort 3, 4, 8 |
| Trunk-poort | Draagt meerdere VLAN's over één kabel met 802.1Q-tags. | Poort 1 |
| Trunk met native VLAN | Eén VLAN gaat untagged mee, de rest tagged. | Poort 2 (VLAN 10 untagged) |

## Hoe verkeer loopt

Een Windows 11-client (VLAN 30) wil een bestand van de domeincontroller (VLAN 20). Beide draaien op dezelfde Proxmox-host, maar het verkeer gaat toch via de firewall:

1. De client stuurt het verzoek naar zijn gateway, de FortiGate (10.0.30.1).
2. Proxmox voegt VLAN-tag 30 toe en stuurt het via de trunk naar de switch en de FortiGate.
3. De FortiGate controleert het beleid: clients naar servers is toegestaan.
4. De FortiGate stuurt het verkeer terug met tag 20. Proxmox haalt de tag eraf en levert het aan de domeincontroller.

Dit is *router on a stick*: één kabel met meerdere VLAN's en een router die ertussen routeert. Het voordeel is dat de firewall al het verkeer tussen zones ziet, ook als de VM's naast elkaar op één host draaien.

## Beperkingen van de switch

De TL-SG108E biedt VLAN's (tagged en untagged), port mirroring en QoS. Hij biedt geen configureerbare STP, geen CLI of SSH en geen 802.1X, SNMP of ACL's. Voor het oefenen van STP en EtherChannel is een tweede, volledig beheerbare switch nodig.

## FortiGate

Status: gepland, hardware besteld. De menu's en commando's gelden voor FortiOS 7.x.

### Firewallbeleid

De FortiGate weigert standaard alles wat niet expliciet is toegestaan (implicit deny). Het beleid wordt van boven naar beneden gelezen en de eerste passende regel wint, dus de deny-regels staan boven de brede toestaan-regels.

| Nr | Naam | Van | Naar | Actie | NAT |
|---|---|---|---|---|---|
| 1 | Beheer-naar-alles | VLAN 10 | VLAN 20, 30, 40, 50, wan1 | Toestaan | Alleen naar wan1 |
| 2 | Servers-internet | VLAN 20 | wan1 | Toestaan | Aan |
| 3 | Clients-naar-servers | VLAN 30 | VLAN 20 | Toestaan | Uit |
| 4 | Clients-internet | VLAN 30 | wan1 | Toestaan | Aan |
| 5 | Gasten-internet | VLAN 40 | wan1 | Toestaan | Aan |
| 6 | Thuis-internet | VLAN 50 | wan1 | Toestaan | Aan |
| 7 | Deny-thuis-naar-lab | VLAN 50 | VLAN 10, 20, 30, 40 | Weigeren | Uit |
| 8 | Deny-lab-naar-thuis | VLAN 10, 20, 30, 40 | VLAN 50 | Weigeren | Uit |

Regel 7 en 8 komen in de praktijk boven regel 1 en 6. Er wordt gewerkt met adresobjecten (bijvoorbeeld `NET-Servers = 10.0.20.0/24`). Logging staat aan: Log Allowed Traffic bij toestaan-regels en Log Violation Traffic bij deny-regels.
