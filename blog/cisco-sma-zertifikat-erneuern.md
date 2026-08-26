---
title: "Zertifikat auf der Cisco SMA erneuern"
navTitle: "SMA-Zertifikat"
description: "Zertifikate lassen sich auf der Cisco SMA nur über die CLI einspielen, und aktuelle AsyncOS-Versionen validieren beim Import die komplette Kette: Ohne hinterlegte Root-CA scheitert er. Der Artikel zeigt die Wege zum neuen Schlüsselpaar, den OpenSSL-Weg im Detail, den Umgang mit dem RC2-40-CBC-Fehler von OpenSSL 3 und den Import der internen Root-CA in den Truststore der Appliance."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 Min. Lesezeit"
themen:
  - "cisco-esa-sma"
  - "smtp-mailflow"
produkte:
  - "cisco"
protokolle:
  - "tls"
  - "smtp"
  - "ldap"
  - "troubleshooting"
hauptthema: "cisco-esa-sma"
slug: "cisco-sma-zertifikat-erneuern"
translationId: "article-69d93a1e5e081848"
url: "https://rafaelpfister.ch/blog/cisco-sma-zertifikat-erneuern"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
---
# Zertifikat auf der Cisco SMA erneuern

Die Cisco SMA (Security Management Appliance, inzwischen unter dem Namen Cisco Secure Email and Web Manager geführt) übernimmt in vielen Mailumgebungen die zentrale Spam-Quarantäne und das Reporting für die Secure-Email-Gateways. Ihr HTTPS-Zertifikat deckt die Admin-GUI und die Quarantäneseite ab, auf der Endbenutzer ihre zurückgehaltenen Mails sichten und freigeben. Läuft es ab, bricht kein Mailfluss zusammen. Sichtbar wird der Ablauf trotzdem sofort: Jeder Aufruf der Quarantäneseite endet mit einer Zertifikatswarnung im Browser, und ausgerechnet die Benutzer, denen Awareness-Schulungen beibringen, bei solchen Warnungen nicht weiterzuklicken, sollen sie dann ignorieren.

Bei einer Erneuerung in einem Kundenprojekt gab es gleich zwei Stolpersteine: Erst quittierte OpenSSL 3 die PFX-Datei der internen CA mit einem kryptischen Fehler zu `RC2-40-CBC`, dann verweigerte die Appliance den Import des fertigen Zertifikats, weil ihr die ausstellende Root-CA nicht bekannt war. Beide Hürden samt Lösung weiter unten.

## Was die SMA anders macht als die ESA

Auf der ESA lässt sich der komplette Zertifikatslebenszyklus über die GUI abwickeln (`Network > Certificates`). Die SMA kann das nicht: Das Serverzertifikat wird ausschliesslich über die CLI eingespielt, mit dem Befehl `certconfig` in einer SSH-Session. Die GUI der SMA zeigt Zertifikate nur an; einzig die Listen der vertrauenswürdigen Zertifizierungsstellen lassen sich dort pflegen, dazu später mehr.

Dazu kommen zwei weitere Eigenheiten:

- Der Einfüge-Dialog akzeptiert nur das PEM-Format. Eine PFX-Datei (PKCS#12) muss vor der Installation konvertiert werden; aktuelle AsyncOS-Versionen bieten daneben einen direkten PKCS#12-Import an, dafür muss die Datei aber erst auf die Appliance gelangen.
- Ältere AsyncOS-Versionen (der Stand der Cisco-Technote) erzeugen selbst weder Schlüssel noch CSRs, das Schlüsselpaar muss ausserhalb entstehen; die drei gangbaren Wege dazu weiter unten. Aktuelle Versionen können mit `certconfig > CERTIFICATE > NEW` ein Self-Signed-Zertifikat samt CSR direkt auf der Appliance erzeugen. Für ein gemeinsames Zertifikat über mehrere Appliances hilft das jedoch nicht, weil der private Schlüssel die Appliance dabei nie verlässt.

Ein einzelnes Zertifikat kann wahlweise alle Dienste bedienen (eingehendes und ausgehendes TLS, HTTPS-Verwaltungszugriff, LDAPS) oder pro Dienst separat hinterlegt werden. Gesteuert wird das im `certconfig`-Dialog; die Kopfzeile des Befehls zeigt jederzeit die aktive Zuweisung (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Eine separate Zuweisungsmaske wie auf der ESA gibt es nicht, und über die GUI lässt sich daran nichts ändern. In den meisten Umgebungen ist ein Zertifikat für alles die pragmatische Wahl: Die Namensliste deckt die FQDNs der Appliances ohnehin ab, und getrennte Schlüsselpaare vervielfachen den Aufwand bei jeder Erneuerung.

Dass der Dialog auf einer Quarantäne-Appliance nach Inbound- und Outbound-TLS fragt, irritiert auf den ersten Blick, denn die SMA steht in keinem MX-Pfad. Sie spricht trotzdem SMTP in beide Richtungen. Inbound (Receiving) ist die Annahmeseite: Die ESAs liefern quarantänisierte Nachrichten per SMTP an die SMA, in die zentrale Spam-Quarantäne auf Port 6025 und in die zentralen Policy-, Virus- und Outbreak-Quarantänen auf Port 7025; letztere Verbindungen sind ab Werk TLS-verschlüsselt, und dabei präsentiert die SMA genau dieses Zertifikat. Outbound (Delivery) ist die Sendeseite: Gibt ein Benutzer eine Nachricht aus der Quarantäne frei, stellt die SMA sie selbst über ihre SMTP-Routen zurück in den Mailfluss zu, und auch Quarantäne-Benachrichtigungen, geplante Reports und Alerts verschickt die Appliance als eigene Mails. Für die Erneuerung heisst das: Kritisch ist in der Praxis HTTPS, die beiden SMTP-Dienste laufen beim Zertifikat für alle Dienste einfach mit.

## Namen festlegen: CN und SAN

Unabhängig vom Weg zum Schlüsselpaar steht zuerst die Namensliste fest. Der Common Name gehört auf den Hostnamen, unter dem die Benutzer die Quarantäneseite aufrufen. In die SAN-Liste gehören zusätzlich die FQDNs der Appliances, damit auch der direkte Zugriff auf die Admin-GUI ohne Warnung funktioniert. Für eine Umgebung mit zwei Appliances sieht die Namensliste so aus:

| Feld | Wert |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Zwei Hinweise dazu: Browser werten seit langem nur noch die SAN-Einträge aus, der CN allein genügt nicht. Der Quarantäne-Hostname muss deshalb auch als SAN erscheinen. Und kurze Hostnamen ohne Domainanteil (etwa `SMA01`) stellt nur eine interne CA aus; öffentliche CAs signieren keine internen Namen.

## Drei Wege zum neuen Schlüsselpaar

Für ein Zertifikat, das mehrere Appliances und den Quarantäne-Hostnamen abdeckt, muss das Schlüsselpaar ausserhalb der Appliance entstehen. Drei Wege haben sich etabliert:

1. Schlüssel und CSR mit OpenSSL innerhalb der eigenen Umgebung erzeugen. Der private Schlüssel entsteht dort, wo er gebraucht wird, und verlässt die Umgebung nie. Der empfohlene Weg, Details im nächsten Abschnitt.
2. Die CA erzeugt das Schlüsselpaar und liefert eine PFX-Datei. Funktioniert, hat aber zwei Haken: Der Schlüssel reist durch fremde Hände (das Passwort gehört deshalb in einen separaten Kanal und nicht in dieselbe Mail wie die Datei), und je nach CA-Werkzeug kommt eine RC2-verschlüsselte PFX zurück, die OpenSSL 3 nur mit Zusatzaufwand öffnet; dazu unten mehr.
3. Der Umweg über eine ESA, dokumentiert in der Cisco-Technote: Dort unter `Network > Certificates` ein Zertifikat mit dem CN der SMA anlegen, den CSR herunterladen und von der CA signieren lassen, das signierte Zertifikat wieder auf die ESA hochladen und das Ganze als PFX exportieren. Auch hier steht am Ende die Konvertierung nach PEM an.

## OpenSSL unter Windows starten

Alle folgenden Schritte laufen über OpenSSL, auf einem System innerhalb der Umgebung, etwa einem Admin-Server. Die Light-Edition der Windows-Builds von Shining Light Productions genügt, der Installer ist rund 6 MB gross und lässt sich gegen die von slproweb publizierte Prüfsummenliste verifizieren.

Der Installer legt alles unter `C:\Program Files\OpenSSL-Win64` ab, die ausführbare Datei liegt in `bin\openssl.exe`. In den Suchpfad trägt er sich nicht ein: Wer in einer frischen Eingabeaufforderung `openssl` tippt, bekommt eine Fehlermeldung. Es gibt drei Möglichkeiten:

- Im Startmenü den Eintrag `Win64 OpenSSL Command Prompt` aufrufen. Er startet die `start.bat` aus dem Installationsverzeichnis, setzt die Umgebung und begrüsst mit der Ausgabe von `openssl version -a`. In diesem Fenster funktioniert `openssl` direkt.
- Den vollständigen Pfad angeben: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- `C:\Program Files\OpenSSL-Win64\bin` dauerhaft in die Umgebungsvariable `Path` eintragen, danach steht `openssl` in jeder Shell zur Verfügung.

Ohne zusätzliche Installation kommt aus, wer Git für Windows schon einsetzt: Es bringt sein eigenes OpenSSL mit (`C:\Program Files\Git\mingw64\bin\openssl.exe`), in der Git Bash liegt es sofort im Suchpfad. Aktuelle Git-Versionen liefern OpenSSL 3.5 samt aktivem Legacy-Provider, das `-legacy` aus dem Abschnitt zur PFX-Konvertierung funktioniert dort also ebenfalls. Kontrollieren lässt sich das so:

```bash
openssl list -providers -provider legacy
```

Eine Stolperfalle hat die Git Bash allerdings: Sie hält Argumente, die mit `/` beginnen, für Pfade und schreibt sie um. Aus `-subj "/C=CH/O=Example AG/CN=..."` wird `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, und OpenSSL bricht ab:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Ein vorangestelltes `MSYS_NO_PATHCONV=1` schaltet die Umschreibung für den einzelnen Aufruf ab. In Eingabeaufforderung, PowerShell und im OpenSSL-Command-Prompt tritt das Problem nicht auf.

## Schlüssel und CSR mit OpenSSL erzeugen

Ein einzelner Aufruf erzeugt Schlüssel und CSR mit der kompletten SAN-Liste:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

Die CSR-Datei geht an die CA, der Schlüssel bleibt auf dem Server. Zurück kommt das signierte Zertifikat samt Intermediate, üblicherweise direkt als PEM. Damit liegt alles für die Installation bereit, die PFX-Konvertierung entfällt auf diesem Weg komplett.

Die Schlüsseldatei ist unverschlüsselt (`-noenc`), weil `certconfig` sie genau so erwartet. Bis zur Installation bleibt sie unter restriktiven Berechtigungen auf dem Server, danach wird sie gelöscht oder in die Passwortverwaltung verschoben.

## PFX nach PEM konvertieren

Dieser und der nächste Abschnitt betreffen die Wege 2 und 3, an deren Ende eine PFX-Datei steht. `certconfig` erwartet Zertifikat und privaten Schlüssel als PEM, den Schlüssel unverschlüsselt. Beides erledigt ein einzelner OpenSSL-Aufruf:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (bis OpenSSL 3.0 hiess die Option `-nodes`) schreibt den privaten Schlüssel ohne Passphrase in die Ausgabedatei. Die Abfrage des Import-Passworts erfolgt ohne Echo, es erscheinen auch keine Sternchen. Die entstandene PEM-Datei enthält Zertifikat, Schlüssel und mitgelieferte Kettenzertifikate in einer Datei und ist entsprechend schützenswert: Nach der Installation löschen oder in die Passwortverwaltung verschieben.

## Wenn OpenSSL 3 die PFX-Datei verweigert

Bei älteren PFX-Dateien bricht die Konvertierung unter OpenSSL 3.x mit dieser Meldung ab:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

Die Ursache ist keine defekte Datei, sondern eine Designentscheidung: OpenSSL 3 hat Altalgorithmen wie RC2, RC4 und DES in einen separaten Legacy-Provider ausgelagert, der standardmässig nicht geladen wird. Viele PFX-Exporte älterer Windows-Systeme und CA-Werkzeuge verschlüsseln den Zertifikatsteil des Containers aber genau mit RC2-40-CBC. OpenSSL 1.1 öffnete solche Dateien anstandslos, OpenSSL 3 lehnt sie ab.

Die Lösung ist eine einzige zusätzliche Option:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` lädt den Legacy-Provider für diesen Aufruf mit, danach läuft die Konvertierung durch. Voraussetzung ist eine OpenSSL-Installation, die den Legacy-Provider mitbringt; bei den gängigen Windows-Builds ist das der Fall.

Wer den Fehler dauerhaft loswerden will, setzt an der Quelle an und lässt die PFX-Datei mit moderner Verschlüsselung exportieren: Aktuelle Export-Dialoge und CA-Werkzeuge bieten AES-256 an, damit entfällt der Legacy-Umweg komplett.

Als grafische Alternative funktioniert XCA (X Certificate and Key Management): Die PFX-Datei über `Importieren > PKCS#12` einlesen, danach das Zertifikat im Tab `Zertifikate` als PEM exportieren und den Schlüssel im Tab `Private Schlüssel` separat als unverschlüsseltes PEM. Beide Exporte werden gebraucht, `certconfig` fragt Zertifikat und Schlüssel einzeln ab. XCA bringt seine eigene Kryptobibliothek mit und öffnet auch Container mit Legacy-Algorithmen.

Noch ein Wort zur Bezugsquelle: Das OpenSSL-Projekt veröffentlicht selbst keine Windows-Binaries, sondern verweist auf Builds Dritter wie Win64 OpenSSL von Shining Light Productions. Download-Portale mit eigenen Installern sind für ein Kryptowerkzeug die falsche Adresse.

## Interne Root-CA zuerst in den Truststore der Appliance

Aktuelle AsyncOS-Versionen validieren beim Anlegen eines Zertifikatsprofils die komplette Kette. Stammt das Zertifikat von einer internen CA, deren Root die Appliance nicht kennt, bricht der Import mit dieser Meldung ab:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

Die Appliance führt zwei Listen vertrauenswürdiger Zertifizierungsstellen: die mitgelieferte Systemliste und eine Custom-Liste für eigene CAs. Die interne Root-CA gehört in die Custom-Liste, und zwar bevor das Serverzertifikat eingespielt wird. Benötigt wird nur das öffentliche CA-Zertifikat als PEM-Datei (`-----BEGIN CERTIFICATE-----` bis `-----END CERTIFICATE-----`), kein privater Schlüssel.

So kommt die Root-CA über die Weboberfläche auf die Appliance:

1. `Network > Certificates` öffnen.
2. Im Abschnitt `Certificate Authorities` auf `Edit Settings` klicken.
3. Bei `Custom List` die Option `Enable` wählen.
4. Über `Choose File` die PEM-Datei hochladen.
5. `Submit` und anschliessend `Commit Changes` ausführen.
6. Unter `Network > Certificates > Manage Trusted Root Certificates` kontrollieren, dass die CA in der Liste der benutzerdefinierten Zertifikate erscheint.

Existiert bereits eine Custom-Liste, diese vorher exportieren und die neue CA an das bestehende PEM-Bundle anhängen: Der Import ersetzt die Liste, sonst verschwinden früher hinterlegte CAs. Bei einer Kette mit Zwischenstufe zuerst die Root-CA importieren, danach die Intermediate-CA. AsyncOS prüft beim Import unter anderem Ablaufdatum, Duplikate und das gesetzte `CA:TRUE`-Flag und lehnt eine Intermediate ab, solange die zugehörige Root fehlt. Derselbe Import geht auch über die CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, danach `commit`.

Zwei Abgrenzungen dazu: Für Updates über einen TLS-inspizierenden Proxy führt die SMA einen separaten Truststore (`updateconfig > TRUSTED_CERTIFICATES > ADD`), die Custom-CA-Liste greift dort nicht. Und die Root-CA auf der SMA beseitigt keine Browserwarnungen: Die Clients brauchen die Root weiterhin über die eigene Zertifikatsverteilung, typischerweise per GPO, und die Appliance muss das Serverzertifikat samt Intermediate ausliefern.

## Installation mit certconfig

Auf der SMA per SSH anmelden und `certconfig` starten. Auf aktuellen AsyncOS-Versionen arbeitet der Dialog mit Zertifikatsprofilen:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Hinter `CERTIFICATE` liegen die Operationen `IMPORT` (PKCS#12-Datei, die zuvor auf die Appliance geladen wurde), `PASTE` (Zertifikat in die CLI einfügen), `NEW` (Self-Signed-Zertifikat samt CSR erzeugen), `EDIT`, `EXPORT`, `DELETE` und `PRINT` (zeigt die Zuweisung an die Dienste). Der übliche Weg über SSH ist `PASTE`: Der Dialog fragt einen Namen für das Profil ab, danach das Zertifikat, den privaten Schlüssel und optional das Intermediate-Zertifikat der CA, jeweils als PEM-Block, abgeschlossen mit einem einzelnen `.` auf eigener Zeile. Eine abschliessende Frage nach der FQDN-Prüfung des Common Name lässt sich mit dem Vorgabewert beantworten. Das Intermediate gehört mit ins Profil, sonst fehlt den Clients die Kette und je nach Browser bleibt die Warnung trotz gültigem Zertifikat bestehen.

Ältere AsyncOS-Versionen (der Stand der Cisco-Technote) zeigen stattdessen einen `SETUP`-Dialog. Er beginnt mit der Frage `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: Ein `y` weist dasselbe Paar allen vier Diensten zu, ein `n` durchläuft die Abfrage von Zertifikat, Schlüssel und Intermediate einmal je Dienst. Das Einfüge-Prinzip ist identisch.

Zwei Punkte entscheiden über Erfolg und Misserfolg: Die Session nicht mit Ctrl+C beenden, das verwirft alle Änderungen sofort. Und am Schluss `commit` ausführen, erst damit ist das Zertifikat aktiv. Bei zwei Appliances wiederholt sich der Ablauf auf beiden, die Zertifikatskonfiguration wird zwischen SMAs nicht synchronisiert.

## Kontrolle

Der schnellste Test läuft von aussen gegen die Quarantäneseite. Der Endbenutzerzugriff der Spam-Quarantäne liegt standardmässig auf HTTPS-Port 83, sofern beim Aktivieren nichts anderes konfiguriert wurde:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

Die Ausgabe muss den neuen Subject und das neue Ablaufdatum zeigen. Auf der Appliance listet `certconfig` mit der Operation `PRINT` die aktiven Zertifikate, und der Browser-Check gegen Admin-GUI und Quarantäneseite bestätigt, dass die Kette sauber aufgebaut ist.

## Quellen

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-Technote mit dem certconfig-Ablauf älterer AsyncOS-Versionen, der PEM-Vorgabe und den Wegen zur Zertifikatserzeugung über ESA oder OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 für Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Admin-Guide-Kapitel mit der Verwaltung der Certificate-Authority-Listen (System- und Custom-Liste) samt der Prüfungen beim CA-Import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco-Anleitung zur Spam-Quarantäne inklusive Endbenutzerzugriff über HTTPS auf Port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referenz für die Erzeugung von Schlüssel und CSR, inklusive `-addext` für die SAN-Liste.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referenz der Konvertierungsoptionen, unter anderem `-noenc` (früher `-nodes`) und `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Hintergrund zur Auslagerung der Altalgorithmen in den Legacy-Provider.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): quelloffenes Werkzeug für Import und Export von PKCS#12- und PEM-Strukturen.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows-Builds von Shining Light Productions, auf die das OpenSSL-Projekt verweist, inklusive publizierter Prüfsummenliste.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Beschreibung der automatischen Pfadumschreibung, die in der Git Bash das `-subj`-Argument verändert, samt `MSYS_NO_PATHCONV`.
