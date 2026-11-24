---
title: "S/MIME mit Organisationseintrag: organizationIdentifier und UID im Zertifikat"
navTitle: "S/MIME mit Organisation"
description: "Was ein S/MIME-Zertifikat mit Organisationseintrag enthält, wie die UID als organizationIdentifier ins Zertifikat kommt und was die Zertifizierungsstelle dafür prüft. Dazu ein Vergleich der Produkte von SwissSign und DigiCert, Befehle zur Prüfung des Zertifikatstyps und die Abgrenzung zum ZertES."
date: "2026-12-15"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "11 Min. Lesezeit"
themen:
  - "e-mail-verschluesselung"
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "smime-organisation-uid"
translationId: "article-e0f9bcc57f360835"
url: "https://rafaelpfister.ch/blog/smime-organisation-uid"
---

# S/MIME mit Organisationseintrag: organizationIdentifier und UID im Zertifikat

Zertifizierungsstellen (CA) unterscheiden S/MIME-Zertifikate mit und ohne Organisationseintrag. Ein Zertifikat mit Organisationseintrag weist neben der Mailadresse auch die Firma aus, zu der das Postfach gehört: mit ihrem geprüften Namen und mit einer Registernummer. Für Schweizer Unternehmen ist diese Registernummer die UID, also die CHE-Nummer aus dem Handelsregister.

Kurzfazit: Einen Organisationseintrag gibt es nur bei den Zertifikatstypen `Organization-validated` und `Sponsor-validated` der S/MIME Baseline Requirements. Er besteht aus dem Firmennamen im Feld `O` und dem Feld `organizationIdentifier` (OID 2.5.4.97), bei einer Schweizer Firma zum Beispiel `NTRCH-CHE-123.456.789`. Die CA prüft die Firma in einem amtlichen Register, bevor sie diese Angaben ins Zertifikat schreibt. Ein mailbox-validiertes Zertifikat belegt dagegen nur den Zugriff auf ein Postfach. Bei SwissSign entspricht das der Wahl zwischen Silver und Gold, bei DigiCert der Wahl zwischen «Secure Email for Individual Mailbox» und den Produkten «for Employee» und «for Organization». Eine qualifizierte Signatur nach ZertES entsteht durch den Organisationseintrag nicht.

**Schweiz-Fokus:** Die rechtliche Einordnung folgt dem Schweizer Recht (ZertES, OR). Wo die EU-Regeln abweichen, steht ein gekennzeichneter Hinweis.

## Die vier Zertifikatstypen

Seit dem 1. September 2023 gelten für öffentlich vertrauenswürdige S/MIME-Zertifikate die `S/MIME Baseline Requirements` (BR) des CA/Browser Forum. Sie unterscheiden vier Zertifikatstypen danach, wer als Inhaber (`Subject`) im Zertifikat steht:

| Typ | Inhaber im Zertifikat | Organisationseintrag | Was die CA prüft |
|---|---|---|---|
| `Mailbox-validated` (MV) | nur die Mailadresse | nicht zulässig | Kontrolle über das Postfach oder die Maildomain |
| `Organization-validated` (OV) | eine Organisation, keine Person | Pflicht | Organisation sowie Postfach oder Maildomain |
| `Sponsor-validated` (SV) | eine Person als Angehörige einer Organisation | Pflicht | wie OV, dazu die Person; diese prüft die CA oder die Organisation selbst |
| `Individual-validated` (IV) | eine Person ohne Organisation | nicht zulässig | Identität der Person und Postfach |

Mit «S/MIME mit Organisationseintrag» sind die Typen OV und SV gemeint. Für beide schreiben die BR die Felder `organizationName` und `organizationIdentifier` zwingend vor, und zwar in allen drei Generationen `Legacy`, `Multipurpose` und `Strict`. Die Generationen unterscheiden sich bei den zulässigen Subject-Feldern, Erweiterungen und Verwendungszwecken. Die Generation `Legacy` haben die BR per 15. Juli 2025 abgekündigt; SwissSign und DigiCert stellen sie seither nicht mehr aus.

OV-Zertifikate eignen sich für Funktionspostfächer wie `info@` oder `buchhaltung@`, die keiner Person zugeordnet sind. SV-Zertifikate sind für persönliche Postfächer von Mitarbeitenden gedacht und enthalten zusätzlich den Namen der Person oder ein Pseudonym.

## Aufbau des organizationIdentifier

Das Feld `organizationIdentifier` (OID 2.5.4.97) enthält eine Registernummer der Organisation. Dasselbe Feld verwenden auch Zertifikate nach eIDAS und ZertES. Für S/MIME-Zertifikate schreiben die BR folgenden Aufbau vor:

| Teil | Beispiel | Bedeutung |
|---|---|---|
| Registrierungsschema, 3 Zeichen | `NTR` | `National Trade Register`, das Handelsregister |
| Ländercode nach ISO 3166-1 | `CH` | Land, in dem das Register geführt wird |
| Bindestrich | `-` | Trennzeichen |
| Registernummer | `CHE-123.456.789` | Nummer der Organisation im Register |

Zusammengesetzt ergibt das `NTRCH-CHE-123.456.789`. Weil die UID selbst einen Bindestrich enthält, legen die BR fest, dass Schema und Ländercode nie einen Bindestrich enthalten. Der erste Bindestrich von links trennt also das Schema von der Registernummer, alle weiteren gehören zur Nummer. Vergibt ein Register Nummern nur auf der Ebene eines Bundesstaats oder einer Provinz, folgt auf den Ländercode ein `+` mit dem Code der Region, etwa `NTRUS+CA-12345678` für Kalifornien.

Die BR kennen diese Registrierungsschemata:

| Schema | Herkunft der Nummer | Beispiel aus den BR |
|---|---|---|
| `NTR` | Handels- oder Unternehmensregister | `NTRGB-12345678` |
| `VAT` | Steuerbehörde (Mehrwertsteuernummer) | `VATEL-123456789` |
| `PSD` | Zulassungsnummer eines Zahlungsdienstleisters nach der EU-Richtlinie 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` nach ISO 17442, Ländercode immer `XG` | `LEIXG-` gefolgt vom LEI |
| `GOV` | Behörden ohne Nummer aus einem der übrigen Schemata | `GOVUS` |
| `INT` | internationale Organisationen | `INTXG` |

Für Schweizer Unternehmen ist `NTR` mit der UID der Regelfall. Die UID vergibt das Bundesamt für Statistik. Die Handelsregister verwenden sie als Firmennummer, sie dient auch als Mehrwertsteuernummer, und sie bleibt gleich, wenn eine Firma ihren Sitz in einen anderen Kanton verlegt. Eine Mehrwertsteuernummer kann dagegen entfallen, wenn die Steuerpflicht endet, und nicht jede Organisation hat eine. SwissSign übernimmt die UID mit den Punkten, so wie sie im UID-Register steht, und nennt als Beispiel `NTRCH-CHE-109.357.012` für die SwissSign AG.

**EU-Hinweis:** In Deutschland vergeben die Amtsgerichte die Handelsregisternummern, und dieselbe Nummer kommt bei mehreren Gerichten vor. Seit Version 1.0.9 vom Mai 2025 erlauben die BR deshalb im Schema `NTR` die `European Unique Identifier` (EUID) und empfehlen sie für deutsche Firmen, etwa `NTRDE-DER3306.HRB12345` für eine Firma im Handelsregister des Amtsgerichts Köln. Eindeutig muss die Registernummer nach den BR nur dort sein, wo Register und Rechtsordnung eindeutige Nummern vergeben.

## Was die CA prüft

Bevor eine CA ein OV- oder SV-Zertifikat ausstellt, muss sie nach den BR folgende Angaben prüfen:

- **Name und Existenz der Organisation:** über eine Behörde im Land der Organisation, über LEI-Daten, einen Besuch vor Ort oder eine Bestätigung mit Registerauszug. Für eine Schweizer Firma sind das das Handelsregister und das UID-Register. Ist die Firma dort als gelöscht oder inaktiv geführt, darf die CA das Zertifikat nicht ausstellen.
- **Registernummer:** Die CA bestätigt, dass die Nummer im `organizationIdentifier` zur Organisation im Feld `O` gehört.
- **Vollmacht:** Über einen zuverlässigen Kommunikationsweg prüft die CA, dass die antragstellende Person für die Organisation Zertifikate beantragen oder als `Enterprise RA` handeln darf.
- **Postfach oder Maildomain:** Die CA prüft die Kontrolle über die Mailadressen im Zertifikat. Handelt die Organisation als `Enterprise RA`, muss sie die Kontrolle über die ganze Maildomain nachweisen.
- **Person (nur SV):** Die BR erlauben, dass die Organisation als `Enterprise RA` ihre Mitarbeitenden selbst bestätigt. Die CA prüft dann die Organisation und nicht jede einzelne Person.

Die Ergebnisse dieser Prüfungen darf eine CA wiederverwenden: die Prüfung der Organisation und der Vollmacht während 825 Tagen, die Prüfung der Domainkontrolle während 398 Tagen. Ein Zertifikat der Generationen `Multipurpose` und `Strict` ist höchstens 825 Tage gültig.

## SwissSign und DigiCert im Vergleich

Beide CAs bieten die Typen MV, SV und OV unter eigenen Produktnamen an. Ein IV-Zertifikat führt keine der beiden. Den Aufbau des `organizationIdentifier` legen die BR fest, er gilt für beide Anbieter gleich.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (früher Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | nicht im Angebot | nicht im Angebot |
| Generation bei SV und OV | `Multipurpose` laut Zertifikatsprofilen | `Strict` als Standard, `Multipurpose` wählbar |
| Person bei SV | Vor- und Nachname oder Pseudonym mit Präfix `pseudo:` im `CN` | Vor- und Nachname oder Pseudonym; im `CN` wahlweise der Name oder die Mailadresse |
| Zusätzliche Verwendungen bei SV | im Profil mit Authentifizierung: Client-Authentifizierung, Smartcard-Anmeldung, EFS | bei `Strict` Nichtabstreitbarkeit, bei `Multipurpose` zusätzlich Datenverschlüsselung und Client-Authentifizierung |

DigiCert beschreibt das SV-Produkt ausdrücklich nach dem Modell der `Enterprise RA`: Die Organisation bestätigt, dass die Person im Zertifikat zu ihren Mitarbeitenden oder Vertretern gehört, und bewahrt die Belege dafür auf. DigiCert selbst prüft nur die Organisation. Bei OV und SV verlangt DigiCert immer den Nachweis der Domainkontrolle. Die Organisationsprüfung gilt 825 Tage, die Domainprüfung 398 Tage. Als Laufzeit stehen ein oder zwei Jahre oder ein eigenes Ablaufdatum bis höchstens 824 Tage zur Wahl. Die früheren Produkte Premium, Email Security Plus, Digital Signature Plus und Class 1 S/MIME hat DigiCert am 10. Juli 2025 eingestellt und durch diese Produkte ersetzt.

Bei SwissSign enthält das OV-Zertifikat im `CN` den Namen der Organisation wie im Feld `O`. Welche Stufe eine Bank für welches Postfach wählen sollte und was das rechtlich bedeutet, beschreibt der Artikel [S/MIME bei Banken: Rechtslage in der Schweiz und die Wahl zwischen Silver und Gold](/blog/smime-silver-gold-schweiz).

## Den Zertifikatstyp prüfen

In Outlook und anderen Mailprogrammen lassen sich die Zertifikatsdetails einer signierten Mail über das Signatursymbol öffnen. Unter Windows stehen die Felder im Register «Details» beim Eintrag «Antragsteller». Für die OID 2.5.4.97 kennt Windows keinen Anzeigenamen und zeigt sie als `OID.2.5.4.97` an.

### Inhaber mit PowerShell anzeigen

Für ein exportiertes Zertifikat (zum Beispiel aus den Zertifikatsdetails einer signierten Mail):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-PfxCertificate` | liest ein Zertifikat aus einer Datei, auch aus einer `.cer`-Datei ohne privaten Schlüssel |
| `-FilePath .\zertifikat.cer` | Pfad zur Zertifikatsdatei |
| `SubjectName.Format($true)` | gibt die Felder des Inhabers zeilenweise aus |

</details>

Ausgabe für ein SV-Zertifikat (Beispiel):

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

Bei einem MV-Zertifikat enthält die Ausgabe nur `CN=` mit der Mailadresse.

### Typ und Generation über die Policy-OID bestimmen

Jedes Zertifikat nach den BR enthält in der Erweiterung `Certificate Policies` eine OID nach dem Muster `2.23.140.1.5.<Typ>.<Generation>`:

| Stelle | Wert | Bedeutung |
|---|---|---|
| Typ | `1` | `Mailbox-validated` |
| Typ | `2` | `Organization-validated` |
| Typ | `3` | `Sponsor-validated` |
| Typ | `4` | `Individual-validated` |
| Generation | `1` | `Legacy` |
| Generation | `2` | `Multipurpose` |
| Generation | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `$zert.Extensions` | Erweiterungen des zuvor eingelesenen Zertifikats |
| `'2.5.29.32'` | OID der Erweiterung `Certificate Policies` |
| `Format($true)` | gibt den Inhalt der Erweiterung mehrzeilig aus |

</details>

Ausgabe (gekürzt):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` steht für ein SV-Zertifikat der Generation `Multipurpose`. Weitere Einträge mit OIDs der CA selbst sind normal.

### S/MIME-Zertifikate im eigenen Zertifikatsspeicher auflisten

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Cert:\CurrentUser\My` | persönlicher Zertifikatsspeicher des angemeldeten Benutzers |
| `EnhancedKeyUsageList.ObjectId` | erweiterte Schlüsselverwendungen des Zertifikats |
| `'1.3.6.1.5.5.7.3.4'` | OID für E-Mail-Schutz (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | zeigt Inhaber und Ablaufdatum |

</details>

### Mit OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `x509` | Unterbefehl für Zertifikate |
| `-in zertifikat.pem` | Eingabedatei im PEM-Format; für eine DER-Datei zusätzlich `-inform der` |
| `-noout` | unterdrückt die Ausgabe des kodierten Zertifikats |
| `-subject` | gibt den Inhaber aus |
| `-nameopt multiline` | ein Feld pro Zeile mit ausgeschriebenen Feldnamen |

</details>

OpenSSL 3 kennt den Feldnamen und zeigt ihn ausgeschrieben an:

```text
subject=
    commonName                = Max Muster
    givenName                 = Max
    surname                   = Muster
    organizationName          = Muster AG
    organizationIdentifier    = NTRCH-CHE-123.456.789
    stateOrProvinceName       = ZH
    countryName               = CH
```

### Registernummer gegenprüfen

Als Empfänger einer signierten Mail können Sie den Organisationseintrag in vier Schritten prüfen:

1. Das Mailprogramm meldet die Signatur als gültig, und das Zertifikat stammt von einer CA, der das Mailprogramm vertraut.
2. Die Policy-OID beginnt mit `2.23.140.1.5.2` oder `2.23.140.1.5.3`, und das Zertifikat enthält die Felder `O` und `organizationIdentifier`.
3. Die UID aus dem `organizationIdentifier` gehört im UID-Register zur erwarteten Firma, und die Firma ist dort aktiv.
4. Die Domain der Absenderadresse gehört zu dieser Firma.

## Abgrenzung zum ZertES

Der Organisationseintrag macht ein S/MIME-Zertifikat nicht zu einem geregelten Zertifikat nach ZertES. Geregelte Zertifikate stellen nur nach ZertES anerkannte Anbieterinnen aus, und das Zertifikat muss den Hinweis enthalten, dass es sich um ein geregeltes Zertifikat handelt (Art. 2 Bst. g und Art. 7 Abs. 2 Bst. b ZertES).

Inhaltlich gibt es Parallelen. Ein geregeltes Zertifikat für eine UID-Einheit muss deren UID enthalten (Art. 7 Abs. 2 Bst. e ZertES), so wie der `organizationIdentifier` sie bei OV und SV enthält. Ein Zertifikat für eine natürliche Person kann zudem den Hinweis enthalten, dass sie zur Vertretung einer bestimmten UID-Einheit berechtigt ist (Art. 7 Abs. 3 Bst. b ZertES); das entspricht im Aufbau einem SV-Zertifikat.

| | S/MIME mit Organisationseintrag | geregeltes Zertifikat nach ZertES |
|---|---|---|
| Grundlage | S/MIME Baseline Requirements des CA/Browser Forum | ZertES und Ausführungsbestimmungen |
| Ausstellerin | öffentlich vertrauenswürdige CA | nach ZertES anerkannte Anbieterin |
| Organisationskennung | `organizationIdentifier`, bei Schweizer Firmen mit der UID | UID, Pflicht bei UID-Einheiten |
| Prüfung der Person | bei SV durch die CA oder durch die Organisation als `Enterprise RA` | grundsätzlich persönliches Erscheinen mit Identitätsnachweis; bei UID-Einheiten eine Vertretung mit Nachweis der Vertretungsmacht (Art. 9 ZertES) |
| Schlüssel | nach den Vorgaben der CA, auch auf einem Gateway | Anforderungen nach Art. 6 ZertES |

Der eigenhändigen Unterschrift gleichgestellt ist nach Art. 14 Abs. 2bis OR nur die qualifizierte elektronische Signatur mit qualifiziertem Zeitstempel, und ein qualifiziertes Zertifikat darf nur auf eine natürliche Person ausgestellt werden (Art. 8 Abs. 1 ZertES). Für die Signatur einer Organisation sieht das ZertES das geregelte elektronische Siegel vor: eine fortgeschrittene elektronische Signatur, die mit einer sicheren Siegelerstellungseinheit erstellt wird und auf einem geregelten Zertifikat einer UID-Einheit beruht (Art. 2 Bst. d ZertES). Eine S/MIME-signierte Mail ist ein Beweismittel, dessen Wert das Gericht frei würdigt (Art. 157 ZPO); ein geprüfter Organisationseintrag stützt dabei die Zuordnung zur absendenden Firma.

**EU-Hinweis:** In der EU regelt die eIDAS-Verordnung elektronische Signaturen und Siegel. Nach Art. 25 Abs. 2 eIDAS ist ebenfalls nur die qualifizierte elektronische Signatur der handschriftlichen Unterschrift gleichgestellt.

## Empfehlung

- **Funktionspostfächer:** OV-Zertifikat (SwissSign Organization Validated, DigiCert Secure Email for Organization). Es weist die Firma aus, ohne eine Person zu nennen.
- **Persönliche Postfächer mit Signatur gegenüber Kunden und Partnern:** SV-Zertifikat (SwissSign Gold, DigiCert Secure Email for Employee). Es weist Person und Firma aus.
- **Nur Verschlüsselung:** Kryptografisch sind alle Typen gleichwertig. Für die Verschlüsselung allein genügt ein MV-Zertifikat.
- **Prüfungen einplanen:** Die Organisationsprüfung gilt 825 Tage, die Domainprüfung 398 Tage. Nach einer Änderung von Firmenname oder Sitz stimmen die Angaben im Zertifikat nicht mehr mit dem Register überein; planen Sie dann neue Zertifikate ein.
- **Beim Empfang prüfen:** Policy-OID und UID zeigen, ob eine Signatur tatsächlich einer geprüften Firma zugeordnet ist.

## Quellen

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Version 1.0.17 vom 20. November 2026; Zertifikatstypen und Policy-OIDs (Abschnitt 1.1), Prüfung der Organisation (3.2.3), Prüfung der Vollmacht (3.2.6), Wiederverwendung von Prüfergebnissen (4.2.1), Laufzeiten (6.3.2), Subject-Felder (7.1.4.2) und Registrierungsschemata (Anhang A).

2.  [cabforum/smime auf GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown-Fassung der S/MIME Baseline Requirements mit Versionsverlauf, unter anderem EUID ab Version 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Produktbezeichnungen Silver und Gold, Beispiel `NTRCH-CHE-109.357.012`, UID als Registernummer und EUID für deutsche Firmen (28.04.2025).

4.  [SwissSign CPR S/MIME](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Zertifikatsprofile von SwissSign für MV, SV und OV mit Subject-Feldern, Verwendungszwecken und Policy-OIDs, Version 14 vom 18.09.2025.

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Produktübersicht mit Zuordnung zu MV, OV und SV, Ende der Generation `Legacy` und der früheren S/MIME-Produkte am 10. Juli 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organisations- und Domainprüfung, Bestätigung der Mitarbeitenden durch die Organisation, Profile `Strict` und `Multipurpose`, Laufzeiten.

7.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Art. 2 (Begriffe, geregeltes Siegel), Art. 6 (Schlüssel), Art. 7 (Inhalt geregelter Zertifikate, UID), Art. 8 (qualifizierte Zertifikate) und Art. 9 (Identitätsprüfung).

8.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 Abs. 2bis zur Gleichstellung der qualifizierten elektronischen Signatur mit der eigenhändigen Unterschrift.

9.  [UID-Register](https://www.uid.admin.ch): Abfrage einer UID mit Name, Sitz und Status, geführt vom Bundesamt für Statistik.

10.  [Zefix](https://www.zefix.ch): zentraler Firmenindex der kantonalen Handelsregister.

11.  [Zivilprozessordnung (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 zur freien Beweiswürdigung.

12.  [Verordnung (EU) Nr. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): Art. 25 Abs. 2 zur Gleichstellung der qualifizierten elektronischen Signatur mit der handschriftlichen Unterschrift.
