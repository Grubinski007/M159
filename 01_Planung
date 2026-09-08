# M159 – Umsetzungsanleitung (Schritt-für-Schritt)

Diese Anleitung beschreibt eine sinnvolle Reihenfolge und die konkreten Arbeitsschritte, um das Projekt-Setup-Sheet vollständig umzusetzen. Sie ergänzt das Setup-Sheet (dort trägst du die konkreten Werte/Passwörter ein), hier steht das **Wie**.



---

## 0. Grobe Reihenfolge im Überblick

1. AWS-Grundlagen: VPC, Subnetze, Sicherheitsgruppen
2. EC2-Instanzen erstellen (DC, Client, Admin Center)
3. On-Premises AD (auf dem DC-Server) installieren
4. AWS Managed Microsoft AD erstellen
5. Trust zwischen On-Prem AD und AWS Managed AD einrichten
6. Admin-Center-Server für die Verwaltung des AWS Managed AD konfigurieren
7. Client-Server der On-Prem-Domäne beitreten
8. Abteilungen/OUs und Benutzer anlegen
9. Öffentliche Domain registrieren (dynv6) und als UPN-Suffix hinzufügen
10. Azure for Students Account + Entra ID Tenant vorbereiten
11. Entra Connect installieren und synchronisieren
12. App-Registrierung (Python-App) in Entra ID
13. Funktionstests & Troubleshooting

---

## 1. AWS-Grundlagen: VPC, Subnetze, Sicherheitsgruppen

### 1.1 VPC und Subnetze
1. AWS-Konsole → **VPC** → falls nicht vorhanden, neue VPC erstellen mit CIDR `10.0.0.0/16` (Name z. B. `M159-VPC`).
2. Vier Subnetze anlegen (Service **Subnets** → *Create subnet*), jeweils der VPC zugeordnet:
   - `M159-subnet-public1-us-east-1a` (z. B. `10.0.0.0/20`, AZ `us-east-1a`)
   - `M159-subnet-public2-us-east-1b` (z. B. `10.0.16.0/20`, AZ `us-east-1b`)
   - `M159-subnet-private1-us-east-1a` (z. B. `10.0.128.0/20`, AZ `us-east-1a`)
   - `M159-subnet-private2-us-east-1b` (z. B. `10.0.144.0/20`, AZ `us-east-1b`)
3. **Internet Gateway** erstellen und an die VPC anhängen.
4. **Routentabelle** für die öffentlichen Subnetze: Route `0.0.0.0/0` → Internet Gateway. Diese Tabelle den beiden `public`-Subnetzen zuweisen.
5. Für die privaten Subnetze reicht vorerst die lokale Routentabelle (kein Internetzugang nötig, da AD-Server nur intern kommunizieren – RDP läuft im Beispiel-Sheet zwar direkt auf öffentlichen IPs, du kannst aber auch NAT/Bastion nutzen, wenn du strikter trennen willst).


### 1.2 Sicherheitsgruppen
1. **Sicherheitsgruppe „DC"** erstellen (für den Domain Controller):
   - Inbound-Regeln exakt gemäss Tabelle 5 im Setup-Sheet anlegen (RDP 3389, LDAP 389, LDAPS 636, Kerberos 88, SMB 445, DNS 53, RPC 135+49152-65535, ICMP, Global Catalog 3268/3269, Kerberos Password 464).
   - Quelle für RDP: `0.0.0.0/0` (nur für Übungszwecke – in Produktion würdest du das einschränken).
   - Quelle für die restlichen Dienste: die VPC-CIDR (`10.0.0.0/16`) oder gezielt die Subnetz-CIDRs, damit nur interne Kommunikation erlaubt ist.
2. **Sicherheitsgruppe „Clients"** erstellen: Regeln gemäss Tabelle 5, Quelle jeweils die drei genannten CIDRs (`10.0.0.0/20`, `10.0.128.0/20`, `10.0.144.0/20`).
3. Outbound-Regeln kannst du offen lassen (Standard: alles erlaubt), ausser dein Lehrmittel verlangt explizit restriktive Outbound-Regeln.

---

## 2. EC2-Instanzen erstellen

Für alle drei Server: **Windows Server 2019/2022 Base** AMI, Instanztyp z. B. `t3.medium` (AD braucht etwas RAM), Key Pair erstellen/verwenden (zum Auslesen des initialen Admin-Passworts).

| Server | Subnetz | Sicherheitsgruppe | Öffentliche IP |
|---|---|---|---|
| DC (On-Prem AD) | `M159-subnet-private1-us-east-1a` | SG „DC" | nein (nur privat, ggf. via Client/Bastion erreichbar) – falls Sheet öffentlich verlangt, dann Elastic IP zuweisen |
| Client | `M159-subnet-public1-us-east-1a` | SG „Clients" | ja (Elastic IP) |
| Admin Center | frei wählbar (empfohlen: public, damit du dich direkt per RDP verbindest) | SG „Clients" oder eigene | ja (Elastic IP) |

Schritte je Instanz:
1. EC2 → *Launch Instance* → Name, AMI, Instanztyp wählen.
2. Netzwerk: richtige VPC/Subnetz auswählen, „Auto-assign public IP" nur für öffentliche Instanzen aktivieren.
3. Sicherheitsgruppe zuweisen.
4. Speicher: Standard (30 GB) reicht meist.
5. Instanz starten, danach unter *Connect* → *RDP client* das Passwort mit dem Key Pair entschlüsseln.
6. Für DC und Client: **Elastic IP** reservieren und zuweisen, falls öffentlich erreichbar (damit die IP fix bleibt).
7. Hostnamen/FQDN setzen (Systemsteuerung → System → Computername ändern), Server neu starten.



---

## 3. On-Premises Active Directory auf dem DC-Server installieren

1. Per RDP auf den DC-Server verbinden.
2. **Server-Manager** → *Add Roles and Features* → Rolle **Active Directory Domain Services** (AD DS) sowie **DNS Server** installieren.
3. Nach der Installation: *Promote this server to a domain controller* → **Add a new forest**.
4. Root-Domain-Name eingeben, z. B. `ec2.tbz.m159` (Third-Level-Domäne gemäss Tabelle 6).
5. Forest-/Domain-Functional-Level wählen (Standard: höchste verfügbare Stufe), DNS-Server-Option aktiviert lassen.
6. **DSRM-Passwort** (Directory Services Restore Mode) vergeben – das ist das „Kennwort-Demote" im Sheet.
7. NetBIOS-Namen bestätigen, Pfade für Datenbank/Logs/SYSVOL übernehmen, Installation abschliessen.
8. Server startet neu und ist danach Domain Controller. Mit `Administrator@ec2.tbz.m159` und dem Domänenadmin-Passwort anmelden.
9. Statische private IP auf dem DC konfigurieren und als DNS-Server für sich selbst eintragen (127.0.0.1 oder eigene IP), damit spätere Clients/Trusts korrekt auflösen.

---

## 4. AWS Managed Microsoft AD erstellen

1. AWS-Konsole → **Directory Service** → *Set up directory* → **AWS Managed Microsoft AD**.
2. Edition wählen (Standard reicht für Schulzwecke).
3. Domain-Name eingeben, z. B. `aws.tbz.m159` (Third-Level-Domäne-2 gemäss Tabelle 6).
4. Admin-Passwort für den Benutzer `admin` vergeben (→ „AWS Managed Admin Passwort").
5. VPC auswählen sowie die **beiden privaten Subnetze** (`M159-subnet-private1-us-east-1a` und `-private2-us-east-1b`) – AWS Managed AD benötigt zwingend zwei Subnetze in unterschiedlichen AZs.
6. Erstellung abschliessen (dauert ca. 20–45 Minuten).
7. Nach Fertigstellung: In der Directory-Übersicht die **DNS-Server-IPs** notieren (zwei Adressen) → Tabelle 6.


---

## 5. Trust zwischen On-Prem AD und AWS Managed AD einrichten

Ein **Tree-Root Trust** (bidirektional empfohlen) zwischen `ec2.tbz.m159` (On-Prem, EC2) und `aws.tbz.m159` (AWS Managed AD) einrichten.

### 5.1 Auf dem On-Prem DC vorbereiten
1. DNS-Weiterleitung (Conditional Forwarder) für die Zone `aws.tbz.m159` einrichten, die auf die beiden DNS-Server des AWS Managed AD zeigt.
2. In **Active Directory Domains and Trusts** → Rechtsklick auf die eigene Domäne → *Properties* → *Trusts* → *New Trust*.
3. Domänenname der Zieldomäne (`aws.tbz.m159`) eingeben, Trust-Typ **Forest Trust** bzw. wie im Sheet vermerkt **Tree-Root Trust**, Richtung **Two-way**.
4. Trust-Passwort festlegen (muss auf beiden Seiten identisch sein) → in Tabelle 6 „Trust Passwort" eintragen.

### 5.2 Auf der AWS-Seite (Directory Service Konsole)
1. Directory Service → deine AWS Managed AD auswählen → Tab **Trust relationships** → *Add trust relationship*.
2. Remote-Domänenname (`ec2.tbz.m159`), dieselbe Trust-Passwort eingeben, Trust-Richtung **Two-way**, Trust-Typ **Forest**.
3. Conditional Forwarder für die On-Prem-Domäne wird von AWS automatisch angelegt – prüfen, ob die On-Prem-DNS-Server auch umgekehrt erreichbar sind (Sicherheitsgruppen: Port 53 TCP/UDP zwischen den Subnetzen freigeben, siehe Kapitel 1.2).
4. Trust-Status abwarten, bis er auf beiden Seiten **„Verified"**/aktiv ist.

### 5.3 Test
- `nltest /trusted_domains` bzw. `nltest /sc_verify:aws.tbz.m159` auf dem DC ausführen.
- Von einem Client aus versuchen, eine Ressource mit einem Benutzer der jeweils anderen Domäne zu öffnen.

---

## 6. Admin-Center-Server für AWS Managed AD

1. EC2-Instanz „Windows Server Admin Center" starten (siehe Kapitel 2), Windows-AMI mit RSAT-fähiger Version.
2. Server der On-Prem-Domäne beitreten **oder** direkt die RSAT-Tools installieren und gegen die AWS Managed AD Domäne arbeiten:
   - *Server Manager* → *Add Roles and Features* → **Remote Server Administration Tools (RSAT)** → *AD DS and AD LDS Tools* installieren.
3. **Active Directory Users and Computers** öffnen → *Change Domain* → `aws.tbz.m159` mit dem `admin`-Benutzer verbinden.
4. Von hier aus kannst du OUs, Benutzer und Gruppen in der AWS Managed AD verwalten (analog zu Kapitel 8, aber für die AWS-Seite, falls dein Sheet das verlangt).
5. Optional: **AWS Managed Microsoft AD Admin Center Delegation** – zusätzliche Delegated-Admin-Gruppen nutzen statt des `admin`-Benutzers, falls gefordert.

---

## 7. Client der On-Prem-Domäne beitreten

1. Per RDP auf den Client-Server verbinden.
2. DNS-Server des Clients auf die IP des On-Prem-DC setzen (Netzwerkadapter-Einstellungen).
3. *System* → *Change settings* → *Change...* → Domäne `ec2.tbz.m159` eingeben, mit Domänenadmin-Zugangsdaten bestätigen.
4. Neustart, danach mit einem Domänenbenutzer anmelden können.

---

## 8. Abteilungen (OUs) und Benutzer anlegen

1. Auf dem DC: **Active Directory Users and Computers** öffnen.
2. Für jede Abteilung eine **Organisationseinheit (OU)** erstellen: `Sekretariat`, `Buchhaltung`, `GL`, `Promoter`.
3. Pro OU je einen Benutzer anlegen (Rechtsklick OU → *New* → *User*):
   - Vorname/Nachname/Benutzername vergeben, Kennwort setzen (Option „Password never expires" für Übungszwecke sinnvoll).
   - „Bereiche" laut Sheet: `Sekretariat`, `Buchhaltung`, `GL` = intern, `Promoter` = extern → das kannst du z. B. über eine zusätzliche Security-Gruppe `Intern`/`Extern` abbilden, der du die Benutzer zuweist, oder über eine OU-Struktur `Intern`/`Extern` mit den Abteilungs-OUs darunter.
4. Gruppenstruktur optional erweitern (z. B. globale Sicherheitsgruppen je Abteilung), falls für spätere Berechtigungen (Freigaben, GPOs) benötigt.


---

## 9. Öffentliche Domain registrieren und als UPN-Suffix hinzufügen

1. Auf **https://dynv6.com/** registrieren und einen Domainnamen anlegen, z. B. `m159tbz.v6.rocks`.
2. Auf dem DC: **Active Directory Domains and Trusts** → Rechtsklick auf den Root-Knoten → *Properties* → *UPN Suffixes* → den neuen Namen (`m159tbz.v6.rocks`) hinzufügen.
3. Bei Benutzern (einzeln oder per PowerShell-Skript) das UPN-Suffix von `@ec2.tbz.m159` auf `@m159tbz.v6.rocks` umstellen:
   ```powershell
   Get-ADUser -Filter * -SearchBase "OU=Sekretariat,DC=ec2,DC=tbz,DC=m159" | 
     ForEach-Object { Set-ADUser $_ -UserPrincipalName ($_.SamAccountName + "@m159tbz.v6.rocks") }
   ```
4. Diesen öffentlichen Domainnamen später auch bei der Entra-ID-Domainverifizierung (Kapitel 11) verwenden, damit UPNs beim Sync konsistent bleiben.

---

## 10. Azure for Students Account & Entra ID vorbereiten

1. Falls noch nicht vorhanden: Azure for Students Account gemäss verlinkter Anleitung im Sheet erstellen (ggf. mit privater/Gmail-Adresse, nicht TBZ-Mail).
2. Im Azure-Portal den **Microsoft Entra ID**-Tenant öffnen, den initialen `.onmicrosoft.com`-Domainnamen notieren.
3. Falls gewünscht: die eigene öffentliche Domain (aus Kapitel 9) unter **Entra ID → Custom domain names** hinzufügen und per TXT-Record bei dynv6 verifizieren.
4. Einen **Global Administrator**-Benutzer notieren/anlegen → Tabelle 6 „Azure AD (Entra ID)".

---

## 11. Entra Connect installieren und synchronisieren

1. Entra Connect (Azure AD Connect / neuer: "Microsoft Entra Connect Sync") auf einem geeigneten Server installieren – meist auf dem DC selbst oder einem dedizierten Member-Server (im Sheet: „Entra Connect Server (DC aus AD)").
2. Installer von Microsoft herunterladen, **Custom installation** wählen (mehr Kontrolle als Express).
3. Sync-Methode wählen: i. d. R. **Password Hash Synchronization** (einfachste Methode für Schulzwecke).
4. Mit dem Entra Global-Admin-Konto verbinden, danach die lokale AD-Domäne (`ec2.tbz.m159`) als *Enterprise Admin* hinzufügen.
5. OU-Filterung: nur die relevanten OUs (Sekretariat, Buchhaltung, GL, Promoter) für die Synchronisation auswählen.
6. UPN-Attribut prüfen: sicherstellen, dass die synchronisierten Benutzer das öffentliche UPN-Suffix (`@m159tbz.v6.rocks`) verwenden, das mit der verifizierten Entra-Domain übereinstimmt.
7. Installation abschliessen, ersten Sync-Zyklus abwarten (Standard alle 30 Minuten, manuell erzwingbar mit `Start-ADSyncSyncCycle -PolicyType Delta`).
8. Im Entra-Portal unter **Users** prüfen, ob die Benutzer erscheinen (Quelle: „Windows Server AD").

---

## 12. Python-App-Registrierung in Entra ID

1. Entra-Portal → **App registrations** → *New registration*.
2. Namen vergeben, Supported account types passend wählen (i. d. R. „Accounts in this organizational directory only").
3. Nach Erstellung: **Application (client) ID** und **Directory (tenant) ID** aus der Übersichtsseite kopieren.
4. **Certificates & secrets** → *New client secret* → Beschreibung/Ablaufdatum wählen → den **Wert** des Secrets sofort kopieren (wird nur einmal angezeigt) → das ist die „Client Secret ID" im Sheet.
5. Unter **API permissions** die für deine Python-App benötigten Berechtigungen hinzufügen (z. B. Microsoft Graph `User.Read` o. ä.) und ggf. **Admin consent** erteilen.

---

## 13. Funktionstests & Troubleshooting

- **DNS-Auflösung**: Von jedem Server aus `nslookup` gegen die jeweils andere Domäne testen.
- **Trust**: `nltest /sc_verify:<remote-domain>` auf dem DC.
- **RDP**: Alle drei Server über ihre öffentliche IP/Elastic IP erreichbar?
- **Login**: Ein Benutzer aus jeder Abteilung kann sich am Client anmelden.
- **Entra Sync**: Benutzer erscheinen im Entra-Portal, UPN stimmt mit öffentlicher Domain überein.
- **App Registration**: Mit den drei Werten (Tenant-ID, Client-ID, Secret) einen einfachen Python-Testscript gegen Microsoft Graph laufen lassen (`requests` + OAuth2 Client Credentials Flow), um die Werte zu verifizieren.
- Häufige Fehlerquellen: fehlende Sicherheitsgruppen-Regeln (Port 53/88/389/445 zwischen den Subnetzen), falsche DNS-Server-Einträge auf den EC2-Instanzen, Trust-Passwort stimmt nicht auf beiden Seiten überein, UPN-Suffix nicht mit der verifizierten Entra-Domain identisch.

---
