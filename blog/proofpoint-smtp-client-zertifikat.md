---
title: "Proofpoint Protection Server: Zertifikat für «SMTP Client» nach einem Patch prüfen"
navTitle: "PPS SMTP Client"
description: "Steht im Proofpoint Protection Server unter System > Certificates > Services die Zuordnung «SMTP Client» auf «-none-», liefert Sendmail ohne TLS-Client-Zertifikat aus. Gegenstellen mit Zertifikatsprüfung lehnen dann ab, etwa Exchange Online mit 5.7.51 oder ein Smarthost mit «Relay access denied». So prüfen und beheben Sie das."
date: "2026-11-05"
kategorie: "SMTP und Mailflow"
timeToRead: "7 min to read"
themen:
  - "smtp-mailflow"
produkte:
  - "proofpoint"
  - "exchange-online"
protokolle:
  - "tls"
  - "smtp"
  - "troubleshooting"
slug: "proofpoint-smtp-client-zertifikat"
url: "https://rafaelpfister.ch/blog/proofpoint-smtp-client-zertifikat"
translationId: "article-6f99807d289cc394"
---

Ich habe heute einen Ausfall im ausgehenden Mailflow eines Proofpoint Protection Server (PPS) 8.22.4 untersucht. Kurzfazit: Nach einem automatisch eingespielten Patch stand unter **System > Certificates > Services** die Zuordnung **SMTP Client** auf `-none-`. Sendmail hat danach ausgehende Verbindungen ohne TLS-Client-Zertifikat aufgebaut. Alle Gegenstellen, die Proofpoint über dieses Zertifikat erkennen, haben die Mails abgewiesen. Gewöhnliche Empfänger mit opportunistischem TLS haben weiter angenommen, deshalb fiel der Fehler erst auf, als einzelne Partner nicht mehr erreichbar waren.

Wenn Sie PPS mit automatischen Patches betreiben und Partner haben, die ein Client-Zertifikat verlangen, prüfen Sie diese Einstellung nach jedem Patch.

## Symptom

Betroffen sind nur Verbindungen, bei denen die Gegenstelle das Client-Zertifikat von Proofpoint prüft. Zwei typische Fälle:

- **Exchange Online mit Partner-Connector**, der per `RestrictDomainsToCertificate` auf ein Zertifikat beschränkt ist. Die Ablehnung lautet `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost, der Relay nur mit Client-Zertifikat erlaubt.** Die Antwort lautet `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Alle anderen Ziele erhalten ihre Mails weiterhin. Die Testmatrix zeigt deshalb ein uneinheitliches Bild: Mails nach extern gehen durch, Mails an einzelne Partner oder über einen bestimmten Smarthost scheitern.

Im Maillog des Agents steht bei jeder ausgehenden Verbindung eine der beiden folgenden Meldungen. Fehlt die Datei ganz:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Ist die Datei vorhanden, aber leer:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Direkt danach baut Sendmail die Verbindung trotzdem mit TLS auf (`verify=OK`), jedoch ohne Client-Zertifikat. Für die Gegenstelle sieht das aus wie ein unbekannter Absender.

## Ursache

PPS ordnet jedem Dienst pro Server ein Zertifikat zu. Unter **System > Certificates > Services** stehen pro Server unter anderem `SMTP Server`, `SMTP Client`, `Administration Server` und `End User Web Application / Encryption`. Aus der Zuordnung bei `SMTP Client` erzeugt PPS die Datei `/etc/pki/tls/certs/sendmail-client.pem`, die Sendmail als Client-Zertifikat verwendet. Steht dort `-none-`, schreibt PPS die Datei leer oder gar nicht.

In der untersuchten Umgebung stand die Zuordnung nach einem automatischen Patch auf allen Servern auf `-none-`. Unter **System > Config History** war der Patch als Eintrag `configuration changes for patch-…` mit dem Ersteller `system` sichtbar, die Dateien unter `/etc/pki/tls/certs/` trugen denselben Zeitstempel. Ob Proofpoint die Zuordnung mit diesem Patch neu eingeführt hat oder ob der Patch eine bestehende Zuordnung zurückgesetzt hat, ist in den öffentlich zugänglichen Release Notes nicht beschrieben. Ich ergänze den Artikel, sobald Proofpoint dazu Stellung nimmt.

Ein Rollback unter **Config History** auf den Snapshot des Patches hat nicht geholfen. PPS hat die Datei danach neu geschrieben, jedoch mit 0 Bytes.

## Prüfen

In der Admin-GUI auf dem Master unter **System > Certificates > Services** oben bei **Server** nacheinander jeden Agent auswählen, der nach aussen ausliefert, und die Zeile `SMTP Client` ansehen.

Auf dem Agent selbst zeigen zwei Befehle den Zustand:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-l` | Langes Format mit Rechten, Besitzer, Grösse und Datum |
| `--time-style=full-iso` | Zeitstempel mit Datum, Uhrzeit und Zeitzone, um ihn mit dem Patch-Zeitpunkt abzugleichen |
| `grep -E` | Erweiterte reguläre Ausdrücke, damit beide Suchbegriffe mit `\|` kombiniert werden können |
| `tail -5` | Nur die letzten fünf Treffer anzeigen |

</details>

Fehlt `sendmail-client.pem` oder hat die Datei 0 Bytes, während `sendmail.pem` einige tausend Bytes gross ist, ist dem Dienst `SMTP Client` kein Zertifikat zugeordnet.

## Beheben

Unter **System > Certificates > Services** für jeden betroffenen Agent bei `SMTP Client` ein gültiges Zertifikat auswählen und mit **Save Changes** speichern. In der Regel ist das dasselbe öffentliche Zertifikat, das bei `SMTP Server` steht.

Die Auswahlliste zeigt alle importierten Zertifikate, auch abgelaufene mit demselben Namen. Unterscheiden lassen sie sich nur über die Seriennummer in Klammern. Vergleichen Sie die Seriennummer vor dem Speichern mit der Liste unter **System > Certificates > Certificates**, dort stehen Ausstellungs- und Ablaufdatum.

Nach dem Speichern verteilt PPS die Konfiguration an die Agents. Die Kontrolle auf dem Agent:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `ls -l` | Zeigt Grösse und Rechte der Datei; nach der Korrektur ist sie nicht mehr leer |
| `grep -E 'STARTTLS=client\|stat='` | Filtert die TLS-Meldungen ausgehender Verbindungen und das Zustellergebnis |
| `tail -20` | Nur die letzten zwanzig Zeilen anzeigen |

</details>

Die Meldungen `SSL_CTX_… failed` und `unsafe` dürfen nach dem Speichern nicht mehr erscheinen, die Zustellungen an die Partner müssen mit `stat=Sent` enden. Mails, die vorher mit einem 4xx-Code zurückgestellt wurden, stellt Sendmail aus der Queue selbst zu. Mails, die mit einem 5xx-Code abgelehnt wurden, müssen die Absender neu senden.

Eine Kopie von `sendmail.pem` nach `sendmail-client.pem` von Hand überbrückt den Ausfall ebenfalls. PPS überschreibt die Datei jedoch beim nächsten Konfigurations-Push wieder, solange die Zuordnung in der GUI fehlt.

## Nach jedem Patch kontrollieren

Bis geklärt ist, warum der Patch die Zuordnung verändert hat, empfehle ich nach jedem Patch eine Kontrolle der Seite **Services**. Einfacher ist eine Überwachung, die bei der Meldung `sendmail-client.pem` im Maillog der Agents alarmiert. Unter **Licenses and Updates > Settings** lässt sich zudem festlegen, wann Patches eingespielt werden. Ein Zeitfenster, in dem jemand den Mailflow prüfen kann, verkürzt einen solchen Ausfall deutlich.

## Client Authentication verschwindet aus öffentlichen Zertifikaten

Unabhängig vom Patch betrifft eine zweite Änderung genau diese Verbindungen. Öffentliche TLS-Zertifikate enthalten in der Erweiterung `Extended Key Usage` bisher meist beide Verwendungszwecke: `TLS Web Server Authentication` und `TLS Web Client Authentication`. Mailgateways nutzen deshalb ihr Serverzertifikat auch als Client-Zertifikat.

Das Chrome Root Program verlangt, dass öffentliche Zertifikate, die ab dem 15.03.2027 ausgestellt werden, nur noch `serverAuth` enthalten. DigiCert stellt öffentliche TLS-Zertifikate seit dem 01.10.2025 standardmässig ohne Client Authentication aus; auf Antrag ist der Zweck noch bis zum 01.03.2027 möglich. Andere CAs haben vergleichbare Zeitpläne. Ob ein Zertifikat den Client-Zweck enthält, zeigt OpenSSL:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `x509` | Unterbefehl für die Anzeige und Bearbeitung von X.509-Zertifikaten |
| `-in zertifikat.pem` | Zertifikatsdatei im PEM-Format |
| `-noout` | Das Zertifikat selbst nicht erneut ausgeben |
| `-subject` | Inhaber (CN und weitere Felder) anzeigen |
| `-enddate` | Ablaufdatum anzeigen |
| `-ext extendedKeyUsage` | Nur die Erweiterung mit den Verwendungszwecken anzeigen |

</details>

Unter Windows zeigt ein Doppelklick auf die Datei dasselbe: Registerkarte **Details**, Feld **Erweiterte Schlüsselverwendung**.

Was das für ausgehende Verbindungen bedeutet, hängt von der Gegenstelle ab:

| Gegenstelle prüft | Folge bei einem Zertifikat ohne Client Authentication |
|---|---|
| Namen im Zertifikat (z. B. Exchange Online über `TlsSenderCertificateName`) | Ob Exchange Online den Verwendungszweck prüft, ist nicht dokumentiert; vor dem Zertifikatswechsel testen |
| Verwendungszweck des Client-Zertifikats | Ablehnung, obwohl Name und CA stimmen |
| Fingerabdruck des Client-Zertifikats | Ablehnung bei jedem Zertifikatswechsel, bis der neue Fingerabdruck hinterlegt ist |

Dazu kommt die kürzere Laufzeit öffentlicher Zertifikate. Nach dem CA/Browser-Forum-Beschluss SC-081v3 sinkt sie schrittweise von 398 Tagen auf 47 Tage bis März 2029, die erste Stufe mit 200 Tagen gilt seit März 2026. Jeder Zertifikatswechsel ist ein Zeitpunkt, an dem eine Gegenstelle das neue Zertifikat nicht kennt.

Für Verbindungen, bei denen Partner Proofpoint über ein Client-Zertifikat erkennen, empfehle ich deshalb:

1.  **Bestehende Zertifikate prüfen:** Enthalten die heutigen Gateway-Zertifikate den Client-Zweck noch, und wann laufen sie ab?

2.  **Ein eigenes Client-Zertifikat planen**, das den Client-Zweck dauerhaft enthält, etwa aus einer privaten PKI oder aus einem Angebot der CA für Server-zu-Server-Verbindungen. Die Partner müssen es kennen.

3.  **Partner fragen, was sie prüfen:** Name, Verwendungszweck, Aussteller oder Fingerabdruck. Danach richtet sich, ob ein Zertifikatswechsel für sie eine Änderung bedeutet.

4.  **Zertifikatswechsel als Change führen**, mit einer Liste aller Gegenstellen, die das Zertifikat kennen müssen, und einem Test pro Gegenstelle.

## Quellen

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): Parameter `RestrictDomainsToCertificate` und `TlsSenderCertificateName` für Partner-Connectoren in Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): Abschnitt 1.3.2 zu dedizierten TLS-Server-Hierarchien; Zertifikate ab dem 15.03.2027 nur noch mit `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): Standard ohne Client Authentication seit 01.10.2025, auf Antrag bis 01.03.2027, Alternativen für Server-zu-Server-Verbindungen.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): Übersicht der verschobenen Termine für Client Authentication und der Laufzeitstufen bei DigiCert und Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Beschluss zur schrittweisen Verkürzung der Laufzeit öffentlicher TLS-Zertifikate bis März 2029.
