# M159 – LB2: Gesamtstruktur (erster DC) & Client

Dieses Repository dokumentiert die Umsetzung von **LB2** im Modul M159: das Aufsetzen der ersten Active-Directory-Gesamtstruktur (DC1) inkl. DNS-Konfiguration, den Beitritt eines Clients zur Domäne sowie ein erstes RDP-Berechtigungskonzept.

## Inhalt

- **AD DS-Rolle** auf dem ersten Domain Controller installieren
- **DC1 promoten** zu einer neuen Gesamtstruktur (Standardspeicherorte)
- **DNS konfigurieren**: Forwarder (9.9.9.9), Forward-Zone, Reverse-Zonen pro Subnetz, PTR-Records, nslookup-Tests
- **AD-Papierkorb** aktivieren
- **Client-Domainbeitritt**: Portanforderungen, Voraussetzungen, Durchführung, PTR-Update
- **Remote Desktop-Berechtigungen**: zwei AD-Gruppen (`RDP-Admins`, `RDP-Users`), manuelle Zuweisung auf dem Client

## Struktur

```
.
├── README.md                                  # diese Datei
├── docs/
│   └── LB2-Gesamtstruktur-DC-Client-Doku.md   # ausführliche Schritt-für-Schritt-Anleitung
└── scripts/
    ├── Setup-DC.ps1                            # AD DS-Rolle installieren & DC1 promoten
    ├── Join-Domain-Client.ps1                  # Client automatisiert der Domäne beitreten lassen
    └── Configure-WindowsSettings.ps1           # Grundeinstellungen (Hostname, Firewall, Tastatur, IPv6 etc.)
```

## Voraussetzungen

- 2 (oder mehr) Windows-Server-Instanzen (z. B. auf AWS EC2), erreichbar per RDP
- Netzwerk/Security Groups mit den benötigten Ports zwischen DC und Client bereits eingerichtet (siehe [Umsetzungsanleitung](docs/LB2-Gesamtstruktur-DC-Client-Doku.md#51-voraussetzungen-pr%C3%BCfen))
- Administratorrechte auf allen beteiligten Servern

## Verwendung

1. Ausführliche Anleitung in [`docs/LB2-Gesamtstruktur-DC-Client-Doku.md`](docs/LB2-Gesamtstruktur-DC-Client-Doku.md) durchgehen.
2. Auf dem zukünftigen DC1: `scripts/Setup-DC.ps1` ausführen (Domainname, NetBIOS-Name und DSRM-Passwort vorher im Skript anpassen).
3. DNS-Forwarder und Reverse-Zonen gemäss Doku manuell im DNS-Manager nachziehen (nicht Teil des Skripts).
4. Auf dem Client: `scripts/Join-Domain-Client.ps1` ausführen, um der Domäne beizutreten.
5. RDP-Gruppen und -Berechtigungen gemäss Kapitel 6 der Doku manuell einrichten.

## Verwandte Ressourcen

- [Bewertungskriterien LB2](../../08-kompetenznachweise/lb2/kompetenzmatrix-lb2.md)
- [Fragenkatalog für den mündlichen Nachweis](fragen.md)
- [Planung: AD & Cloud Setup Sheet](../01-planung/resources/01-a-planung-ad-cloud-setup-sheet.md)

## Autor / Klasse

Rocco Grubisic
PE24c

