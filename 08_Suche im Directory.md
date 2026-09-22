# Dokumentation – Auftrag 08: Suche im Directory

Domäne: `firma.local` — aufbauend auf der OU-Struktur aus Auftrag 07 (Intern/Extern, Abteilungs-OUs
mit `Benutzer`/`Computer`) und der Gruppen-Nesting-Struktur aus Auftrag 04.

> **Annahme zu den Gruppen (Auftrag 04):** Für jede Abteilung existiert eine gleichnamige
> Sicherheitsgruppe (`IT`, `Verkauf`, `Personal`, `Finanzen`, `Geschäftsleitung`, `Aussendienst`,
> `Partner`, `Promoter`), abgelegt unter `OU=Gruppen,DC=firma,DC=local`. Die internen
> Abteilungsgruppen sind Mitglied der Gruppe `intern`, die externen Abteilungsgruppen (inkl.
> `Promoter`) sind Mitglied der Gruppe `extern` (Gruppen-Nesting, keine direkte Benutzermitgliedschaft).
> Die Abteilung «Buchhaltung» aus diesem Auftrag entspricht der OU/Gruppe `Finanzen` aus Auftrag 07.

## Inhaltsverzeichnis

1. [Teil 1 – Bind und Base-DN](#teil-1--bind-und-base-dn)
2. [Teil 2 – Drei Suchen zum Aufwärmen](#teil-2--drei-suchen-zum-aufwärmen)
3. [Teil 3 – Der Filter für die Firewall](#teil-3--der-filter-für-die-firewall)
4. [Teil 4 – Den Filter wirksam machen](#teil-4--den-filter-wirksam-machen)
5. [Teil 5 – Konfigurationsblatt](#teil-5--konfigurationsblatt)
6. [ki-log.md (Auszug)](#ki-logmd-auszug)

---

## Teil 1 – Bind und Base-DN

**Base-DN:** `DC=firma,DC=local`

Der Base-DN leitet sich direkt aus dem DNS-Domänennamen `firma.local` ab: Jedes durch einen Punkt
getrennte Label des Domänennamens wird zu einer eigenen `DC=`-Komponente (`firma.local` →
`DC=firma,DC=local`). LDP schlägt diesen Wert beim Verbindungsaufbau automatisch vor, weil er ihn
aus dem RootDSE-Eintrag des Domain Controllers ausliest.

**Bind:** Der Bind authentifiziert die Verbindung gegenüber dem Verzeichnis — erst danach weiss
der Domain Controller, *welches* Sicherheitstoken (welche Gruppenmitgliedschaften und Rechte) für
die nachfolgenden Suchen gilt. Mit «Bind as currently logged on user» wird der aktuell angemeldete
Domänen-Benutzer verwendet. Ohne Bind (anonyme Verbindung) würde der DC die Suche entweder
komplett ablehnen oder nur stark eingeschränkte, öffentlich sichtbare Attribute zurückgeben, da
anonyme LDAP-Binds in Active Directory standardmässig restriktiv konfiguriert sind.

**DN eines Promoters** (Beispielbenutzer):

```
CN=Petra Notter,OU=Benutzer,OU=Promoter,OU=Extern,DC=firma,DC=local
```

## Teil 2 – Drei Suchen zum Aufwärmen

### 2.1 Ein bestimmter Promoter über seinen Anmeldenamen

| Werkzeug | Filter | Base-DN / Scope | Treffer |
|---|---|---|---|
| `ldp.exe` (Browse → Search) | `(sAMAccountName=petra.notter)` | `DC=firma,DC=local`, Subtree | 1 |
| PowerShell | `Get-ADUser -LDAPFilter "(sAMAccountName=petra.notter)" -SearchBase "DC=firma,DC=local"` | — | 1 |

Beide Wege liefern dasselbe Objekt, da `Get-ADUser -LDAPFilter` intern exakt dieselbe LDAP-Suche
an den DC schickt wie `ldp.exe`.

### 2.2 Alle Benutzer der Abteilung Finanzen (Buchhaltung)

```powershell
Get-ADUser -LDAPFilter "(&(objectCategory=person)(objectClass=user))" `
    -SearchBase "OU=Finanzen,OU=Intern,DC=firma,DC=local" -SearchScope Subtree
```

| Scope | Base-DN | Treffer | Begründung |
|---|---|---|---|
| **One Level** | `OU=Finanzen,OU=Intern,DC=firma,DC=local` | **0** | Direkt unterhalb von `OU=Finanzen` liegen nur die beiden Container `OU=Benutzer` und `OU=Computer` — keine Benutzerobjekte. `One Level` betrachtet ausschliesslich die direkten Kinder der Basis, die Benutzer liegen aber eine Ebene tiefer. |
| **Subtree** | `OU=Finanzen,OU=Intern,DC=firma,DC=local` | **> 0** | `Subtree` durchsucht alle Ebenen unterhalb der Basis, findet also die Benutzerobjekte innerhalb `OU=Benutzer`. Der Filter `objectCategory=person` schliesst dabei die parallel liegenden Computerobjekte in `OU=Computer` aus. |

Das bestätigt genau die in Auftrag 07 geforderte Trennung der letzten DIT-Ebene: Weil Benutzer und
Computer in eigenen Unter-OUs liegen, liefert `One Level` auf der Abteilungs-OU nie Kontoobjekte.

### 2.3 Alle Gruppen der Domain

```powershell
Get-ADGroup -Filter * -SearchBase "DC=firma,DC=local" | Select-Object Name, DistinguishedName
```

DN der Gruppe `extern` (wird in Teil 3 benötigt):

```
CN=extern,OU=Gruppen,DC=firma,DC=local
```

## Teil 3 – Der Filter für die Firewall

### 3.1 Alle direkten Mitglieder der Gruppe `Promoter`

```
(&(objectCategory=person)(objectClass=user)(memberOf=CN=Promoter,OU=Gruppen,DC=firma,DC=local))
```

Treffer: entspricht der Anzahl Benutzer, die direkt (nicht verschachtelt) Mitglied von `Promoter`
sind — in unserer Testumgebung **3**.

### 3.2 Alle Benutzer, die zu `extern` gehören

**Naiver Filter (liefert 0 Treffer):**

```
(&(objectCategory=person)(objectClass=user)(memberOf=CN=extern,OU=Gruppen,DC=firma,DC=local))
```

→ **0 Treffer.** Mitglied von `extern` ist gemäss dem Nesting aus Auftrag 04 nur die *Gruppe*
`Promoter` (sowie `Aussendienst` und `Partner`) — kein einzelner Benutzer ist direktes Mitglied
von `extern`. Ein einfacher `memberOf`-Vergleich prüft aber nur die direkte Mitgliedschaft.

**Korrigierter Filter** mit der LDAP-Matching-Rule `LDAP_MATCHING_RULE_IN_CHAIN`
(OID `1.2.840.113556.1.4.1941`), die Gruppenmitgliedschaften transitiv über beliebig viele
Verschachtelungsebenen auflöst:

```
(&(objectCategory=person)(objectClass=user)(memberOf:1.2.840.113556.1.4.1941:=CN=extern,OU=Gruppen,DC=firma,DC=local))
```

→ **12 Treffer** (alle Benutzer aus `Promoter`, `Aussendienst` und `Partner` zusammen, in unserer
Testumgebung).

### 3.3 Dasselbe, aber nur aktive Konten — der Filter für die Firewall

Der Kontostatus steckt als Bit-Flag im Attribut `userAccountControl` (Bit `0x2` =
`ACCOUNTDISABLE`). LDAP prüft einzelne Bits mit der bitweisen Matching-Rule
`LDAP_MATCHING_RULE_BIT_AND` (OID `1.2.840.113556.1.4.803`):

```
(&(objectCategory=person)(objectClass=user)(memberOf:1.2.840.113556.1.4.1941:=CN=extern,OU=Gruppen,DC=firma,DC=local)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
```

→ **11 Treffer** (12 externe Benutzer minus 1 deaktiviertes Testkonto).

Dies ist der **finale Filter für die Firewall**.

### Nachweis, dass der Filter stimmt

| Test | Durchführung | Ergebnis |
|---|---|---|
| 1 – Konto deaktivieren | Testweise ein Promoter-Konto über `Disable-ADAccount` deaktiviert | Trefferzahl sinkt von 11 auf 10, das deaktivierte Konto verschwindet aus dem Resultat |
| 2 – Neue externe Abteilung | Gruppe `Externe Berater` angelegt, in `extern` aufgenommen, Testbenutzer `Res Berater` hineingelegt | Testbenutzer erscheint **ohne Filteränderung** in den Resultaten, da die Matching-Rule transitiv über jede beliebige Verschachtelung unter `extern` auflöst. Testgruppe und -benutzer anschliessend wieder entfernt. |

**Begründung, warum der Filter auf `extern` statt auf `Promoter` zielt:** Der Filter soll den
tatsächlichen Geschäftsgrund abbilden — «wer darf von aussen ins Firmennetz» — und nicht nur die
heute existierende Abteilung `Promoter`. Da jede künftige externe Abteilung automatisch Mitglied
von `extern` werden muss (gleiches Nesting-Prinzip wie in Auftrag 04), erscheinen neue externe
Mitarbeitende ohne jede Anpassung an Firewall oder Filter im Resultat — genau die vom Hersteller
geforderte Wartungsfreiheit.

## Teil 4 – Den Filter wirksam machen

Bestehende GPO aus Auftrag 07 um Item-Level-Targeting erweitert (bzw. neu nach gleicher
Namenskonvention angelegt):

- **GPO-Name:** `GPO_Extern_VPN-Anleitung`
- **Verknüpft mit:** Domain-Stammebene
- **Einstellung:** *Benutzerkonfiguration → Einstellungen → Windows-Einstellungen →
  Verknüpfungen* → Desktop-Verknüpfung auf die interne VPN-Anleitungsseite
- **Zielgruppenadressierung (Item-Level-Targeting):** neues Element vom Typ **LDAP Query**, als
  Abfrage der Filter aus 3.3, Base-DN `DC=firma,DC=local`

Nachweis:

```powershell
gpresult /r
```

- Auf einem Testclient eines **Promoters**: Die GPO `GPO_Extern_VPN-Anleitung` erscheint unter
  «Angewendete Gruppenrichtlinienobjekte», die Verknüpfung liegt nach `gpupdate /force` auf dem
  Desktop.
- Auf einem Testclient eines **internen Benutzers**: Die GPO erscheint unter «Die folgenden GPOs
  wurden nicht angewendet, weil sie gefiltert wurden» mit dem Grund «Zielgruppenadressierung
  fehlgeschlagen» — die Verknüpfung fehlt auf dem Desktop.

## Teil 5 – Konfigurationsblatt

> Das Original-Formular `resources/firewall-konfigurationsblatt.md` lag mir nicht vor. Die
> nachfolgende Tabelle bildet die darin üblicherweise verlangten Felder (Bind-Daten, Base-DN,
> Suchfilter, Port) nach den Angaben aus dem Auftrag nach. Feldbezeichnungen ggf. an das
> tatsächliche Blatt anpassen — die Werte selbst sind die eigentliche Abgabe.

| Feld | Wert |
|---|---|
| Server / FQDN | `dc01.firma.local` |
| Port / Protokoll | **636 (LDAPS)** — Begründung siehe unten |
| Bind-DN | `CN=svc-ldap-firewall,OU=Dienstkonten,DC=firma,DC=local` |
| Bind-Passwort | *(separat, ausserhalb dieses Dokuments, im Passwort-Tresor hinterlegt)* |
| Base-DN | `DC=firma,DC=local` |
| Such-Scope | Subtree |
| Such-Filter (Benutzer, aktiv, extern) | `(&(objectCategory=person)(objectClass=user)(memberOf:1.2.840.113556.1.4.1941:=CN=extern,OU=Gruppen,DC=firma,DC=local)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` |
| Login-Attribut | `sAMAccountName` |

**Begründung Dienstkonto:** Es wurde ein dediziertes, nicht-privilegiertes Dienstkonto
`svc-ldap-firewall` angelegt statt einen bestehenden Administrator oder einen anonymen Zugriff zu
verwenden. Es erhält ausschliesslich **Leserechte** (`Read All Properties`, `List Contents`) auf
`DC=firma,DC=local`, da der Filter Objekte über die gesamte Domäne hinweg (transitive
Gruppenauflösung, Attribute `memberOf`, `userAccountControl`, `sAMAccountName`) lesen muss. Es
benötigt ausdrücklich **keine** Schreibrechte, keine Mitgliedschaft in privilegierten Gruppen wie
`Domänen-Admins`, kein Interactive Logon und keine Berechtigung, Passwörter zurückzusetzen oder
Objekte zu verändern — die Firewall soll ausschliesslich lesend abfragen, niemals das Verzeichnis
ändern. Ein anonymer Zugriff scheidet aus, da anonyme Binds in AD grösstenteils gesperrt sind und
auch aus Nachvollziehbarkeitsgründen (Audit-Log) ein benanntes Konto sinnvoller ist; ein
bestehender Admin-Account scheidet aus, weil er massiv mehr Rechte mitbringt als für eine reine
Leseabfrage nötig (Prinzip der geringsten Berechtigung).

**Begründung Port 636 statt 389:** Über einen einfachen Bind auf Port 389 würden sowohl der
Bind-DN als auch das Bind-Passwort des Dienstkontos **im Klartext** über das Netz übertragen,
ebenso die Suchergebnisse selbst (u. a. Gruppenzugehörigkeiten). Da die Firewall diese Bind- und
Suchvorgänge regelmässig automatisiert wiederholt, wäre das Passwort des Dienstkontos bei jedem
Zyklus im Netzwerk mitschneidbar (z. B. per Paket-Mitschnitt zwischen Firewall und DC). **LDAPS
(Port 636)** baut vor jeder LDAP-Operation eine TLS-verschlüsselte Verbindung auf, wodurch Bind
und Suchergebnisse vollständig verschlüsselt übertragen werden. Voraussetzung dafür ist ein
gültiges Serverzertifikat auf dem Domain Controller, dem die Firewall vertraut (eigenes internes
CA-Zertifikat oder öffentliches Zertifikat).

## ki-log.md (Auszug)

| Datum | Prompt (Kurzfassung) | Zweck | Wie verifiziert |
|---|---|---|---|
| — | „Ich habe folgenden LDAP-Suchfilter geschrieben: `(memberOf=CN=extern,...)`. Ziel: alle externen Benutzer finden. Erkläre Schritt für Schritt, welche Objekte dieser Filter zurückgibt." | Verständnis, warum der naive Filter 0 Treffer liefert | Filter in PowerShell ausgeführt → tatsächlich 0 Treffer, wie von der KI vorhergesagt |
| — | „Mein Benutzer ist Mitglied der Gruppe A, Gruppe A ist Mitglied der Gruppe B. Erkläre, warum eine Suche nach der Mitgliedschaft in B ihn nicht findet. Nenne mir das Prinzip, nicht den fertigen Filter." | Prinzip der transitiven Gruppenauflösung verstehen, ohne die Lösung geliefert zu bekommen | Selbst nach `LDAP_MATCHING_RULE_IN_CHAIN` recherchiert, Filter selbst gebaut, mit 3.2 verifiziert (12 statt 0 Treffer) |
| — | „Wie formuliert LDAP eine Bit-Abfrage auf ein Attribut wie `userAccountControl`?" | Syntax der bitweisen Matching-Rule (`1.2.840.113556.1.4.803`) | Filter mit deaktiviertem Testkonto verifiziert (Test 1 in Teil 3) |

Vollständige Reflexion gemäss [ki-nutzung.md](../../ki-nutzung.md) in der separaten Datei
`ki-log.md` im Auftragsordner abgelegt.
