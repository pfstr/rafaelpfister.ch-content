---
title: "Totemomail-Zertifikat erneuern: 4096-Bit-Schlüssel, PKCS#12-Import und Neustart pro Node"
navTitle: "Zertifikat erneuern"
description: "Der Antragsdialog von Totemomail erzeugt nur 2048-Bit-Schlüssel ohne alternative Namen. Viele interne CAs signieren aber nur noch 4096 Bit. Schlüssel und Antrag entstehen deshalb mit openssl, danach folgen Bestellung bei der PKI-Stelle, PKCS#12-Import, Anschlussbindung und Neustart pro Node."
date: "2026-10-06"
kategorie: "Totemomail"
timeToRead: "12 Min. Lesezeit"
themen:
  - "totemomail"
  - "e-mail-verschluesselung"
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "totemomail-zertifikat-erneuern"
translationId: "article-1e59c4ee01e408a3"
url: "https://rafaelpfister.ch/blog/totemomail-zertifikat-erneuern"
---

# Totemomail-Zertifikat erneuern: 4096-Bit-Schlüssel, PKCS#12-Import und Neustart pro Node

Das Serverzertifikat eines Totemomail-Verbunds (heute Kiteworks Email Protection Gateway) zu erneuern, sieht nach einer Routineaufgabe aus: Antrag in der Verwaltungsoberfläche erzeugen, signieren lassen, Antwort importieren. In der Praxis scheitert dieser Weg häufig schon am ersten Schritt. Der Dialog „New PKCS#10" erzeugt fest einen 2048-Bit-Schlüssel und bietet kein Feld für alternative Namen (Subject Alternative Names). Viele interne Zertifizierungsstellen signieren inzwischen nur noch 4096 Bit, und ein Zertifikat ohne alternative Namen erkennen aktuelle TLS-Gegenstellen gar nicht als gültig für einen Namen an.

Der folgende Ablauf hat sich bei einem Tausch im Herbst 2026 in Vorproduktion und Produktion bewährt: Schlüssel und Antrag mit openssl auf dem Gateway, Bestellung bei der PKI-Stelle, Import als PKCS#12, Bindung an die Anschlüsse und Neustart Node für Node. Beim Tausch fielen mehrere Eigenheiten auf, darunter ein Fehler in der Oberfläche, der beim Lösen des alten Zertifikats auftritt.

## Zwei Zertifikate mit unterschiedlichem Zweck

Ein Totemomail-Verbund hinter Exchange Online braucht in der Regel zwei Arten von Zertifikaten.

| Art | Zweck | Aussteller | Laufzeit |
|---|---|---|---|
| Intern | Weboberfläche, Verwaltung, interne SMTP-Verbindungen. Enthält die internen Namen der Nodes. | interne PKI, etwa Active Directory Certificate Services | frei wählbar, typisch 12 bis 13 Monate |
| Öffentlich | Strecke zwischen Exchange Online und dem Gateway, wenn Exchange Online das Zertifikat prüfen soll | öffentliche Zertifizierungsstelle | seit 15.03.2026 höchstens 200 Tage, ab 15.03.2027 höchstens 100 Tage |

Ein einziges Zertifikat für beide Zwecke funktioniert nicht. Öffentliche Zertifizierungsstellen stellen keine internen Namen und keine Kurznamen ohne Domäne aus, und jedes öffentliche Zertifikat erscheint in den Certificate-Transparency-Protokollen. Umgekehrt vertraut Exchange Online keiner internen Kette. Wie Exchange Online die Strecke zu einem Verschlüsselungsgateway behandelt, beschreibt der Beitrag [Mail-Schlaufe mit Verschlüsselungsgateway hinter EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online).

Die folgenden Schritte gelten für das interne Zertifikat. Für das öffentliche ist der Ablauf bis zum Import identisch, die Unterschiede stehen im letzten Abschnitt.

## Der Ablauf im Überblick

1. Schlüssel und Antrag mit openssl auf einem Node erzeugen.
2. Antrag bei der PKI-Stelle bestellen.
3. Geliefertes Zertifikat prüfen.
4. Zertifikat, Schlüssel und Zwischenstelle zu einer PKCS#12-Datei zusammenfassen und auf den Rechner mit dem Browser holen.
5. In der Totemomail-Oberfläche importieren und an die Anschlüsse binden.
6. Jeden Node einzeln neu starten und prüfen.
7. Mailfluss testen, danach aufräumen.

Planen Sie mindestens drei Wochen vor dem Ablauf. Solange das alte Zertifikat gültig ist, gibt es einen Rückweg. Ist in Ihrer Umgebung eine Vorproduktion vorhanden, tauschen Sie dort zuerst und verwenden den Durchlauf als Referenz.

## Schritt 1: Schlüssel und Antrag mit openssl

Arbeiten Sie auf einem Node des Verbunds mit Ihrem persönlichen Benutzer, nicht mit dem Dienstkonto `totemo`. So können Sie die PKCS#12-Datei später direkt per `scp` holen. Für alle Schritte sind keine Root-Rechte nötig.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

Die Konfiguration enthält Inhaber, Verwendungszwecke und alle alternativen Namen. Die Namen im Beispiel sind Platzhalter: drei Nodes und die Dienstnamen, unter denen der Verbund intern angesprochen wird.

```bash
cat > intern.cnf <<'EOF'
[ req ]
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
C  = CH
O  = Beispiel AG
CN = SecureMail

[ ext ]
basicConstraints = critical, CA:FALSE
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt

[ alt ]
DNS.1 = gw01.intern.example.ch
DNS.2 = gw02.intern.example.ch
DNS.3 = gw03.intern.example.ch
DNS.4 = securemail.intern.example.ch
EOF
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Eintrag | Wirkung |
|---|---|
| `default_md = sha256` | Hashverfahren für die Signatur des Antrags |
| `prompt = no` | Werte aus der Datei übernehmen statt interaktiv abzufragen |
| `req_extensions = ext` | Erweiterungen aus dem Abschnitt `[ ext ]` in den Antrag schreiben |
| `basicConstraints = critical, CA:FALSE` | Endzertifikat, keine Zertifizierungsstelle |
| `keyUsage` | Schlüssel für Signatur und Schlüsselaustausch, wie für TLS-Server üblich |
| `extendedKeyUsage = serverAuth, clientAuth` | Server- und Client-Authentifizierung. Das Gateway ist auf manchen Strecken Server, auf anderen Client. |
| `subjectAltName = @alt` | alternative Namen aus dem Abschnitt `[ alt ]` |

</details>

Nehmen Sie nur vollständige Namen auf. Kurznamen ohne Domäne lehnen viele Registrierungsstellen ab, und Sie sollten sich nicht darauf verlassen, dass sie im Zertifikat landen.

Den Schlüssel erzeugen Sie ohne Passphrase und schützen ihn über die Dateirechte. Er liegt nur bis zum Import auf dem Node und wird danach gelöscht. Eine Passphrase auf einem Schlüssel, der wenige Tage existiert, schützt wenig und erzeugt ein neues Risiko: Geht sie verloren, ist der Schlüssel unbrauchbar und das Zertifikat muss neu ausgestellt werden.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `genpkey -algorithm RSA` | neuen privaten Schlüssel vom Typ RSA erzeugen |
| `-pkeyopt rsa_keygen_bits:4096` | Schlüssellänge 4096 Bit |
| `-out intern.key` | Datei für den Schlüssel, ohne Passphrase |
| `req -new` | neuen Zertifikatsantrag (CSR) erzeugen |
| `-key intern.key` | vorhandenen Schlüssel verwenden |
| `-config intern.cnf` | Inhaber, Erweiterungen und Namen aus der Konfiguration |
| `-out intern.csr` | Datei für den Antrag |

</details>

Prüfen Sie den Antrag, bevor er die Maschine verlässt:

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Erwartet sind `verify OK`, der richtige Inhaber, `4096 bit` und die Anzahl Ihrer Namen. Notieren Sie zusätzlich den Fingerabdruck des öffentlichen Schlüssels. Mit ihm ordnen Sie das gelieferte Zertifikat später eindeutig diesem Schlüssel zu, auch wenn zwei Anträge denselben Inhaber tragen:

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Schritt 2: Bestellung bei der PKI-Stelle

Die Stelle, die das Zertifikat ausstellt, braucht pro Antrag diese Angaben:

- den CSR als Text
- den SHA-256-Fingerabdruck des öffentlichen Schlüssels
- die Art: intern oder öffentlich, und bei mehreren Umgebungen welche
- die Namensliste zum Kopieren, ein Name pro Zeile
- Schlüssellänge 4096, erweiterte Verwendung Server Authentication und Client Authentication
- den Termin, bis wann das Zertifikat vorliegen muss, mindestens zwei Wochen vor dem Ablauf

Bei einem Tausch mit mehreren Umgebungen und Zertifikatsarten lohnt sich eine Übersicht am Anfang der Mail: welche Zertifikate es gibt, welche intern und welche öffentlich ausgestellt werden, in welcher Reihenfolge sie gebraucht werden. Für die Ausstellung ist kein Change am System nötig. Die Zertifikate aller Umgebungen können daher gleichzeitig ausgestellt werden, auch wenn das Einspielen nacheinander erfolgt.

Beim Tausch im Herbst 2026 fielen bei der Registrierungsstelle (RA) mehrere Punkte auf, die in vielen Umgebungen ähnlich auftreten dürften:

- **Die RA setzt den Inhaber selbst.** Im Zertifikat standen nur Land, Organisation und Common Name, auch wenn der Antrag Organisationseinheit, Ort und Kanton enthielt.
- **Keine Kurznamen.** Namen ohne Domäne fehlten im ausgestellten Zertifikat.
- **Namen von Hand übernommen.** Die RA übernahm die alternativen Namen nicht aus dem CSR, sie wurden in der Maske eingetippt. Ein Name kam abgeschnitten an. Die gelieferte Namensliste ist deshalb immer zu prüfen.
- **Falsches Profil.** Eine RA, die sowohl die interne CA als auch eine öffentliche CA bedient, stellte Anträge für öffentliche Zertifikate zuerst über das interne Profil aus. Im Aussteller stand die interne CA. Solche Zertifikate sind für die Strecke zu Exchange Online wertlos.
- **Nur ein Name bei öffentlichen Einzelzertifikaten.** Ein Single-Domain-Produkt bricht mit einem Antrag mit zwei Namen ab, etwa mit „Only one Subject Alternative Name is allowed". Ein Name genügt aber, siehe den Abschnitt zum öffentlichen Zertifikat.

Zertifikate, die durch Fehlausstellungen entstehen, sollte die PKI-Stelle anschliessend sperren. Sie tragen einen gültigen Schlüssel und laufen sonst ein Jahr lang mit.

## Schritt 3: Lieferung prüfen

Legen Sie das gelieferte Zertifikat im selben Verzeichnis ab wie den Schlüssel, etwa mit `cat > intern.crt`, Inhalt einfügen, `Strg+D`. Dann prüfen:

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Kontrollieren Sie den Aussteller, die vollständige Namensliste ohne Tippfehler und Doppelungen, und die beiden Verwendungszwecke. Ob Zertifikat und Schlüssel zusammengehören, zeigt der Vergleich der Fingerabdrücke. Beide Werte müssen gleich sein:

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

Bei einem öffentlichen Zertifikat kommen zwei Prüfungen dazu. Die Richtlinien-OID `2.23.140.1.2.2` steht für ein organisationsvalidiertes Zertifikat nach den Regeln des CA/Browser Forum, und das Zertifikat muss eingebettete Certificate-Transparency-Nachweise tragen. Wenige Minuten nach der Ausstellung erscheint es unter seinem Namen auf crt.sh. Fehlt beides, ist es kein öffentliches Zertifikat, unabhängig von der Beschriftung der Lieferung.

## Schritt 4: PKCS#12-Datei bauen

Totemomail importiert Zertifikat und Schlüssel zusammen als PKCS#12. Holen Sie dafür das Zertifikat der ausstellenden Zwischenstelle. Die Adresse steht im Zertifikat unter `Authority Information Access`:

```bash
openssl x509 -in intern.crt -noout -ext authorityInfoAccess
curl -sS -o issuing.crt http://pki.example.ch/crt/Issuing-CA.crt
file issuing.crt
```

Meldet `file` nicht `PEM certificate`, ist die Datei DER-kodiert und muss umgewandelt werden:

```bash
openssl x509 -inform DER -in issuing.crt -out issuing.pem
mv issuing.pem issuing.crt
```

Der Subject Key Identifier der Zwischenstelle muss mit dem Authority Key Identifier des Zertifikats übereinstimmen. Danach die Datei bauen. Der Befehl fragt nach einem Exportkennwort, das Sie beim Import brauchen:

```bash
openssl pkcs12 -export \
  -inkey intern.key \
  -in intern.crt \
  -certfile issuing.crt \
  -name "SecureMail" \
  -out intern.p12
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-export` | PKCS#12-Datei erzeugen |
| `-inkey intern.key` | privater Schlüssel |
| `-in intern.crt` | ausgestelltes Zertifikat |
| `-certfile issuing.crt` | weitere Zertifikate der Kette, hier die Zwischenstelle |
| `-name "SecureMail"` | Anzeigename des Eintrags in der Datei |
| `-out intern.p12` | Zieldatei, geschützt mit dem Exportkennwort |

</details>

Kontrolle, es müssen zwei Einträge erscheinen, das Zertifikat und die Zwischenstelle:

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

Die Totemomail-Oberfläche läuft im Browser, typischerweise auf einem Jumphost. Die Datei muss dorthin. Unter Windows ist der OpenSSH-Client mit `scp` vorhanden:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

Haben Sie den Schlüssel doch als `totemo` erzeugt, liegt er unter `/opt/totemomail`, und Ihr Benutzer kann ihn nicht lesen. Kopieren Sie die PKCS#12-Datei dann kurz nach `/tmp`, holen sie dort ab und löschen sie sofort wieder. Der Umweg über die Zwischenablage mit Base64 funktioniert zwar, ist bei einer Zeile von rund 10'000 Zeichen aber fehleranfällig.

## Schritt 5: In Totemomail importieren und binden

Unter `Key Management`:

1. **`Issuer Certificates`**: die Zwischenstelle `issuing.crt` importieren.
2. **`Own Server Certificates`**, Schaltfläche **`Import`**: Der Dialog bietet zwei Wege. Links „Import certificate" ist für Zertifikat mit Schlüssel, also die PKCS#12-Datei. Rechts „Import a PKCS#10 certificate reply" ist nur für Antworten auf Anträge, die Totemomail selbst erzeugt hat. Wählen Sie links und geben Sie im zweiten Schritt das Exportkennwort ein.
3. Das neue Zertifikat öffnen (Stift-Symbol). Unter `Connector` sollten alle Anschlüsse angehakt sein, in der beschriebenen Installation `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT` und `8444=AdminAPI`. `Host` steht auf `*`.
4. Das alte Zertifikat öffnen und alle Anschlüsse bis auf einen abwählen.

Zu Punkt 4 gehört der Fehler, der beim Tausch auftrat: Wählt man beim alten Zertifikat **alle** Anschlüsse ab, meldet die Oberfläche „Could not edit selected server certificate". Im Protokoll steht dazu:

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

Totemomail verträgt kein Serverzertifikat ohne Anschluss. Lassen Sie deshalb einen Anschluss stehen, etwa `8444=AdminAPI`, und löschen Sie das alte Zertifikat nach der Beobachtungszeit. Löschen statt Abwählen funktioniert, das Zertifikat landet unter `Deleted Certificates`.

Für das Verständnis der Bindung sind drei Beobachtungen hilfreich:

- **Die Anschlussliste gilt nur für die Webdienste.** Port 25 erscheint dort nicht. Ist kein Zertifikat vom Typ SMTPS vorhanden, verwendet SMTP das HTTPS-Zertifikat. Deshalb zeigt Port 25 nach dem Tausch das neue Zertifikat, obwohl in der Liste bei SMTPS kein Haken steht.
- **Eine ausdrückliche Zuordnung hat Vorrang vor dem Sternchen.** Der Anschluss, der beim alten Zertifikat stehen bleibt, zeigt weiterhin das alte, auch wenn das neue mit `*` für alle eingetragen ist.
- **Die Liste ist für den Verbund gemeinsam.** Sie sieht auf jedem Node gleich aus, unabhängig davon, ob der Node die Änderung schon übernommen hat. Was ein Node tatsächlich vorzeigt, zeigt nur eine Abfrage auf den Ports.

## Schritt 6: Neustart Node für Node

Totemomail liest die Zertifikate beim Start ein. Nach dem Import zeigen alle Nodes weiterhin das alte Zertifikat, bis sie neu gestartet sind. Starten Sie jeden Node einzeln neu, nie alle gleichzeitig, damit hinter dem Loadbalancer immer Nodes aktiv bleiben. Beginnen Sie mit den Nodes, über die Sie nicht angemeldet sind, und nehmen Sie den Node mit der geöffneten Oberfläche zuletzt.

Auf dem Node als `totemo`:

```bash
totemomail stop
totemomail start
```

Weitere Hinweise zum kontrollierten Stoppen stehen im Beitrag [Wichtigste Controls für Totemomail-Admins](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

Die Webdienste sind nach etwa einer Minute wieder erreichbar, der SMTP-Dienst auf Port 25 erst nach einigen Minuten. Eine leere Antwort auf Port 25 kurz nach dem Start deutet also noch nicht auf einen Fehler hin. Erst wenn ein Node auf allen Ports umgestellt ist, kommt der nächste an die Reihe.

Die Prüfung über alle Nodes und Ports:

```bash
for h in gw01 gw02 gw03; do
  for p in 25 443 8443; do
    if [ "$p" = 25 ]; then s="-starttls smtp"; else s=""; fi
    c=$(echo | openssl s_client -connect "$h:$p" $s 2>/dev/null |
        openssl x509 -noout -serial -enddate 2>/dev/null | tr '\n' ' ')
    printf "%-6s %-5s %s\n" "$h" "$p" "$c"
  done
done
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `s_client -connect host:port` | TLS-Verbindung zum Dienst aufbauen |
| `-starttls smtp` | bei Port 25 zuerst den SMTP-Dialog führen und dann mit STARTTLS auf TLS umschalten |
| `x509 -noout` | Zertifikat lesen, ohne es auszugeben |
| `-serial -enddate` | Seriennummer und Ablaufdatum anzeigen |

</details>

Bleibt Port 25 nach fünf Minuten leer, zeigt `ss -lnt | grep ':25 '`, ob der Dienst lauscht, und das Protokoll unter `/opt/totemomail` den Grund.

Ob Totemomail die Zwischenstelle mitsendet, zeigt die Option `-showcerts`:

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

Beim beschriebenen Tausch erschien nur das Endzertifikat, obwohl die Zwischenstelle in der PKCS#12-Datei und unter `Issuer Certificates` lag. Für interne Strecken, auf denen niemand das Zertifikat prüft, hat das keine Folgen. Für die Prüfung durch Exchange Online muss die Kette dagegen vollständig sein.

## Schritt 7: Tests, Rückweg, Aufräumen

Testen Sie nach dem letzten Neustart den Mailfluss in beide Richtungen: eine Nachricht von extern durch die Schleife und eine nach extern über das Gateway. In der Nachrichtenverfolgung von Exchange Online müssen beide Schenkel als zugestellt erscheinen, und in Richtung Gateway darf nichts in der Warteschlange hängen.

Der **Rückweg** besteht darin, beim alten Zertifikat die Anschlüsse wieder anzuhaken, beim neuen bis auf einen abzuwählen und die Nodes erneut einzeln neu zu starten. Das geht nur, solange das alte Zertifikat gültig ist.

Nach erfolgreichen Tests:

- die PKCS#12-Datei auf dem Jumphost löschen
- den Arbeitsordner auf dem Node löschen, der Schlüssel liegt jetzt im Schlüsselspeicher von Totemomail
- nach einigen Tagen das alte Zertifikat löschen und die Nodes noch einmal einzeln neu starten, damit auch der letzte Anschluss umstellt
- Fehlausstellungen bei der PKI-Stelle sperren lassen
- das neue Ablaufdatum in die Wiedervorlage eintragen

Kündigen Sie den Tausch den Administratoren an, wenn das neue Zertifikat keine Kurznamen mehr enthält. Wer die Oberfläche bisher mit `https://gw01:8443` aufgerufen hat, sieht danach eine Zertifikatswarnung. Das gilt auch für Überwachungen und Skripte, die Kurznamen verwenden.

## Eigenheiten im Überblick

| Beobachtung | Folge | Umgang |
|---|---|---|
| „New PKCS#10" erzeugt 2048 Bit ohne alternative Namen | Antrag für 4096-Bit-CAs unbrauchbar | Schlüssel und Antrag mit openssl erzeugen |
| Änderungen wirken erst nach dem Neustart | Nodes zeigen das alte Zertifikat weiter | jeden Node einzeln neu starten |
| Port 25 kommt einige Minuten nach den Webdiensten | leere Antwort kurz nach dem Start | warten, erst dann den nächsten Node |
| Zertifikat ohne Anschluss löst eine NullPointerException aus | altes Zertifikat lässt sich nicht vollständig lösen | einen Anschluss stehen lassen, später löschen |
| Ausdrückliche Zuordnung hat Vorrang vor `*` | ein Anschluss zeigt weiter das alte Zertifikat | altes Zertifikat nach der Beobachtung löschen |
| Nur das Endzertifikat wird gesendet | Gegenstellen, die prüfen, können die Kette nicht bilden | vor einer Prüfung durch Exchange Online klären |
| RA übernimmt Namen von Hand | Tippfehler und fehlende Namen möglich | Namensliste der Lieferung prüfen |

## Das öffentliche Zertifikat für die Strecke zu Exchange Online

Damit Exchange Online die Strecke zum Gateway zusätzlich zur Verschlüsselung auch prüft, braucht das Gateway auf Port 25 ein öffentliches Zertifikat. Erst dann lässt sich der ausgehende Connector auf `TlsSettings DomainValidation` mit einem `TlsDomain` umstellen und der eingehende Connector über `TlsSenderCertificateName` an das Zertifikat binden.

Ein einziger Name genügt dafür. Exchange Online vergleicht in beiden Richtungen nur den Namen im Zertifikat mit dem eingetragenen Wert, die Adresse dahinter prüft es nicht. Ein Zertifikat für den Namen, unter dem das Gateway nach aussen bekannt ist, deckt Hin- und Rückweg ab. Bei der Bestellung sollten Sie ausdrücklich nach Client Authentication fragen: Mehrere öffentliche Zertifizierungsstellen haben diese Verwendung 2026 aus TLS-Zertifikaten entfernt. Für den Rückweg, auf dem das Gateway sich als Client ausweist, ist sie nötig.

Für den Einsatz in Totemomail zeichnet sich nach den Beobachtungen oben dieser Weg ab: das öffentliche Zertifikat als Typ SMTPS importieren, damit Port 25 es vorzeigt und die Webdienste das interne behalten. Vorher sind zwei Punkte zu klären. Die Kette muss vollständig gesendet werden, und Port 25 zeigt das öffentliche Zertifikat danach allen Einsendern, auch internen wie einem vorgelagerten Gateway. Erwartet ein Einsender ein bestimmtes Zertifikat, bricht diese Strecke.

## Quellen

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Fahrplan für die Laufzeit öffentlicher TLS-Zertifikate, 200 Tage ab März 2026, 100 Tage ab März 2027, 47 Tage ab März 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector): Parameter `TlsSettings` und `TlsDomain` für die Prüfung des Zertifikats auf der Gegenseite.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector): Parameter `TlsSenderCertificateName` für die Zuordnung über das Zertifikat des Einsenders.

4.  [OpenSSL-Dokumentation: openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Aufbau der Konfigurationsdatei und Optionen für Zertifikatsanträge.

5.  [OpenSSL-Dokumentation: openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Erzeugen und Prüfen von PKCS#12-Dateien.

6.  [crt.sh](https://crt.sh): Suche in den Certificate-Transparency-Protokollen, um die Ausstellung eines öffentlichen Zertifikats zu prüfen.
