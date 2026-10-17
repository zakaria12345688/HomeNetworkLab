# Netwerkontwerp

## Doel

Alle apparaten hangen achter één firewall en zijn verdeeld over vijf VLAN's. Wat tussen de VLAN's mag, wordt bepaald door het firewallbeleid op de FortiGate. De werklaptop zit niet fysiek in het lab, maar verbindt via een VPN.

## Topologie

![Topologie](../diagrams/topologie.svg)

Modem naar WAN1 van de FortiGate. De DMZ-poort van de FortiGate is een trunk naar poort 1 van de TL-SG108E. Er komt maar één kabel van de modem de kamer in.

## VLAN-plan

| VLAN | Naam    | Netwerk      | Gateway   | Gebruik                                |
| ---- | ------- | ------------ | --------- | -------------------------------------- |
| 10   | Beheer  | 10.0.10.0/24 | 10.0.10.1 | Proxmox-beheer, FortiGate-beheer       |
| 20   | Servers | 10.0.20.0/24 | 10.0.20.1 | Active Directory, Linux, Zabbix        |
| 30   | Clients | 10.0.30.0/24 | 10.0.30.1 | Windows 11 client-VM's                 |
| 40   | Gasten  | 10.0.40.0/24 | 10.0.40.1 | Alleen internet                        |
| 50   | Thuis   | 10.0.50.0/24 | 10.0.50.1 | Desktop-pc, alleen internet            |

Het adresbereik 10.0.x.x is bewust gekozen om niet te botsen met het netwerk achter de modem (meestal 192.168.x.x).

De VPN heeft geen eigen VLAN. Zie [Toegang op afstand](#toegang-op-afstand-vpn) voor de VPN-adressen.

## IP-schema (vaste adressen)

| Apparaat                          | VLAN | Adres                                      |
| --------------------------------- | ---- | ------------------------------------------ |
| Proxmox-server                    | 10   | 10.0.10.10                                 |
| Windows Server 2025 (AD, DNS)     | 20   | 10.0.20.10                                 |
| Linux-VM                          | 20   | 10.0.20.20                                 |
| Zabbix                            | 20   | 10.0.20.30                                 |
| Windows 11-client                 | 30   | DHCP (FortiGate)                           |
| Desktop-pc                        | 50   | 10.0.50.10 (vast, buiten de DHCP-range)    |
| Werklaptop, fase 1 (Tailscale)    | -    | Tailscale-IP 100.x.y.z                     |
| Werklaptop, fase 2 (IPsec)        | -    | Adres uit de pool 10.0.99.0/24             |
| Reserve-laptop (switchbeheer)     | 1    | 192.168.0.x (vast, zie [Noodtoegang](#noodtoegang)) |

De adressen van de Linux-VM en Zabbix zijn voorstellen en worden vastgelegd zodra de VM's bestaan. De IPsec-pool 10.0.99.0/24 is een voorstel en wordt vastgelegd bij fase 2.

## DHCP en DNS

De FortiGate is de DHCP-server. Er draait geen DHCP-rol op de Windows Server.

| VLAN | DHCP                | Range (voorstel)          | DNS die de client krijgt        |
| ---- | ------------------- | ------------------------- | ------------------------------- |
| 10   | Nee, vaste adressen | -                         | -                               |
| 20   | Nee, vaste adressen | -                         | -                               |
| 30   | FortiGate           | 10.0.30.100 - 10.0.30.200 | 10.0.20.10 (Active Directory)   |
| 40   | FortiGate           | 10.0.40.100 - 10.0.40.200 | Via de FortiGate                |
| 50   | FortiGate           | 10.0.50.100 - 10.0.50.200 | Via de FortiGate                |

Voor VLAN 30 geeft de FortiGate de Windows Server (10.0.20.10) als DNS-server mee. Zonder dat kan een Windows 11-client het domein niet vinden en dus niet toetreden. De Windows Server stuurt DNS-vragen naar buiten door voor alles wat niet bij het domein hoort.

Zolang de Windows Server nog niet bestaat, krijgt VLAN 30 tijdelijk een publieke DNS-server mee (bijvoorbeeld `1.1.1.1`). Zodra Active Directory draait, wordt dit `10.0.20.10`. Er wordt bewust geen DHCP-relay gebruikt: de FortiGate blijft de enige DHCP-server.

## Poortindeling TL-SG108E

| Poort      | Verbonden met          | VLAN-instelling                | PVID |
| ---------- | ---------------------- | ------------------------------ | ---- |
| 1          | FortiGate (DMZ), trunk | Tagged 10, 20, 30, 40, 50      | 1    |
| 2          | Proxmox-server         | Untagged 10, tagged 20, 30, 40 | 10   |
| 3, 5, 6, 7 | Niet in gebruik        | -                              | -    |
| 4          | Desktop-pc             | Untagged 50                    | 50   |
| 8          | Reserve (switchbeheer) | Untagged 1                     | 1    |

De werklaptop heeft geen eigen poort. Die verbindt via de VPN.

## Trunk en access

| Type                  | Betekenis                                               | Hier                       |
| --------------------- | ------------------------------------------------------- | -------------------------- |
| Access-poort          | Hoort bij een VLAN. Het apparaat weet niets van VLAN's. | Poort 4, 8                 |
| Trunk-poort           | Draagt meerdere VLAN's over één kabel met 802.1Q-tags.  | Poort 1                    |
| Trunk met native VLAN | Eén VLAN gaat untagged mee, de rest tagged.             | Poort 2 (VLAN 10 untagged) |

## Hoe verkeer loopt

Een Windows 11-client (VLAN 30) wil een bestand van de domeincontroller (VLAN 20). Beide draaien op dezelfde Proxmox-host, maar het verkeer gaat toch via de firewall:

1. De client stuurt het verzoek naar zijn gateway, de FortiGate (10.0.30.1).
2. Proxmox voegt VLAN-tag 30 toe en stuurt het via de trunk naar de switch en de FortiGate.
3. De FortiGate controleert het beleid: clients naar servers is toegestaan (regel 6).
4. De FortiGate stuurt het verkeer terug met tag 20. Proxmox haalt de tag eraf en levert het aan de domeincontroller.

Dit is *router on a stick*: één kabel met meerdere VLAN's en een router die ertussen routeert. Het voordeel is dat de firewall al het verkeer tussen zones ziet, ook als de VM's naast elkaar op één host draaien.

## Toegang op afstand (VPN)

De werklaptop verbindt in twee fases. De twee oplossingen reiken niet even ver.

| | Fase 1: Tailscale | Fase 2: IPsec-VPN (Fortinet) |
|---|---|---|
| Status | Klaar | Gepland |
| Tunnel eindigt op | De Proxmox-host | De FortiGate |
| Adres werklaptop | Tailscale-IP 100.x.y.z | Adres uit de pool 10.0.99.0/24 |
| Bereikbaar | **Alleen de Proxmox-host** (webinterface op poort 8006, SSH) | **De host én de VM's**: beheer, servers en clients, volgens het firewallbeleid |
| Firewallbeleid | Niet van toepassing, het verkeer loopt buiten de FortiGate om | Regel 10 |
| Authenticatie | Tailscale-account | Gekoppeld aan Active Directory |

**Tailscale is bewust beperkt tot de host.** Tailscale draait niet op de FortiGate en op de VM's staat het niet. Er worden geen subnet routes geadverteerd, dus de VM's, de FortiGate en de rest van het netwerk zijn via Tailscale niet bereikbaar. Een VM beheer je in fase 1 via de console in de Proxmox-webinterface. De webinterface van de FortiGate bereik je in fase 1 via een SSH-tunnel door de Proxmox-host (`ssh -L 8443:10.0.10.1:443 <gebruiker>@<tailscale-ip-host>`), waarna je `https://localhost:8443` opent. Tailscale zelf routeert dan nog steeds niets.

**De IPsec-VPN is voor het hele lab.** De tunnel eindigt op de FortiGate, dus de laptop kan daarachter de host én de VM's bereiken. Wat precies mag, bepaalt het firewallbeleid.

Tailscale maakt alleen een uitgaande verbinding vanaf de Proxmox-host naar buiten. Daarom moet VLAN 10 naar internet mogen (regel 4).

## Noodtoegang

Omdat de werklaptop geen fysieke poort meer heeft, zijn er twee noodroutes:

- **Console op de FortiGate:** USB-naar-RJ45-consolekabel (FTDI) en PuTTY. Dit werkt altijd, ook zonder netwerk.
- **Fysiek op een interne poort van de FortiGate:** zolang de interne interface beheertoegang houdt (`192.168.1.99`), kom je er met een laptop in een interne poort altijd in. Dit is een bewuste keuze en kan later worden uitgezet.
- **Switchbeheer via poort 8:** de beheerpagina van de TL-SG108E zit op VLAN 1 en is alleen bereikbaar via poort 8. De FortiGate routeert VLAN 1 niet, dus de pagina is niet via het netwerk of de VPN te bereiken. Dat is een voordeel: de switch kan niet vanaf buiten worden beheerd.

Om de switch via poort 8 te bereiken, krijgt de reserve-laptop een **vast IP in het netwerk van de switch**, bijvoorbeeld `192.168.0.10` met subnetmasker `255.255.255.0`. De standaardadres van de TL-SG108E is `192.168.0.1`. Controleer dit op jouw switch, want het kan zijn aangepast. Poort 8 blijft op VLAN 1, zodat de beheerpagina bereikbaar blijft als er bij het instellen van VLAN's iets misgaat.

## Beperkingen van de switch

De TL-SG108E biedt VLAN's (tagged en untagged), port mirroring en QoS. Hij biedt geen configureerbare STP, geen CLI of SSH en geen 802.1X, SNMP of ACL's. Voor het oefenen van STP en EtherChannel is een tweede, volledig beheerbare switch nodig.

## FortiGate

Status: bezig. De VLAN-interfaces, DHCP en het firewallbeleid staan op de FortiGate, de switch volgt. De menu's en commando's gelden voor FortiOS 6.4, de versie die op het apparaat staat. Het apparaat heeft geen actieve licenties, dus er zijn geen FortiGuard-updates en geen support.

### Beheertoegang tot de FortiGate

Alleen VLAN 10 (Beheer) mag de FortiGate beheren. Alle andere VLAN's krijgen geen toegang tot de webinterface of SSH.

| Interface              | Toegestaan                         |
| ---------------------- | ---------------------------------- |
| wan1, wan2             | Alleen PING (geen HTTPS, SSH of FMG-Access) |
| dmz (trunk)            | Alleen PING (geen IP-adres, geen beheer) |
| vlan10 (Beheer)        | PING, HTTPS, SSH                   |
| vlan20, 30, 40, 50     | Alleen PING                        |

- Het beheer van de FortiGate loopt niet via het firewallbeleid, maar via de beheertoegang per interface. Daarom is een deny-regel voor VLAN 50 niet genoeg: HTTPS op vlan50 moet echt uit.
- De admin-account krijgt als *Trusted Host* `10.0.10.0/24`, en bij fase 2 ook de IPsec-pool `10.0.99.0/24`. Dit wordt pas ingesteld nadat het beheer vanaf VLAN 10 getest is, anders sluit je jezelf buiten.

### Firewallbeleid

De FortiGate weigert standaard alles wat niet expliciet is toegestaan (implicit deny). Het beleid wordt van boven naar beneden gelezen en de eerste passende regel wint. Daarom staan de deny-regels bovenaan en de brede toestaan-regels daaronder.

| Nr | Naam                 | Van                 | Naar                     | Actie    | NAT              |
| -- | -------------------- | ------------------- | ------------------------ | -------- | ---------------- |
| 1  | Deny-thuis-naar-lab  | VLAN 50             | VLAN 10, 20, 30, 40      | Weigeren | Uit              |
| 2  | Deny-lab-naar-thuis  | VLAN 10, 20, 30, 40 | VLAN 50                  | Weigeren | Uit              |
| 3  | Beheer-naar-lab      | VLAN 10             | VLAN 20, 30, 40          | Toestaan | Uit              |
| 4  | Beheer-internet      | VLAN 10             | wan1                     | Toestaan | Aan              |
| 5  | Servers-internet     | VLAN 20             | wan1                     | Toestaan | Aan              |
| 6  | Clients-naar-servers | VLAN 30             | VLAN 20                  | Toestaan | Uit              |
| 7  | Clients-internet     | VLAN 30             | wan1                     | Toestaan | Aan              |
| 8  | Gasten-internet      | VLAN 40             | wan1                     | Toestaan | Aan              |
| 9  | Thuis-internet       | VLAN 50             | wan1                     | Toestaan | Aan              |
| 10 | VPN-naar-lab (fase 2) | IPsec-tunnel (10.0.99.0/24) | VLAN 10, 20, 30 | Toestaan | Uit              |

Opmerkingen bij het beleid:

- Regel 1 en 2 houden Thuis (VLAN 50) volledig los van het lab, in beide richtingen. Ze staan bovenaan, zodat geen enkele toestaan-regel eronder dit kan omzeilen. Daardoor kan ook Beheer niet bij VLAN 50.
- Antwoordverkeer op toegestane verbindingen wordt automatisch doorgelaten (stateful), daarvoor is geen aparte regel nodig.
- Naast deze regels staat bovenaan de regel `Intern-noodtoegang` (interne poorten naar wan1, NAT aan). Die is bedoeld voor het bouwen van het lab en wordt later uitgezet of verwijderd. De beheertoegang op de interne interface (`192.168.1.99`) blijft als fysieke noodtoegang bestaan zolang ik dat wil.
- Regel 10 is alleen actief in fase 2. De VPN heeft geen toegang tot Gasten (VLAN 40) en Thuis (VLAN 50).
- Tailscale (fase 1) valt buiten dit beleid. Het verkeer gaat tussen de laptop en de Proxmox-host via een uitgaande verbinding, toegestaan door regel 4.
- DHCP-verkeer van de clients naar de FortiGate zelf hoeft niet in het beleid, omdat de FortiGate dat zelf afhandelt.
- Er wordt gewerkt met adresobjecten (bijvoorbeeld `NET-Servers = 10.0.20.0/24`). Logging staat aan: Log Allowed Traffic bij toestaan-regels en Log Violation Traffic bij deny-regels.
