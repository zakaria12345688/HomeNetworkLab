# Proxmox

## Rol

De server-laptop draait Proxmox VE en host alle virtuele machines. Beheer gaat via de webinterface (poort 8006) en SSH, vanaf de werklaptop via de VPN. In fase 1 (Tailscale) bereik je alleen de host zelf. De VM's beheer je dan via de console in de webinterface. In fase 2 (IPsec) bereik je ook de VM's, bijvoorbeeld via RDP.

Status: Proxmox VE 9.2-1 is geïnstalleerd en draait. Het VLAN-gedeelte volgt zodra de FortiGate in het netwerk staat.

## Installatie

- ISO: Proxmox VE 9.2-1, gecontroleerd met de SHA256-som van de downloadpagina.
- USB-stick gemaakt met Rufus in DD Image-modus.
- Virtualisatie (VT-x of AMD-V) aangezet in de BIOS.
- Bestandssysteem **ext4**, geen ZFS, omdat ZFS veel RAM gebruikt voor caching.
- Tijdens de installatie een vast IP-adres in het netwerk van de modem. Dit wordt omgezet naar 10.0.10.10 zodra de FortiGate er staat.
- Een bootloader-fout tijdens de installatie is beschreven in [troubleshooting](troubleshooting.md).

## Na de installatie

- Enterprise-repository uit en no-subscription-repository aan, via *Node, Updates, Repositories*.
- Systeem bijwerken.
- Eigen gebruiker aanmaken in plaats van alles met root te doen.
- In `/etc/systemd/logind.conf` de waarde `HandleLidSwitch=ignore` zetten, zodat de laptop met gesloten deksel blijft draaien.

## Tailscale op de host

Tailscale is de VPN van fase 1. Het draait alleen op de Proxmox-host, niet op de VM's.

- Installeren met het installatiescript van Tailscale en daarna `tailscale up` uitvoeren.
- Het Tailscale-IP van de host opvragen met `tailscale ip -4`.
- Er worden **geen subnet routes** geadverteerd. Daardoor is via Tailscale alleen de host bereikbaar (webinterface en SSH) en niet de VM's of de rest van het netwerk. Dat is een bewuste keuze. Voor toegang tot de VM's is de IPsec-VPN van fase 2 bedoeld, zie [netwerk.md](netwerk.md#toegang-op-afstand-vpn).
- Tailscale maakt een uitgaande verbinding. Zodra de host in VLAN 10 zit, moet het firewallbeleid VLAN 10 naar internet toestaan (regel 4).

## VLAN-aware bridge

De bridge `vmbr0` is de virtuele switch in de server. Door hem VLAN-aware te maken, verwerkt Proxmox VLAN-tags. Elke VM krijgt een virtuele netwerkkaart met een VLAN-tag (*VM, Hardware, Network Device, VLAN Tag*). De VM weet zelf niets van VLAN's: Proxmox voegt de tag toe zodra het pakket de VM verlaat en haalt hem eraf bij binnenkomst.

## Virtuele machines

| VM                                | VLAN | Adres            | RAM   | Cores | Schijf       |
| --------------------------------- | ---- | ---------------- | ----- | ----- | ------------ |
| Proxmox zelf                      | 10   | 10.0.10.10       | 2 GB  | -     | 10 GB        |
| Windows Server 2025 (AD, DNS)     | 20   | 10.0.20.10       | 4 GB  | 2     | 40 tot 60 GB |
| Linux-VM                          | 20   | 10.0.20.20       | 1 GB  | 1     | 10 tot 20 GB |
| Zabbix                            | 20   | 10.0.20.30       | 2 GB  | 2     | 20 tot 30 GB |
| Windows 11-client                 | 30   | DHCP (FortiGate) | 4 GB  | 2     | 50 tot 64 GB |
| Totaal                            |      |                  | 13 GB |       |              |

Bij 16 GB RAM blijft ongeveer 3 GB vrij voor een lichte LXC-container. Het totaal aan toegewezen RAM blijft onder het fysieke geheugen, anders gaat de host swappen. Windows 11 vereist UEFI (OVMF) en een virtuele TPM 2.0. Schijven worden thin geprovisioned.

De Windows Server doet alleen Active Directory en DNS. De FortiGate deelt de IP-adressen uit (DHCP), zie [netwerk.md](netwerk.md#dhcp-en-dns).

## Opstarten en afsluiten

- Bij elke VM *Start at boot* aan.
- Opstartvolgorde: eerst de Windows Server (domeincontroller), daarna met vertraging de rest, omdat clients en Zabbix DNS en AD nodig hebben.
- Afsluiten gaat via *Shutdown* in de webinterface, zodat VM's netjes afsluiten. Dat is vooral belangrijk voor de Active Directory-database.
