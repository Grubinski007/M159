# LB2 – Gesamtstruktur (erster DC) & Client: Umsetzungsanleitung

Diese Doku führt durch die Erstellung der ersten AD-Gesamtstruktur (DC1), die DNS-Konfiguration, den Client-Domainbeitritt und das RDP-Berechtigungskonzept.

---

## 1. AD DS-Rolle auf DC1 hinzufügen

1. Per RDP auf den Server verbinden, der DC1 werden soll.
2. **Server Manager** → *Manage* → *Add Roles and Features*.
3. Installationstyp **Role-based or feature-based installation**, eigenen Server auswählen.
4. Rolle **Active Directory Domain Services** anhaken → *Add Features* bestätigen (inkl. abhängiger Tools) → *Install*.
5. Nach Abschluss erscheint im Server Manager ein gelbes Ausrufezeichen-Symbol → **Promote this server to a domain controller** anklicken (startet den nächsten Schritt).

> Alternativ per PowerShell: `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools`

---

## 2. DC1 promoten (neue Gesamtstruktur)

1. Im Assistenten **"Add a new forest"** auswählen.
2. **Root domain name** eingeben (deine geplante Domäne, z. B. `m159.tbz`).
3. **Domain Controller Options**:
   - Forest-/Domain-Functional-Level: höchste verfügbare Stufe wählen (ausser dein Setup verlangt Abwärtskompatibilität).
   - **DNS Server** bleibt angehakt (wird zusammen installiert).
   - DSRM-Passwort (Safe Mode) vergeben und notieren (Setup-Sheet: „Kennwort-Demote").
4. **DNS Options**: Warnung wegen fehlender Delegation ist normal bei einer neuen Root-Domäne → ignorieren/weiter.
5. **Additional Options**: NetBIOS-Name bestätigen/anpassen.
6. **Paths**: **Standardspeicherorte belassen** (Datenbank, Logs, SYSVOL unter `C:\Windows\...`) – wie im Auftrag verlangt, hier nichts ändern.
7. **Review Options** prüfen, *Install* klicken.
8. Server startet automatisch neu und ist danach DC1 in der neuen Gesamtstruktur.

---

## 3. DNS konfigurieren

### 3.1 DNS-Weiterleitung (Forwarder) auf 9.9.9.9

1. **DNS Manager** öffnen → Rechtsklick auf den Servernamen → *Properties* → Tab **Forwarders** → *Edit*.
2. IP `9.9.9.9` eintragen, mit *Validate* testen, dann *OK*.

**Warum 9.9.9.9 (Quad9) statt 8.8.8.8 (Google Public DNS)?**
- **Quad9 (9.9.9.9)** ist ein gemeinnütziger DNS-Dienst, betrieben von der Quad9 Foundation zusammen mit Partnern wie IBM und PCH. Der Fokus liegt auf **Sicherheit**: Anfragen zu bekannten Malware-, Phishing- und Botnet-Domains werden anhand von Threat-Intelligence-Feeds automatisch blockiert.
- Quad9 wirbt zudem mit einer **datenschutzfreundlicheren Haltung**: Es werden laut eigener Aussage keine IP-Adressen dauerhaft mit Anfragen verknüpft/gespeichert, und es werden keine personenbezogenen Daten für Werbezwecke genutzt.
- **8.8.8.8 (Google)** ist technisch sehr zuverlässig und schnell, aber Google ist ein Werbeunternehmen; DNS-Anfragen können (im Rahmen von Googles Datenschutzrichtlinien) mit anderen Google-Diensten in Verbindung gebracht werden. Für eine Schul-/Firmenumgebung, in der man bewusst Wert auf Datensparsamkeit und eingebauten Malware-Schutz legt, ist Quad9 daher oft die bevorzugte Wahl.
- *Hinweis für deine Doku:* Diese Einschätzung ist eine gängige Begründung in der Fachliteratur – prüfe für deinen Nachweis zusätzlich die aktuellen Datenschutzerklärungen beider Anbieter, da sich Angaben ändern können.

### 3.2 Forward-Zone

- Wird bei der Promotion automatisch erstellt (Zone = deine AD-Domäne, z. B. `m159.tbz`). Keine manuelle Aktion nötig, nur zur Kontrolle im DNS-Manager unter **Forward Lookup Zones** prüfen.

### 3.3 Reverse-Zone (pro Subnetz)

1. **DNS Manager** → **Reverse Lookup Zones** → Rechtsklick → *New Zone*.
2. Zonentyp **Primary zone**, in AD gespeichert (*Store the zone in Active Directory*).
3. **Network ID** des jeweiligen Subnetzes eingeben (z. B. `10.0.128.0/20`) – der Assistent berechnet daraus die passende `in-addr.arpa`-Zone.
4. **Wiederholen für jedes Subnetz**, in dem AD-Mitglieder liegen (DC, Client, ggf. weitere) – bei AWS besonders wichtig, da sonst die Reverse-Auflösung einen falschen/generischen FQDN liefert (AWS vergibt sonst automatisch eigene interne PTR-Einträge).
5. Dynamische Updates aktivieren: *Zone Properties* → **Dynamic updates: Secure only** (da AD-integriert).

### 3.4 PTR-Record für den DC aktualisieren

1. Im **DNS Manager** unter der Forward-Zone prüfen, ob der A-Eintrag für DC1 vorhanden ist.
2. Rechtsklick auf diesen A-Eintrag → *Properties* → Häkchen bei **"Update associated pointer (PTR) record"** setzen → OK.
   - Alternativ: In der Reverse-Zone manuell einen PTR-Eintrag für die IP des DC anlegen, der auf den FQDN zeigt.

### 3.5 NSLOOKUP testen

Auf dem DC in einer Konsole:
```
nslookup dc1.m159.tbz
nslookup 10.0.128.10
```
- Erster Befehl (Forward): sollte die IP des DC liefern.
- Zweiter Befehl (Reverse): sollte den FQDN des DC liefern.
- Falls die Reverse-Auflösung fehlschlägt oder einen AWS-internen Namen statt deines FQDN liefert → Reverse-Zone/PTR-Eintrag nochmals prüfen (siehe 3.3/3.4).

---

## 4. Diverse AD-Einstellungen

### AD-Papierkorb (Recycle Bin) aktivieren

1. **Active Directory Administrative Center** öffnen.
2. Links auf den Domänennamen klicken → im Aufgabenbereich rechts **"Enable Recycle Bin..."** anklicken.
3. Bestätigen (Achtung: Dies ist **nicht rückgängig machbar** und erfordert, dass sich die Änderung über alle DCs repliziert – bei nur einem DC sofort aktiv).

Alternativ per PowerShell:
```powershell
Enable-ADOptionalFeature -Identity 'Recycle Bin Feature' `
    -Scope ForestOrConfigurationSet -Target 'm159.tbz' -Confirm:$false
```

---

## 5. Client in die Domäne einbinden

### 5.1 Voraussetzungen prüfen

- Client-Edition muss **Pro/Enterprise/Ultimate/Education** sein (Home-Editionen können keiner Domäne beitreten – bei Windows Server ist das ohnehin gegeben).
- DNS-Server des Clients muss auf den DC zeigen (nicht auf 9.9.9.9 oder AWS-DNS direkt).
- Benötigte Ports zwischen Client und DC müssen offen sein (siehe Tabelle unten) – bei AWS über die Security Group des Client-Subnetzes sicherstellen.

**Benötigte Ports (Client → DC):**

| Port | Protokoll | Zweck |
|---|---|---|
| 53 | UDP | DNS |
| 88 | TCP | Kerberos-Authentifizierung |
| 135 | TCP | RPC Endpoint Mapper |
| 139 | TCP | NetBIOS Session Service |
| 389 | TCP | LDAP |
| 445 | TCP | SMB/CIFS |
| 49152–65535 | TCP | Dynamische RPC-Ports |

Test vor dem Domain-Join:
```powershell
Test-NetConnection -ComputerName dc1.m159.tbz -Port 389
Test-NetConnection -ComputerName dc1.m159.tbz -Port 445
Test-NetConnection -ComputerName dc1.m159.tbz -Port 88
```
Wenn hier `TcpTestSucceeded : False` erscheint, zuerst die Security Group korrigieren, **bevor** du den Domain-Join versuchst (sonst genau das Symptom aus dem Auftrag: Join dauert extrem lange, endet scheinbar erfolgreich und wirft danach einen Fehler).

### 5.2 Domain-Join durchführen

1. Auf dem Client: **Systemsteuerung** → **System** → **Einstellungen für Computername, Domäne und Arbeitsgruppe ändern** → **Ändern**.
2. Computernamen prüfen/setzen (muss eindeutig sein, darf im AD noch nicht existieren).
3. Bei **Domäne** die FQDN eintragen (z. B. `m159.tbz`) → OK.
4. Anmeldefenster: Domänenbenutzer mit Beitrittsrecht angeben (standardmässig Domänen-Administratoren).
5. Erfolgsmeldung bestätigen → **Neustart**.
6. Nach dem Neustart: Anmeldung als Domänenbenutzer möglich. Der Computer liegt standardmässig in der OU **Computers**.

**Alternativ per PowerShell** (analog zum bereits vorhandenen `Join-Domain-Client.ps1`):
```powershell
Add-Computer -DomainName "m159.tbz" -Credential (Get-Credential) -Restart
```

### 5.3 PTR-Record für den Client aktualisieren

- Gleich wie beim DC (Kapitel 3.4): im DNS-Manager beim A-Eintrag des Clients **"Update associated pointer (PTR) record"** aktivieren, oder manuell PTR-Eintrag in der passenden Reverse-Zone erstellen.

### 5.4 Anmeldeformate merken

| Format | Bedeutung |
|---|---|
| `.\Logonname` | Lokale Anmeldung am Client |
| `Computername\Logonname` | Lokale Anmeldung am Client |
| `Domain\Logonname` | Domänenanmeldung über NetBIOS-Namen |
| `Logonname@Domain` | Domänenanmeldung über DNS-Namen (UPN) |

---

## 6. Remote Desktop als Admin & User (manuelles Konzept)

Ziel: zwei AD-Gruppen erstellen, die den RDP-Zugriff steuern – zunächst manuell auf einem Client umgesetzt (später via GPO).

### 6.1 Zwei AD-Gruppen anlegen

Im **Active Directory Users and Computers** (auf DC1):
1. Neue Sicherheitsgruppe erstellen, z. B. `RDP-Admins` (Bereich: Global, Typ: Security).
2. Zweite Sicherheitsgruppe erstellen, z. B. `RDP-Users`.
3. Die gewünschten Benutzer den jeweiligen Gruppen als Mitglieder hinzufügen.



### 6.2 Manuelle Zuweisung auf dem Client

1. Auf dem **Client** (nicht auf dem DC): `sysdm.cpl` ausführen (Win+R → `sysdm.cpl` → Enter) – entspricht "Schritt 1" im Auftrag.
2. Tab **Remote** → Bereich **Remotedesktop** → **"Select Users..."** klicken – entspricht "Schritt 2".
3. Im Dialog **Add...** klicken, die Gruppe `RDP-Users` (bzw. einzelne Benutzer) eintragen, mit **Check Names** validieren, OK.
4. Damit dürfen nun auch normale Benutzer aus dieser Gruppe sich per RDP mit dem Client verbinden – ohne diesen Schritt würden nur lokale Administratoren durchkommen.

### 6.3 Test

- Mit einem Benutzer aus `RDP-Users` versuchen, sich per RDP am Client anzumelden → sollte funktionieren.
- Mit einem Benutzer, der in **keiner** der beiden Gruppen ist, testen → RDP-Verbindung sollte mit einer Berechtigungsfehlermeldung abgelehnt werden.


