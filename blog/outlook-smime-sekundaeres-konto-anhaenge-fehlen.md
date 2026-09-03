---
title: "Outlook: S/MIME-Signatur im sekundären Konto nicht überprüfbar, Anhänge fehlen"
navTitle: "S/MIME im Zweitkonto"
description: "Das neue Outlook meldet beim freigegebenen Postfach, die S/MIME-Signatur könne im sekundären Konto nicht überprüft werden, und zeigt keine Anhänge. Der Artikel erklärt den Unterschied zwischen Clear Signing und Opaque Signing, warum die Anhänge bei opak signierten Mails verschwinden, weshalb das neue Outlook S/MIME nur im Primärkonto verarbeitet und welche Auswege es gibt, inklusive Auspacken der smime.p7m mit PowerShell oder OpenSSL."
date: "2026-09-03"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "8 Min. Lesezeit"
themen:
  - "microsoft-365-exchange"
  - "e-mail-verschluesselung"
produkte:
  - "exchange-online"
  - "windows-client"
protokolle:
  - "verschluesselung"
  - "troubleshooting"
related:
  - "e-mail-header-analysieren-ohne-upload"
slug: "outlook-smime-sekundaeres-konto-anhaenge-fehlen"
translationId: "article-f1e9d4ab5be67349"
url: "https://rafaelpfister.ch/blog/outlook-smime-sekundaeres-konto-anhaenge-fehlen"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
---
# Outlook: S/MIME-Signatur im sekundären Konto nicht überprüfbar, Anhänge fehlen

Im neuen Outlook für Windows erscheint beim Öffnen einer digital signierten Mail in einem freigegebenen Postfach ein roter Balken: "Das S/MIME-Zeichen kann beim Anzeigen im sekundären Konto nicht überprüft werden." Die Mail selbst wird angezeigt, die Anhänge aber nicht, obwohl der Absender welche mitgeschickt hat. Kolleginnen und Kollegen, die dasselbe Postfach als Hauptkonto nutzen, sehen die Anhänge problemlos.

Dahinter stecken zwei Dinge, die sich gegenseitig verstärken: das neue Outlook verarbeitet S/MIME nur im primären Konto, und der Absender hat die Mail opak signiert. Bei dieser Signaturform steckt der komplette Inhalt inklusive Anhänge in einem einzigen kryptografischen Container. Kann der Client den Container nicht öffnen, bleiben die Anhänge unsichtbar. Beides lässt sich einzeln beheben.

## Was die Meldung bedeutet

"Sekundäres Konto" heisst im neuen Outlook jedes Postfach, das nicht das Konto ist, mit dem Sie sich angemeldet haben. Das trifft auf freigegebene Postfächer (Shared Mailboxes) zu, die über Vollzugriff und Automapping eingeblendet werden, ebenso auf Postfächer, die Sie über "Freigegebenes Postfach hinzufügen" eingebunden haben, und auf Delegationen. Die S/MIME-Verarbeitung ist im neuen Outlook fest an das Primärkonto gebunden. Trifft eine signierte Nachricht in einem anderen Konto ein, prüft der Client die Signatur nicht und zeigt stattdessen die Meldung.

Das ist keine Aussage über die Gültigkeit der Signatur und kein Zertifikatsproblem beim Absender. Dieselbe Mail lässt sich im Primärkonto oder im klassischen Outlook prüfen und öffnen.

## Clear Signing und Opaque Signing

Der S/MIME-Standard (RFC 8551) kennt zwei Formate für signierte Nachrichten. Beide liefern dieselbe Signatur, verpacken die Nachricht aber unterschiedlich.

| | Clear Signing | Opaque Signing |
|---|---|---|
| MIME-Typ | `multipart/signed` mit `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` mit `smime-type=signed-data` |
| Aufbau | Zwei Teile: der lesbare Nachrichtentext samt Anhängen und daneben die abgetrennte Signatur | Ein Teil: Nachrichtentext, Anhänge und Signatur zusammen in einem CMS-SignedData-Container, Base64-kodiert |
| Anhang, den ein Client ohne S/MIME sieht | `smime.p7s` (nur die Signatur, wenige KB) | `smime.p7m` (die gesamte Nachricht) |
| Lesbar ohne S/MIME-Unterstützung | Ja, Text und Anhänge werden normal angezeigt | Nein, der Client sieht nur den Container |
| Empfindlichkeit auf dem Transportweg | Signatur wird ungültig, wenn ein Mailserver oder Gateway Zeilenumbrüche, Kodierung oder Leerzeichen ändert | Der Container schützt den Inhalt vor solchen Änderungen |
| RFC-8551-Abschnitt | 3.5.3 | 3.5.2 |

In der Nachrichtenquelle erkennen Sie die beiden Formate an der Kopfzeile `Content-Type`. Eine clear-signed Mail beginnt so:

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

Eine opaque-signed Mail so:

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

Der Unterschied erklärt das Verhalten im neuen Outlook vollständig. Bei einer clear-signed Mail zeigt der Client Text und Anhänge auch dann an, wenn er die Signatur nicht prüft; es fehlt nur der Signaturstatus. Bei einer opaque-signed Mail muss der Client den Container erst über die S/MIME-Verarbeitung auspacken, um an Text und Anhänge zu kommen. Verweigert er das, weil die Nachricht in einem sekundären Konto liegt, bleibt der Container zu. Dass der Text trotzdem lesbar ist, liegt an Exchange Online: Der Dienst rendert den Textteil serverseitig, die Anhänge aus dem Container aber nicht.

Beide Formate verschlüsseln nichts. Auch die opake Variante ist nur Base64-kodiert und für jeden lesbar, der die Nachricht in die Hand bekommt. Microsoft weist in der Exchange-Online-Dokumentation ausdrücklich darauf hin.

## Welches Format der Absender wählt

Im klassischen Outlook steuert die Option "Signierte Nachrichten als Klartext senden" (Datei > Optionen > Trust Center > E-Mail-Sicherheit) das Format. Sie ist standardmässig aktiviert, Outlook signiert also clear-signed. Wer die Option abschaltet, sendet opak. Das neue Outlook und Outlook im Web bieten diese Einstellung nicht an.

Mail-Gateways, die zentral signieren, haben eine eigene Einstellung für das Signaturformat. Einige Produkte signieren aus Robustheitsgründen standardmässig opak, weil die Signatur dann auch nach einem Umbruch durch nachgelagerte Systeme gültig bleibt. Erhalten Sie von einem bestimmten Absender regelmässig Mails mit fehlenden Anhängen, lohnt sich ein Blick in dessen Gateway-Konfiguration.

## Warum das neue Outlook S/MIME nur im Primärkonto verarbeitet

Microsoft dokumentiert die Einschränkung, nennt aber keinen technischen Grund. Der Grund ergibt sich aus der Architektur des Clients.

Das neue Outlook ist im Kern der Web-Client von Outlook im Web in einer nativen Hülle. Im Browser darf JavaScript nicht auf den Windows-Zertifikatspeicher zugreifen. Deshalb brauchte Outlook im Web jahrelang eine separate S/MIME-Browsererweiterung. Das neue Outlook ersetzt diese Erweiterung durch eine eingebaute Brücke zwischen der Web-Oberfläche und der Windows-Kryptografie. Diese Brücke wird beim Anmelden des Primärkontos initialisiert und erhält dessen Mailbox-Sitzung, dessen Zertifikate und dessen S/MIME-Einstellungen aus Einstellungen > E-Mail > S/MIME.

Freigegebene Postfächer und Zweitkonten laufen über andere Datenpfade. Zweitkonten haben eigene Sitzungen, freigegebene Postfächer werden über die Delegation des Primärkontos geladen. Für diese Pfade war die Brücke bisher nicht angebunden. Das gilt auch für das reine Prüfen einer Signatur, obwohl dafür kein privater Schlüssel nötig wäre: das Parsen und Auspacken der PKCS#7-Struktur läuft über dieselbe Komponente.

Im klassischen Outlook tritt das Problem nicht auf, weil dort die S/MIME-Verarbeitung im MAPI-Stack pro Nachricht passiert, unabhängig davon, aus welchem Store die Nachricht stammt.

Microsoft ergänzt die fehlende Anbindung: Roadmap-Eintrag 565861 erweitert S/MIME im neuen Outlook auf freigegebene und delegierte Postfächer, die am Primärkonto hängen. Die allgemeine Verfügbarkeit ist für Juli 2026 angekündigt, der Rollout erfolgt gestaffelt. Sehen Sie die Meldung weiterhin, ist die Änderung in Ihrem Tenant oder Ihrer Client-Version noch nicht angekommen. Separat hinzugefügte Zweitkonten mit eigener Anmeldung deckt der Eintrag nicht ab.

## Auswege

Welcher Weg passt, hängt davon ab, wie das Postfach eingebunden ist und ob Sie die Signatur prüfen müssen oder nur an die Anhänge wollen.

| Weg | Voraussetzung | Ergebnis |
|---|---|---|
| Mail im Primärkonto öffnen | Sie haben das Postfach selbst als Hauptkonto oder die Mail wurde an Sie weitergeleitet | Signaturprüfung und Anhänge |
| Postfach als eigenständiges Konto im neuen Outlook hinzufügen (Einstellungen > Konten > Konto hinzufügen) | Das Postfach hat eigene Anmeldedaten; bei freigegebenen Postfächern ohne Passwort nicht möglich | Signaturprüfung und Anhänge, sobald Sie in dieses Konto wechseln |
| Klassisches Outlook | Noch installiert oder über den Schalter "Neues Outlook" zurückschaltbar; Postfach dort als eigenes Konto einbinden (Datei > Kontoeinstellungen > Neu) | Signaturprüfung und Anhänge in jedem Store |
| Outlook im Web | Postfach direkt öffnen (`outlook.office.com/mail/<adresse>`), S/MIME-Erweiterung für Edge oder Chrome installiert | Signaturprüfung und Anhänge |
| Absender bittet um Clear Signing | Absender nutzt klassisches Outlook oder ein Gateway mit wählbarem Format | Anhänge sichtbar, Signaturstatus im Zweitkonto weiterhin nicht |
| Container manuell auspacken | Die Mail als `.eml` speichern oder `smime.p7m` sichern | Anhänge ohne Signaturprüfung |

## Den Container manuell auspacken

Für den Einzelfall reicht es, den Container ausserhalb von Outlook zu öffnen. Die Signatur wird dabei rechnerisch geprüft, die Vertrauenskette des Zertifikats aber nicht. Speichern Sie die Nachricht als `.eml` oder sichern Sie den Anhang `smime.p7m` in einen Ordner.

Unter Windows genügt PowerShell. Das .NET-Framework bringt die Klasse `SignedCms` mit, die den PKCS#7-Container liest:

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Anweisung | Wirkung |
|---|---|
| `Add-Type -AssemblyName System.Security` | Lädt die Assembly mit den PKCS-Klassen (in Windows PowerShell 5.1 nötig, in PowerShell 7 bereits geladen) |
| `[IO.File]::ReadAllBytes(...)` | Liest den binären DER-Container; die aus Outlook gespeicherte `smime.p7m` ist bereits dekodiert |
| `$cms.Decode($bytes)` | Parst die CMS-SignedData-Struktur |
| `$cms.CheckSignature($true)` | Prüft nur die Signatur über dem Inhalt (`$true` = verifySignatureOnly); mit `$false` würde zusätzlich die Zertifikatskette geprüft. Bei ungültiger Signatur bricht der Befehl mit einer Ausnahme ab |
| `$cms.ContentInfo.Content` | Der signierte Inhalt: eine vollständige MIME-Nachricht mit Text und Anhängen |
| `[IO.File]::WriteAllBytes(...)` | Schreibt diese MIME-Nachricht als `.eml`, die Sie in Outlook oder Thunderbird öffnen |

</details>

Unter Linux, macOS oder mit Git für Windows steht OpenSSL zur Verfügung. Liegt die ganze Mail als `.eml` vor, übernimmt OpenSSL auch das Base64-Dekodieren:

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `cms` | CMS-Werkzeug von OpenSSL; `smime` funktioniert gleichwertig, `cms` ist die neuere Schnittstelle |
| `-verify` | Prüft die Signatur und gibt den signierten Inhalt aus |
| `-noverify` | Überspringt die Prüfung der Zertifikatskette; die Signatur selbst wird trotzdem geprüft |
| `-in nachricht.eml` | Die komplette Mail im S/MIME-Format (Base64 mit MIME-Kopfzeilen); für eine gesicherte `smime.p7m` zusätzlich `-inform DER` |
| `-out inhalt.eml` | Der ausgepackte Inhalt als MIME-Nachricht |

</details>

Die Datei `inhalt.eml` enthält den ursprünglichen Nachrichtentext und alle Anhänge als normale MIME-Teile. Ein Doppelklick öffnet sie in Outlook, wo Sie die Anhänge wie gewohnt speichern.

## Quellen

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): Der Fall aus der Praxis mit derselben Meldung im freigegebenen Postfach; die Antwort bestätigt das Verhalten als bekannt und nennt keine Abhilfe im neuen Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Abschnitte 3.5.2 (application/pkcs7-mime mit SignedData) und 3.5.3 (multipart/signed) mit den Aussagen zur Lesbarkeit ohne S/MIME und zur Robustheit auf dem Transportweg.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): Die Option "Signierte Nachrichten als Klartext senden" im klassischen Outlook, standardmässig aktiviert; im neuen Outlook nicht vorhanden.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): S/MIME-Einstellungen im neuen Outlook unter Einstellungen > E-Mail > S/MIME; Zertifikate müssen manuell oder per Richtlinie installiert werden.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Hinweis, dass opak signierte Nachrichten nur Base64-kodiert und nicht vertraulich sind.

6.  [Microsoft 365 Roadmap, Eintrag 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME für freigegebene und delegierte Postfächer im neuen Outlook für Windows, angekündigt für Juli 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Welche Kontotypen das neue Outlook unterstützt und wie freigegebene Postfächer eingebunden werden.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature und ContentInfo für das Auspacken des Containers mit PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Optionen `-verify`, `-noverify`, `-inform` und `-out`.
