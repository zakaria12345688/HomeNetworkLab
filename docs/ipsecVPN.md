# FortiGate IPsec Dialup VPN: documentatie

Remote-access VPN (`remote-VPN`) voor toegang tot het lab (vlan10, vlan20, vlan30).

## Belangrijk

- **NAT Traversal staat aan (Enable).** Er staat een modem voor de FortiGate met port forwarding van **UDP 500** (IKE) en **UDP 4500** (NAT-T) naar `wan1`. De FortiGate zit dus achter NAT. Zonder NAT-T werkt de tunnel niet.
- **IKE versie 1 met Main mode (ID protection).** Main mode verbergt de identiteit tijdens de onderhandeling en is veiliger dan Aggressive.
- **Pre-shared key van 30+ tekens.** Lang genoeg om offline kraken praktisch onmogelijk te maken.
- **Diffie-Hellman groep 20** (elliptic curve P-384, ca. 192 bit beveiliging). Dit is de gekozen groep, want ze is veiliger dan groep 19 (P-256, ca. 128 bit). Groep 20 wordt ook gebruikt voor PFS in Phase 2.
- **Split tunnel voor vlan10, vlan20 en vlan30.** Alleen het labverkeer gaat door de tunnel, het internet blijft lokaal op de client (zie onderaan).

---

## 1. Network

![Network](images/01-network.png)

---

## 2. Authentication

![Authentication](images/02-authentication.png)

---

## 3. Phase 1 Proposal

![Phase 1 Proposal](images/03-phase1-proposal.png)

---

## 4. Phase 2 Selectors

![Phase 2 Selectors](images/04-phase2-selectors.png)

---

## Split tunnel (vlan10, vlan20, vlan30)

Bij *Accessible Networks* staat nu `all`, waardoor de client al het verkeer door de tunnel stuurt (full tunnel). Dit wordt aangepast naar split tunnel, zodat enkel de lab-vlans via de VPN bereikt worden.

**Aanpassing:**

1. Maak een address group aan (*Policy & Objects → Addresses → Create New → Address Group*), bijvoorbeeld `VPN-lab-networks`, met daarin:
   - `vlan10 address`
   - `vlan20 address`
   - `vlan30 address`
2. Ga naar *VPN → IPsec Tunnels → remote-VPN → Edit* en zet bij **Accessible Networks** de group `VPN-lab-networks` in plaats van `all`.
3. Laat **Enable IPv4 Split Tunnel** aangevinkt.
4. Verbreek de VPN op de client en verbind opnieuw, zodat de nieuwe routes opgehaald worden.

**Gevolgen:**

- Alleen verkeer naar vlan10, vlan20 en vlan30 gaat door de tunnel. Internet loopt via de lokale verbinding van de client.
- De policy `internetVPN` (`remote-VPN → wan1`) is dan niet meer nodig voor normaal gebruik.
- De logs worden overzichtelijker: geen ruis meer van achtergrondverkeer (zoals Tailscale) naar internetadressen, enkel verkeer naar het lab.
- De toegang per server/poort blijft geregeld door de firewall policies (bijvoorbeeld `VPN-Naar-Lab`). Split tunnel bepaalt alleen welke routes de client krijgt.
- Controleer dat de vlan-subnetten niet overlappen met het thuisnetwerk waar de client op zit.
- Screenshot 1 (Network) moet na de wijziging opnieuw genomen worden.
