# Dokumentation – RSAT Tools & Windows Admin Center V2 (WAC)

Verwaltung eines AWS Managed Microsoft Active Directory

## Inhaltsverzeichnis

- [Dokumentation – RSAT Tools \& Windows Admin Center V2 (WAC)](#dokumentation--rsat-tools--windows-admin-center-v2-wac)
  - [Inhaltsverzeichnis](#inhaltsverzeichnis)
  - [1. Einleitung und Zielsetzung](#1-einleitung-und-zielsetzung)
  - [2. Architektur und Netzwerk-Planung](#2-architektur-und-netzwerk-planung)
    - [2.1 Platzierung der Admin-Center-Instanz (Public vs. Private Subnet)](#21-platzierung-der-admin-center-instanz-public-vs-private-subnet)
  - [3. RSAT Tools installieren](#3-rsat-tools-installieren)
    - [3.2 Verwaltung von Objekten im AWS Managed AD](#32-verwaltung-von-objekten-im-aws-managed-ad)
  - [4. Admin Center Server in die AWS Managed Domäne aufnehmen](#4-admin-center-server-in-die-aws-managed-domäne-aufnehmen)
    - [4.1 Admin Center Setup-Datei lokalisieren](#41-admin-center-setup-datei-lokalisieren)
    - [4.2 NAT-Gateway](#42-nat-gateway)
    - [4.3 Admin Center installieren](#43-admin-center-installieren)
  - [5. Domain Controller aus dem EC2-AD zum Admin Center hinzufügen](#5-domain-controller-aus-dem-ec2-ad-zum-admin-center-hinzufügen)
    - [5.1 WinRM aktivieren und Firewall konfigurieren](#51-winrm-aktivieren-und-firewall-konfigurieren)
    - [5.2 Verbindung testen und DC hinzufügen](#52-verbindung-testen-und-dc-hinzufügen)
  - [6. Externer Zugriff auf das Admin Center (HTTPS/RDP)](#6-externer-zugriff-auf-das-admin-center-httpsrdp)
  - [7. Fazit](#7-fazit)

---

## 1. Einleitung und Zielsetzung

Diese Dokumentation beschreibt die Einrichtung eines Windows Admin Center V2 (WAC) sowie der Remote Server Administration Tools (RSAT) zur Verwaltung eines AWS Managed Microsoft Active Directory. Ziel ist es, sowohl über RSAT als auch über das webbasierte Admin Center administrative Aufgaben in der Domäne durchführen zu können.

Folgende Teilziele wurden umgesetzt:

- Neue EC2-Client-Instanz innerhalb des AWS Managed AD erstellt
- RSAT-Tools zur Verwaltung des AWS Managed AD installiert
- Windows Admin Center V2 installiert und erfolgreich angemeldet
- Domain Controller des EC2-AD zum Admin Center hinzugefügt
- Externer Zugriff auf das Admin Center (HTTPS/RDP) unter Berücksichtigung von Sicherheitsaspekten realisiert

> **Hinweis:** Die Domain Controller des AWS Managed AD selbst können dem Admin Center nicht hinzugefügt werden, da AWS dies nicht zulässt. Verwaltet wird daher stattdessen der Domain Controller aus dem eigenen EC2-AD.

## 2. Architektur und Netzwerk-Planung

### 2.1 Platzierung der Admin-Center-Instanz (Public vs. Private Subnet)

| Kriterium | Public Subnet | Private Subnet |
|---|---|---|
| Erreichbarkeit von aussen | Direkt über Internet Gateway möglich | Nur über Bastion Host, VPN oder Reverse Proxy |
| Angriffsfläche | Höher – Instanz besitzt öffentliche IP | Geringer – keine direkte Erreichbarkeit von aussen |
| Internetzugriff für Setup/Updates | Direkt über Internet Gateway | Nur via NAT-Gateway |
| Best Practice für Domänen-Infrastruktur | Eher unüblich | Empfohlen |

**Gewählte Variante: Private Subnet**

Die Admin-Center-Instanz wurde bewusst in einem **privaten Subnetz** platziert, da sie administrativen Zugriff auf die Domäne hat und deshalb keine öffentliche IP-Adresse besitzen soll. Eine direkt aus dem Internet erreichbare Management-Instanz wäre ein bevorzugtes Angriffsziel (Credential-Stuffing, Brute-Force, Exploits gegen den WAC-Dienst). Durch die Platzierung im privaten Subnetz ist die Instanz nur innerhalb der VPC bzw. über kontrollierte Zugriffswege erreichbar. Der benötigte Internetzugriff für die Installation von Windows Admin Center wird über ein NAT-Gateway realisiert (siehe Abschnitt 4.2).

**Zukünftiger Zugriff auf das Admin Center:** Der Zugriff erfolgt künftig ausschliesslich über ein AWS Client VPN in die VPC. Nach Aufbau der VPN-Verbindung kann das Admin Center über HTTPS mit der privaten IP-Adresse der Instanz erreicht werden. Ein direkter RDP- oder HTTPS-Zugriff aus dem öffentlichen Internet ist nicht vorgesehen (siehe Abschnitt 6).

## 3. RSAT Tools installieren

Die Remote Server Administration Tools (RSAT) ermöglichen die Verwaltung der Domäne direkt von einer Client-Instanz aus, ohne dass ein Domain Controller selbst bearbeitet werden muss.

Installation mittels PowerShell:

```powershell
Install-WindowsFeature -Name RSAT -IncludeAllSubFeature -IncludeManagementTools
```

Nach der Installation stehen folgende Konsolen zur Verfügung:

| Konsole | Funktion |
|---|---|
| `dsa.msc` | Active Directory-Benutzer und -Computer |
| `gpmc.msc` | Gruppenrichtlinien-Verwaltung |
| `dnsmgmt.msc` | DNS-Verwaltung |

### 3.2 Verwaltung von Objekten im AWS Managed AD

Über `dsa.msc` wird eine Verbindung zur Domäne des AWS Managed AD hergestellt (Rechtsklick auf **Active Directory-Benutzer und -Computer** → **Domäne ändern…** und Angabe des Domänennamens). Anschliessend können Objekte wie folgt verwaltet werden:

1. **Neues Objekt anlegen:** Rechtsklick auf die gewünschte Organisationseinheit (OU) → **Neu** → z. B. **Benutzer** oder **Gruppe**, anschliessend Attribute (Name, Anmeldename, Passwort) im Assistenten ausfüllen.
2. **Objekt bearbeiten:** Rechtsklick auf ein bestehendes Objekt → **Eigenschaften**, z. B. um Gruppenmitgliedschaften, Kontooptionen oder Kontaktangaben anzupassen.
3. **Berechtigungen prüfen:** Da es sich um ein AWS Managed AD handelt, sind einige Container (z. B. `Domain Controllers`) schreibgeschützt bzw. werden von AWS verwaltet. Eigene Objekte werden in der delegierten OU-Struktur unterhalb der von AWS bereitgestellten OU angelegt, in welcher der administrative Benutzer volle Rechte besitzt.

Auf die gleiche Weise werden DNS-Einträge über `dnsmgmt.msc` und Gruppenrichtlinien über `gpmc.msc` verwaltet – auch hier erfolgt die Verbindung zur verwalteten Domäne über die jeweilige "Verbinden zu"-Funktion der Konsole.

## 4. Admin Center Server in die AWS Managed Domäne aufnehmen

Damit die Admin-Center-Instanz Mitglied der Domäne werden kann, wurde zunächst der primäre und sekundäre DNS-Server der Netzwerkkarte auf die IP-Adressen der beiden AWS-Managed-AD-Domain-Controller angepasst (diese stellt AWS beim Erstellen des Managed AD zur Verfügung, z. B. `10.0.1.10` und `10.0.2.10`). Anschliessend wurde die Instanz über **System → Erweiterte Systemeinstellungen → Computername ändern** der Domäne beigetreten und neu gestartet.

![Join](resources/join.png)

> **Wichtig:** Ab dem Domänen-Beitritt darf die Anmeldung an diesem System nicht mehr mit lokalen Anmeldedaten erfolgen – ausgenommen ist die Anmeldung am Admin Center selbst, welche weiterhin über einen lokalen Benutzer erfolgt.

### 4.1 Admin Center Setup-Datei lokalisieren

Bei Windows Server 2025 (AWS-Image) ist die Setup-Datei bereits vorhanden und wurde mit folgendem Befehl gesucht:

```powershell
Get-ChildItem -Recurse -Path C:\ -Filter *AdminCenter*.exe -ErrorAction SilentlyContinue
```

Der Befehl lieferte die Installationsdatei im Verzeichnis `C:\Windows\Setup\Scripts\WindowsAdminCenter.exe` (Pfad kann je nach AMI-Version leicht abweichen).

### 4.2 NAT-Gateway

Da die Instanz in einem privaten Subnetz erstellt wurde, das Setup jedoch Internetzugriff benötigt, wurde ein NAT-Gateway eingerichtet:

| Schritte | Printscreens |
|---|---|
| Schritt 1 (Der NAT-Gateway muss in einem "public" Subnetz erstellt werden) | ![nat-gateway-0](resources/nat-gateway-0.png) |
| Schritt 2 (Bearbeiten der Routingtabelle des privaten Subnetzes: `0.0.0.0/0` → NAT-Gateway) | ![nat-gateway-1](resources/nat-gateway1.png) |
| Schritt 3 (Verbindung erfolgreich getestet, Internetzugriff aus dem privaten Subnetz vorhanden) | ![nat-gateway-2](resources/nat-gateway2.png) |

### 4.3 Admin Center installieren

Die Installation erfolgte grösstenteils mit den Standardwerten des Setup-Assistenten, da diese für den vorliegenden Anwendungsfall (interne Verwaltungsumgebung ohne Sonderanforderungen) passend sind.

| Schritte | Printscreen |
|---|---|
| Schritt 1 | ![AdminCenterInstall (1)](resources/admincenterinstall-1.png) |
| Schritt 2 | ![AdminCenterInstall (2)](resources/admincenterinstall-2.png) |
| Schritt 3 | ![AdminCenterInstall (3)](resources/admincenterinstall-3.png) |
| Schritt 4 | ![AdminCenterInstall (4)](resources/admincenterinstall-4.png) |
| Schritt 5 (Standardport 443 für den HTTPS-Zugriff auf das Admin Center beibehalten) | ![AdminCenterInstall (5)](resources/admincenterinstall-5.png) |
| Schritt 6 (Anmeldung mit dem lokalen Administrator) | ![Login](resources/login.png) |
| Schritt 7 | Weiter mit dem Hinzufügen des Domain Controllers, siehe Abschnitt 5 |

## 5. Domain Controller aus dem EC2-AD zum Admin Center hinzufügen

Damit der Domain Controller über das Admin Center verwaltet werden kann, muss WinRM (Windows Remote Management) erreichbar sein. Dafür wurden folgende Ports in der Security Group des Domain Controllers freigegeben:

| Protokoll | Port | Richtung | Quelle |
|---|---|---|---|
| TCP | 5985 (HTTP) | Ingress | Private IP der Admin-Center-Instanz (`/32`) |
| TCP | 5986 (HTTPS) | Ingress | Private IP der Admin-Center-Instanz (`/32`) |

> **Sicherheitshinweis:** Es wurde bewusst **nicht** `0.0.0.0/0` verwendet, sondern der Zugriff exakt auf die private IP-Adresse der Admin-Center-Instanz eingeschränkt, um die Angriffsfläche auf ein Minimum zu reduzieren.

### 5.1 WinRM aktivieren und Firewall konfigurieren

Auf dem Domain Controller wurde WinRM aktiviert:

![winrm-aktivieren](resources/winrm-aktivieren.png)

```powershell
Enable-PSRemoting -Force
```

Anschliessend wurden die entsprechenden Windows-Firewall-Regeln freigegeben:

```powershell
# Regel für HTTP (Port 5985)
Set-NetFirewallRule -Name "WINRM-HTTP-In-TCP-PUBLIC" -RemoteAddress Any -Enabled True -Profile Domain,Private,Public -Action Allow

# Optional: HTTPS (Port 5986) – nur wenn konfiguriert
Set-NetFirewallRule -Name "WINRM-HTTPS-In-TCP" -RemoteAddress Any -Enabled True -Profile Domain,Private,Public -Action Allow
```

Die eigentliche Einschränkung auf die zulässige Quell-IP erfolgt über die AWS Security Group (siehe Tabelle oben); die Windows-Firewall-Regel selbst lässt die Herkunfts-IP offen, da die Filterung bereits auf Netzwerkebene (VPC) greift.

### 5.2 Verbindung testen und DC hinzufügen

Die WinRM-Verbindung vom Admin-Center-Server zum Domain Controller wurde erfolgreich getestet:

![winrm](resources/winrm-check.png)

Anschliessend wurde der Domain Controller im Admin Center über **Alle Verbindungen → Hinzufügen → Servermanager** mit dem vollqualifizierten Domänennamen (FQDN) sowie Domänen-Administrator-Anmeldedaten hinzugefügt:

![add-dc-admin-center](resources/add-dc-admin-center.png)

## 6. Externer Zugriff auf das Admin Center (HTTPS/RDP)

Da sich die Admin-Center-Instanz in einem privaten Subnetz befindet, ist sie standardmässig nicht aus dem Internet erreichbar. Für den Zugriff von aussen wurden folgende Sicherheitsüberlegungen berücksichtigt:

- HTTPS-Zugriff nur über ein gültiges Zertifikat (kein self-signed Zertifikat im Produktivbetrieb)
- Kein direkter Zugriff auf das Admin Center aus dem öffentlichen Internet
- Security Groups so restriktiv wie möglich konfiguriert (keine `0.0.0.0/0`-Regeln)
- RDP-Zugriff nur für administrative Ausnahmefälle, nie direkt aus dem Internet freigegeben
- Multi-Faktor-Authentifizierung (MFA) für den VPN-Zugang aktiviert
- Regelmässiges Logging und Monitoring der Zugriffe über VPC Flow Logs und CloudTrail

**Gewählte Lösung:** Der externe Zugriff erfolgt über ein **AWS Client VPN**, das mit der privaten VPC verbunden ist. Nach erfolgreichem Verbindungsaufbau (inkl. Zertifikats-basierter Authentifizierung und MFA) kann sowohl das Admin Center per HTTPS (`https://<private-ip>`) als auch bei Bedarf ein RDP-Zugriff auf die Instanz aufgebaut werden. Diese Variante wurde gewählt, da sie im Vergleich zu einem öffentlich exponierten Reverse Proxy oder einer Portfreigabe die geringste Angriffsfläche bietet und der gesamte Datenverkehr verschlüsselt über den VPN-Tunnel läuft, ohne dass eine öffentliche IP-Adresse oder Portfreigabe auf der Instanz selbst nötig ist.

## 7. Fazit

Die Einrichtung von RSAT und Windows Admin Center V2 zur Verwaltung des AWS Managed AD hat gezeigt, dass beide Werkzeuge sich sinnvoll ergänzen: RSAT eignet sich für schnelle, skriptbasierte Verwaltung direkt via MMC-Konsolen, während das Admin Center eine übersichtliche, webbasierte Oberfläche mit Server-Monitoring bietet. Die grösste Herausforderung war die Netzwerk-Planung, insbesondere die Entscheidung für ein privates Subnetz samt NAT-Gateway sowie die restriktive Konfiguration der WinRM-Ports über Security Groups. Durch die Kombination aus privatem Subnetz, NAT-Gateway für ausgehenden Internetzugriff und einem Client-VPN für den administrativen Zugriff von aussen wurde eine Lösung erreicht, die sowohl funktional als auch sicherheitstechnisch den Best Practices für die Verwaltung einer Active-Directory-Infrastruktur in AWS entspricht.
