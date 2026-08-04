---
title: "Zertifikat auf der Cisco SMA erneuern: von der PFX-Datei bis certconfig"
navTitle: "SMA-Zertifikat"
description: "Zertifikate lassen sich auf der Cisco SMA nur über die CLI und nur im PEM-Format einspielen, eigene Schlüssel erzeugt die Appliance nicht. Der Artikel zeigt drei Wege zum neuen Schlüsselpaar, geht den empfohlenen OpenSSL-Weg im Detail durch und erklärt, warum OpenSSL 3 ältere PFX-Dateien mit dem Fehler RC2-40-CBC ablehnt und was dagegen hilft."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "8 Min. Lesezeit"
themen:
  - "cisco-esa-sma"
  - "smtp-mailflow"
hauptthema: "cisco-esa-sma"
slug: "cisco-sma-zertifikat-erneuern"
translationId: "article-69d93a1e5e081848"
url: "https://rafaelpfister.ch/blog/cisco-sma-zertifikat-erneuern"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. Installation über certconfig in der CLI, 5. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
---
# Zertifikat auf der Cisco SMA erneuern: von der PFX-Datei bis certconfig

Die Cisco SMA (Security Management Appliance, inzwischen unter dem Namen Cisco Secure Email and Web Manager geführt) übernimmt in vielen Mailumgebungen die zentrale Spam-Quarantäne und das Reporting für die Secure-Email-Gateways. Ihr HTTPS-Zertifikat deckt die Admin-GUI und die Quarantäneseite ab, auf der Endbenutzer ihre zurückgehaltenen Mails sichten und freigeben. Läuft es ab, bricht kein Mailfluss zusammen. Sichtbar wird der Ablauf trotzdem sofort: Jeder Aufruf der Quarantäneseite endet mit einer Zertifikatswarnung im Browser, und ausgerechnet die Benutzer, denen Awareness-Schulungen beibringen, bei solchen Warnungen nicht weiterzuklicken, sollen sie dann ignorieren.

Bei einer Erneuerung in einem Kundenprojekt war die Appliance selbst der einfache Teil. Gestolpert bin ich über OpenSSL 3, das die PFX-Datei der internen CA mit einem kryptischen Fehler zu `RC2-40-CBC` quittierte. Dazu weiter unten mehr.

## Was die SMA anders macht als die ESA

Auf der ESA lässt sich der komplette Zertifikatslebenszyklus über die GUI abwickeln (`Network > Certificates`). Die SMA kann das nicht: Zertifikate werden ausschliesslich über die CLI eingespielt, mit dem Befehl `certconfig` in einer SSH-Session. Die GUI der SMA zeigt Zertifikate nur an.

Dazu kommen zwei weitere Eigenheiten, die Cisco in der Technote zur SMA dokumentiert:

- Die SMA akzeptiert beim Import nur das PEM-Format. Eine PFX-Datei (PKCS#12) muss vor der Installation konvertiert werden.
- Die SMA erzeugt selbst weder Schlüssel noch CSRs. Das Schlüsselpaar muss ausserhalb der Appliance entstehen; die drei gangbaren Wege dazu weiter unten.

Ein einzelnes Zertifikat kann wahlweise alle Dienste bedienen (eingehendes und ausgehendes TLS, HTTPS-Verwaltungszugriff, LDAPS) oder pro Dienst separat hinterlegt werden. In den meisten Umgebungen ist ein Zertifikat für alles die pragmatische Wahl.

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

Weil die SMA keine Schlüssel erzeugt, muss das Schlüsselpaar auf einem der folgenden Wege entstehen:

1. Schlüssel und CSR mit OpenSSL innerhalb der eigenen Umgebung erzeugen. Der private Schlüssel entsteht dort, wo er gebraucht wird, und verlässt die Umgebung nie. Der empfohlene Weg, Details im nächsten Abschnitt.
2. Die CA erzeugt das Schlüsselpaar und liefert eine PFX-Datei. Funktioniert, hat aber zwei Haken: Der Schlüssel reist durch fremde Hände (das Passwort gehört deshalb in einen separaten Kanal und nicht in dieselbe Mail wie die Datei), und je nach CA-Werkzeug kommt eine RC2-verschlüsselte PFX zurück, die OpenSSL 3 nur mit Zusatzaufwand öffnet; dazu unten mehr.
3. Der Umweg über eine ESA, dokumentiert in der Cisco-Technote: Dort unter `Network > Certificates` ein Zertifikat mit dem CN der SMA anlegen, den CSR herunterladen und von der CA signieren lassen, das signierte Zertifikat wieder auf die ESA hochladen und das Ganze als PFX exportieren. Auch hier steht am Ende die Konvertierung nach PEM an.

## Schlüssel und CSR mit OpenSSL erzeugen

Für den ersten Weg braucht es OpenSSL auf einem System innerhalb der Umgebung, etwa einem Admin-Server. Die Light-Edition der Windows-Builds von Shining Light Productions genügt, der Installer ist rund 6 MB gross und lässt sich gegen die von slproweb publizierte Prüfsummenliste verifizieren.

Ein einzelner Aufruf erzeugt Schlüssel und CSR mit der kompletten SAN-Liste:

```bash
openssl req -new -newkey rsa:2048 -noenc -keyout spam-quarantine.example.ch.key -out spam-quarantine.example.ch.csr -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
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

## Installation mit certconfig

Auf der SMA per SSH anmelden und `certconfig` starten. Der Dialog fragt zuerst, ob ein Zertifikat für alle Dienste gelten soll:

```text
sma01.example.ch> certconfig

Choose the operation you want to perform:
- SETUP - Configure security certificates and keys.
[]> setup

Do you want to use one certificate/key for receiving, delivery,
HTTPS management access, and LDAPS? [Y]> y

paste cert in PEM format (end with '.'):
```

Jetzt den Block von `-----BEGIN CERTIFICATE-----` bis `-----END CERTIFICATE-----` aus der PEM-Datei einfügen und mit einem einzelnen `.` auf eigener Zeile abschliessen. Danach fragt der Dialog den privaten Schlüssel ab, zum Schluss optional das Intermediate-Zertifikat der CA. Das Intermediate gehört mit hinein, sonst fehlt den Clients die Kette und je nach Browser bleibt die Warnung trotz gültigem Zertifikat bestehen.

Zwei Punkte entscheiden über Erfolg und Misserfolg: Die Session nicht mit Ctrl+C beenden, das verwirft alle Änderungen sofort. Und am Schluss `commit` ausführen, erst damit ist das Zertifikat aktiv. Bei zwei Appliances wiederholt sich der Ablauf auf beiden, die Zertifikatskonfiguration wird zwischen SMAs nicht synchronisiert.

## Kontrolle

Der schnellste Test läuft von aussen gegen die Quarantäneseite. Der Endbenutzerzugriff der Spam-Quarantäne liegt standardmässig auf HTTPS-Port 83, sofern beim Aktivieren nichts anderes konfiguriert wurde:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 -servername spam-quarantine.example.ch </dev/null 2>/dev/null | openssl x509 -noout -subject -enddate
```

Die Ausgabe muss den neuen Subject und das neue Ablaufdatum zeigen. Auf der Appliance listet `certconfig` mit der Operation `PRINT` die aktiven Zertifikate, und der Browser-Check gegen Admin-GUI und Quarantäneseite bestätigt, dass die Kette sauber aufgebaut ist.

## Quellen

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-Technote mit dem certconfig-Ablauf, der PEM-Vorgabe und dem Hinweis, dass die SMA selbst keine Zertifikate erzeugt.

2.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco-Anleitung zur Spam-Quarantäne inklusive Endbenutzerzugriff über HTTPS auf Port 83.

3.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referenz für die Erzeugung von Schlüssel und CSR, inklusive `-addext` für die SAN-Liste.

4.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referenz der Konvertierungsoptionen, unter anderem `-noenc` (früher `-nodes`) und `-legacy`.

5.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Hintergrund zur Auslagerung der Altalgorithmen in den Legacy-Provider.

6.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): quelloffenes Werkzeug für Import und Export von PKCS#12- und PEM-Strukturen.

7.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows-Builds von Shining Light Productions, auf die das OpenSSL-Projekt verweist, inklusive publizierter Prüfsummenliste.
