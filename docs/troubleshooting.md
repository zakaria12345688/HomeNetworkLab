# Problemen en oplossingen

Elk probleem krijgt hier een korte beschrijving: wat er gebeurde, wat de oorzaak was en hoe het is opgelost.

## Proxmox-installatie: bootloader-fout bij het schrijven van de EFI-boot-entry

**Melding**

```
bootloader setup errors:
- failed to prepare EFI boot using Grub on '/dev/nvme0n1p2':
  unable to install the EFI boot loader on '/dev/nvme0n1'
```

De installatie liep door tot 100% ("installation finished") en faalde pas bij het registreren van de bootloader.

**Omgeving**

- Proxmox VE 9.2-1
- Laptop met NVMe-schijf en UEFI-firmware
- Secure Boot stond al uit, Fast Boot stond aan

**Oorzaak**

De installatie schreef de bestanden, maar de firmware kon geen opstartregel (boot-entry) aanmaken. Dit komt vaker voor op laptops en kan komen door Fast Boot, volle of oude boot-entries in het UEFI-geheugen, overgebleven partitieresten of een oude BIOS-versie.

**Aanpak**

1. Fast Boot uitgezet in de BIOS.
2. Overige opties om te proberen: Terminal UI-installatie, oude boot-entries verwijderen, schijf volledig leegmaken met `wipefs -a`, BIOS bijwerken.

**Oplossing**

[Invullen: welke stap het probleem heeft opgelost.]

**Geleerd**

Een bootloader-fout aan het einde van de installatie betekent niet dat de schijfinstallatie mislukt is, maar dat de firmware de boot-entry niet kon aanmaken. De eerste verdachten zijn Fast Boot en de boot-entries in de firmware.

## Sjabloon voor nieuwe problemen

**Melding of symptoom:**

**Omgeving:**

**Oorzaak:**

**Aanpak:**

**Oplossing:**

**Geleerd:**
