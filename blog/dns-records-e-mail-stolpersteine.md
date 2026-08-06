---
title: "Guide für DNS-Admins: MX, SPF, DKIM, DMARC und die üblichen Stolpersteine"
navTitle: "E-Mail-DNS-Records"
description: "Wer eine Zone betreut, bekommt Mail-Records meist fertig geliefert und soll sie nur noch publizieren. Was dabei regelmässig schiefgeht: die 255-Byte-Grenze bei DKIM, doppelte SPF-Records, das Lookup-Limit, MX auf einem CNAME, der automatisch angehängte Zonen-Suffix und Policies, die niemand mehr durchsetzt."
date: "2026-08-04"
kategorie: "SMTP und Mailflow"
timeToRead: "15 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "e-mail-verschluesselung"
produkte:
  - "uebergreifend"
protokolle:
  - "dns"
  - "smtp"
  - "tls"
  - "verschluesselung"
hauptthema: "smtp-mailflow"
related:
  - "smtp-verbindung-testen-linux"
  - "ghost-sender-exchange-online-nebeneingang"
slug: "dns-records-e-mail-stolpersteine"
translationId: "article-e4699ad7fcea2e20"
url: "https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine"
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
---
# Guide für DNS-Admins: MX, SPF, DKIM, DMARC und die üblichen Stolpersteine

Wer eine DNS-Zone betreut, bekommt Mail-Records selten selbst geschrieben geliefert. Das Mailteam, ein Provider oder ein Marketingdienstleister schickt eine Zeile mit dem Hinweis, sie müsse "nur noch publiziert werden". Genau daraus entstehen die meisten Fehler, denn Mail-Records sind die Record-Art, bei der ein Tippfehler zwei völlig verschiedene Folgen haben kann. Entweder die Zustellung bricht sofort ab und jemand meldet sich innerhalb von Minuten, oder sie läuft unverändert weiter und lediglich die Absenderprüfung fällt still aus. Der zweite Fall bleibt regelmässig monatelang unbemerkt, bis ein grosser Empfänger die Domain in Quarantäne schiebt.

Seit Google und Yahoo im Februar 2024 ihre Anforderungen an Massenversender verschärft haben und Microsoft im Mai 2025 nachgezogen ist, ist die Toleranz für halb konfigurierte Domains klein geworden. SPF, DKIM und ein DMARC-Record sind für Versender ab einem gewissen Volumen keine Kür mehr, sondern Zustellvoraussetzung.

Alle Beispiele in diesem Artikel verwenden `example.com` und generische Selektoren. Die gezeigten Werte sind gekürzt, damit sie lesbar bleiben.

## Regeln, die für jeden Mail-Record gelten

### Die 255-Byte-Grenze bei TXT-Records

Ein TXT-Record besteht laut RFC 1035 aus einer oder mehreren `character-strings`, und eine einzelne solche Zeichenkette fasst maximal 255 Byte. Der Record als Ganzes darf länger sein, er muss dann aber in mehrere Zeichenketten zerlegt werden. Auswertende Systeme hängen diese Teile ohne Trennzeichen wieder aneinander.

Praktisch relevant wird das genau an einer Stelle: bei DKIM-Schlüsseln mit 2048 Bit. Deren Base64-Wert ist rund 400 Zeichen lang und passt nicht in eine Zeichenkette.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

Die meisten DNS-Management-Systeme übernehmen diese Aufteilung selbst, wenn der Wert über das reguläre Eingabefeld eingetragen wird. Wer stattdessen von Hand Anführungszeichen setzt, muss die Grenze exakt einhalten. Ein umbrochener Wert mit einem Leerzeichen an der Nahtstelle ergibt einen Schlüssel, der syntaktisch existiert und kryptografisch nicht mehr passt.

Die Kontrolle danach ist wichtig, denn ein falsch zusammengesetzter Schlüssel sieht in der GUI vollkommen unauffällig aus:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

### Ein Record pro Zweck

SPF und DMARC sind so definiert, dass an einem Namen genau ein passender Record stehen darf. Bei SPF führen zwei `v=spf1`-Records zu einem `permerror`, und damit gilt die Prüfung als gescheitert, nicht etwa als bestanden. Bei DMARC ignorieren Empfänger die Domain vollständig, wenn mehrere Records mit `v=DMARC1` beginnen: statt einer strengen Policy greift dann gar keine.

Das ist der mit Abstand häufigste Fehler bei gewachsenen Zonen. Ein neuer Dienstleister wird angebunden, jemand legt "seinen" SPF-Record dazu, statt den bestehenden zu erweitern, und ab diesem Moment schlägt die Prüfung für sämtliche Absender fehl. Vor jedem neuen Record deshalb zwingend prüfen, was bereits da ist:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

Für DKIM gilt das Gegenteil: Dort ist pro Selektor ein Record vorgesehen, und mehrere Selektoren nebeneinander sind der Normalfall, weil jeder Versanddienst seinen eigenen Schlüssel mitbringt.

### Der Zonen-Suffix in Web-Oberflächen

In Infoblox, im Windows-DNS und bei praktisch allen Hosting-Oberflächen wird der Zonenname automatisch an den eingegebenen Namen angehängt. Wer im Feld "Name" den vollqualifizierten Namen einträgt, bekommt einen Record, der doppelt so lang ist wie beabsichtigt:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

In der Zonendatei ist das Gegenstück der fehlende abschliessende Punkt. `mail.example.com` ohne Punkt am Ende ist ein relativer Name und wird um den Zonennamen ergänzt, `mail.example.com.` mit Punkt ist absolut. Bei MX- und CNAME-Zielen entscheidet dieser eine Punkt darüber, ob die Domain erreichbar ist.

### Copy-Paste ist die häufigste Fehlerquelle

Mail-Record-Werte werden fast nie getippt, sondern aus einem PDF, einem Ticket, einer Excel-Zelle oder einem Chat kopiert. Dabei entstehen Schäden, die im Eingabefeld unsichtbar bleiben:

- Ein doppeltes `p=` am Anfang des DKIM-Schlüssels, weil der Präfix beim Zusammensetzen zweimal gesetzt wurde. Der Wert `v=DKIM1;k=rsa;p=p=MIIBIjAN...` ist ein realer Klassiker und ergibt einen unbrauchbaren Schlüssel.
- Typografische Anführungszeichen aus Word statt gerader.
- Geschützte Leerzeichen aus PDF-Layouts, die wie normale aussehen.
- Zeilenumbrüche mitten im Base64-Block, wenn der Wert im PDF über mehrere Zeilen lief.

Base64 kennt genau die Zeichen A bis Z, a bis z, 0 bis 9, `+`, `/` und `=` als Auffüllzeichen. Alles andere im `p=`-Teil ist ein Fehler. Ein kurzer Filter vor dem Eintragen erspart die spätere Fehlersuche:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

Kommt hier etwas anderes als `0` heraus, enthält der Schlüssel Fremdzeichen.

### TTL vor Umstellungen senken

Vor jedem geplanten Wechsel eines MX-, SPF- oder DKIM-Records gehört die TTL für einige Stunden auf einen niedrigen Wert, typischerweise 300 Sekunden. Andernfalls hängt der alte Wert je nach Zone noch einen Tag oder länger in fremden Resolvern, und ein Rollback dauert genauso lange. Nach der Umstellung und einer Beobachtungsphase wird die TTL wieder auf den regulären Wert gesetzt.

## MX

Der MX-Record legt fest, welcher Host Mails für die Domain annimmt. Es gibt zwei Regeln dazu, gegen die regelmässig verstossen wird.

**Das Ziel muss ein Hostname mit A- oder AAAA-Record sein.** Weder eine IP-Adresse noch ein CNAME sind zulässig. RFC 2181 stellt ausdrücklich fest, dass das Ziel eines MX-Records kein Alias sein darf. In der Praxis funktioniert es bei vielen Empfängern trotzdem, bei anderen nicht, was zu Fehlerbildern führt, die scheinbar nur einzelne Absender betreffen.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Die Zahl ist eine Präferenz, keine Gewichtung.** Der niedrigere Wert wird zuerst versucht. Ein zweiter MX mit hoher Zahl ist nur dann sinnvoll, wenn dieses System denselben Empfängerfilter kennt. Backup-MX-Einträge auf Systeme ohne Empfängerprüfung sind ein beliebtes Ziel für Spam, weil Angreifer gezielt den schwächsten Eintrag ansteuern.

Domains, die ausschliesslich senden oder gar nichts mit Mail zu tun haben, bekommen einen Null-MX nach RFC 7505. Er signalisiert, dass die Domain keine Mail annimmt, und sorgt für eine sofortige und eindeutige Ablehnung statt für Timeouts:

```text
example.com.  IN  MX  0 .
```

Der Null-MX ersetzt jedoch keinen SPF- und DMARC-Record. Nicht empfangen heisst nicht, dass niemand in Ihrem Namen sendet. Gerade parkierte Nebendomains werden für Spoofing genutzt, weil dort selten jemand hinschaut.

## A, AAAA, PTR und der HELO-Name

Der PTR-Record für die ausgehende IP-Adresse liegt nicht in Ihrer Zone, sondern in der `in-addr.arpa`-Zone des Providers, dem der Adressblock gehört. Er wird deshalb beim Provider bestellt und nicht selbst gesetzt. Viele grosse Empfänger verlangen, dass PTR und der zugehörige Vorwärts-Record zusammenpassen, also dass der Name aus dem PTR wieder auf dieselbe IP-Adresse auflöst.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

Der Name, den Ihr Mailserver im HELO oder EHLO nennt, sollte derselbe sein und ebenfalls auflösbar. Ein Gateway, das sich als `localhost.localdomain` oder mit einem internen Namen meldet, wird von grösseren Empfängern schlechter bewertet.

Beim Hinzufügen eines AAAA-Records ist Vorsicht geboten. Sobald der Mailserver über IPv6 erreichbar und sendend wird, gelten dieselben Anforderungen wie für IPv4, in Teilen sogar strengere. Google verlangt für sendende IPv6-Adressen einen gültigen PTR. Fehlt er, wird der Versand abgelehnt, während er über IPv4 tadellos lief. Ein AAAA-Record am Mailserver ist deshalb nie eine reine DNS-Änderung.

## SPF

SPF legt fest, welche Systeme im Namen der Domain senden dürfen. Der Record steht als TXT an der Domain selbst.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### Das Lookup-Limit

Die Auswertung eines SPF-Records darf höchstens zehn DNS-abfragende Mechanismen auslösen. Gezählt werden `include`, `a`, `mx`, `ptr`, `exists` und `redirect`, und zwar rekursiv: Jeder `include` bringt die Lookups des eingebundenen Records mit. Nicht gezählt werden `ip4`, `ip6` und `all`.

Wird das Limit überschritten, ist das Ergebnis ein `permerror`. Für DMARC bedeutet das ein nicht bestandenes SPF, unabhängig davon, ob der sendende Server eigentlich autorisiert wäre. Das Tückische daran: Der Fehler entsteht oft ohne eigenes Zutun, weil ein eingebundener Provider seinen Record erweitert. Der eigene Record hat sich nicht verändert, die Zustellung bricht trotzdem ein.

Zusätzlich sind höchstens zwei "void lookups" erlaubt, also Abfragen ohne Ergebnis. Ein `include` auf eine Domain, die es nicht mehr gibt, zählt hier hinein. Verweise auf abgelöste Dienstleister gehören deshalb entfernt und nicht vorsichtshalber stehen gelassen.

### Was in einen SPF-Record nicht gehört

- **`ptr`** ist zwar spezifiziert, gilt aber seit RFC 7208 als überholt und soll nicht verwendet werden. Auswertende Systeme dürfen ihn ignorieren.
- **`+all`** autorisiert jeden beliebigen Absender und ist damit schädlicher als gar kein SPF-Record.
- **`?all`** ist neutral und damit für DMARC praktisch wertlos.
- **Ein separater Record vom Typ SPF (Typ 99)** wird nicht mehr benötigt. Er ist seit RFC 7208 abgeschafft, SPF steht ausschliesslich in TXT.

Zwischen `~all` (softfail) und `-all` (hardfail) entscheidet, wie vollständig die Versandwege erfasst sind. Solange daran Zweifel bestehen, ist `~all` die richtige Wahl. Wer bereits DMARC durchsetzt und die Reports auswertet, kann auf `-all` gehen.

### Subdomains erben nichts

Ein SPF-Record an `example.com` gilt nicht für `newsletter.example.com`. Jede sendende Subdomain braucht einen eigenen Record. Für alle übrigen empfiehlt sich ein Wildcard-Eintrag, der klarstellt, dass von dort nichts kommt:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Vorsicht: Ein TXT-Wildcard beantwortet auch Anfragen für Namen wie `_dmarc.sub.example.com`, sofern dort kein expliziter Record steht. Das ist meist unproblematisch, kann aber die Fehlersuche verwirren, weil auf jede TXT-Abfrage eine Antwort kommt.

### SPF-Flattening

Werkzeuge, die alle `include`-Verweise auflösen und durch die dahinterliegenden IP-Adressen ersetzen, lösen das Lookup-Limit auf Kosten der Wartbarkeit. Ändert der Provider seine Adressen, bricht der Versand, und niemand merkt es, weil im eigenen Record scheinbar alles stimmt. Wer diesen Weg geht, braucht deshalb einen automatisierten Abgleich, der die Liste regelmässig gegen die Quelle prüft. Als einmalige Handarbeit fällt das Verfahren früher oder später aus.

## DKIM

DKIM signiert ausgehende Nachrichten. Der öffentliche Schlüssel steht unter `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Der Selektor ist frei wählbar und wird vom sendenden System vorgegeben. Ein sprechender Name mit Datum erleichtert die spätere Rotation deutlich mehr als `s1` und `s2`.

### Delegation per CNAME

Wo der Versanddienst es anbietet, ist die CNAME-Variante der direkten Eintragung vorzuziehen:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Der Provider kann seinen Schlüssel dann eigenständig rotieren, ohne dass jemand in Ihrer Zone tätig werden muss. Genau diese Rotation bleibt sonst regelmässig liegen, weil sie eine Koordination zwischen zwei Teams erfordert. Ein CNAME schliesst allerdings jeden weiteren Record an demselben Namen aus, das ist eine Grundregel des DNS und keine Eigenheit von DKIM.

### Rotation ohne Ausfall

Beim Schlüsselwechsel wird der neue Selektor zuerst publiziert, dann stellt der sendende Server auf ihn um, und erst danach wird der alte Record entfernt. Wer den alten Schlüssel sofort löscht, entwertet die Signaturen aller Nachrichten, die noch unterwegs oder in Warteschlangen sind, und macht nachträgliche Prüfungen unmöglich. Ein Vorlauf von einigen Tagen zwischen Umstellung und Löschung ist angemessen.

Ein Record mit leerem `p=` ist übrigens kein defekter Eintrag, sondern die spezifizierte Art, einen Schlüssel als zurückgezogen zu kennzeichnen.

### Schlüssellänge

1024 Bit gelten als überholt, 2048 Bit sind der Standard. Grössere RSA-Schlüssel bringen praktisch keinen Zusatznutzen und erhöhen nur die Wahrscheinlichkeit, dass ein Zwischensystem den Record nicht sauber verarbeitet.

## DMARC

DMARC verbindet SPF und DKIM mit einer Anweisung, was bei einer nicht bestandenen Prüfung geschehen soll, und liefert Berichte zurück. Der Record steht unter `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Seit Mai 2026 gilt mit RFC 9989 sowie den Report-Spezifikationen RFC 9990 und RFC 9991 die überarbeitete Fassung, die RFC 7489 ablöst. Für die Praxis sind drei Änderungen wichtig:

- **`pct` ist entfallen.** Die gestaffelte Einführung über einen Prozentsatz gibt es nicht mehr. An ihre Stelle tritt `t=y`, das die Domain als im Testbetrieb kennzeichnet: Reports laufen weiter, die Policy soll nicht durchgesetzt werden.
- **`np` ist neu.** Es setzt die Policy für nicht existierende Subdomains und schliesst damit eine Lücke, die Angreifer gerne nutzen, weil erfundene Subdomains bisher nur über `sp` erfasst waren. Ohne eigene Angabe folgt `np` dem Wert von `sp`.
- **Die Public Suffix List ist durch einen `Tree Walk` ersetzt.** Die Organisationsdomain wird nicht mehr aus einer extern gepflegten Liste bestimmt, sondern über gestufte DNS-Abfragen entlang des Namensbaums. Für grosse Namensräume mit vielen Ebenen ändert das die Auswertung spürbar.

### Alignment ist der eigentliche Kern

DMARC besteht nicht, weil SPF oder DKIM technisch bestanden haben, sondern nur, wenn mindestens eines von beiden zusätzlich zur sichtbaren Absenderdomain im `From`-Header passt. SPF wird dabei gegen die Envelope-Absenderdomain geprüft, und die weicht bei Weiterleitungen, Newsletter-Diensten und Ticketsystemen regelmässig ab. Genau deshalb überstehen Nachrichten mit gültigem SPF gelegentlich die DMARC-Prüfung nicht.

Mit `adkim=r` und `aspf=r` (relaxed, der Standard) genügt Übereinstimmung auf Ebene der Organisationsdomain. `s` verlangt exakte Gleichheit inklusive Subdomain und scheitert in der Praxis fast immer an einem der Versandwege.

### Externe Report-Adressen brauchen eine Freigabe

Sollen Berichte an eine Adresse ausserhalb der eigenen Domain gehen, etwa an einen DMARC-Auswertungsdienst, muss die empfangende Domain das autorisieren. Ohne diesen Record schicken viele Empfänger schlicht nichts, und die Auswertung bleibt leer, während im eigenen Record alles korrekt aussieht:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Diesen Eintrag legt der Betreiber der Zielzone an, nicht Sie. Bei kommerziellen Diensten geschieht das automatisch, bei einer selbst betriebenen Sammelmailbox in einer anderen eigenen Domain aber nicht.

### Typische Syntaxfehler

Tag-Namen und Policy-Werte sind kleinzuschreiben, `p=Reject` ist ungültig. Zwischen den Tags steht ein Semikolon, ein fehlendes Trennzeichen macht den Rest der Zeile unwirksam. Und `p` muss als erstes Tag nach `v` stehen. Ein Record, der nur aus `v=DMARC1; rua=...` besteht, enthält keine Policy und ist unvollständig.

### Der Rollout

`p=none` ist ein Messzustand, kein Ziel. Es ändert am Umgang der Empfänger mit Ihren Mails nichts und dient allein dazu, über die Reports alle legitimen Versandwege zu finden. Wer nach der Einführung nicht innerhalb weniger Monate über `quarantine` auf `reject` geht, hat den Aufwand betrieben, ohne den Schutz zu bekommen. Die organisatorische Seite dieses Wegs, inklusive Entscheidungsvorlage, ist ein eigenes Thema und im DMARC-Blueprint beschrieben.

## MTA-STS und TLS-RPT

SMTP verschlüsselt opportunistisch: Bietet die Gegenstelle STARTTLS an, wird verschlüsselt, sonst nicht. Ein Angreifer in der Position, den Verkehr zu manipulieren, kann die STARTTLS-Ankündigung entfernen und die Verbindung damit im Klartext halten. MTA-STS schliesst diese Lücke für empfangende Domains.

MTA-STS besteht aus zwei Teilen, und nur einer davon liegt im DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

Die eigentliche Policy liegt als Datei unter `https://mta-sts.example.com/.well-known/mta-sts.txt` und muss über ein gültiges Zertifikat ausgeliefert werden:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Die Stolpersteine liegen fast alle ausserhalb der Zone:

- **Die `id` muss sich bei jeder Policy-Änderung ändern.** Sie ist der einzige Hinweis für sendende Systeme, dass eine neue Policy abzuholen ist. Wer die Datei anpasst und die `id` stehen lässt, arbeitet bis zum Ablauf von `max_age` gegen zwischengespeicherte Kopien.
- **Die MX-Liste in der Policy und die MX-Records müssen übereinstimmen.** Ein neuer MX, der in der Policy fehlt, wird von Sendern mit `mode: enforce` abgelehnt. Bei Migrationen gehört die Policy deshalb vor dem MX-Wechsel angepasst.
- **`mode: testing` zuerst.** In diesem Modus werden Verstösse nur gemeldet, nicht erzwungen. Der Wechsel auf `enforce` erfolgt, wenn die Berichte sauber sind.
- **Ein CAA-Record kann die Zertifikatsausstellung für den Policy-Host blockieren**, wenn dort eine andere Zertifizierungsstelle eingetragen ist als die verwendete.

TLS-RPT liefert die dazugehörigen Berichte und ist ein einzelner Record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT ist auch ohne MTA-STS sinnvoll, weil es fehlgeschlagene Transportverschlüsselung überhaupt erst sichtbar macht.

## DANE

DANE erreicht dasselbe Ziel wie MTA-STS, verankert das Vertrauen aber im DNS statt in der Web-PKI. Es setzt eine durchgehend mit DNSSEC signierte Zone voraus, und ohne DNSSEC ist ein TLSA-Record wirkungslos.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Entscheidend im Betrieb: Bei jedem Zertifikatswechsel muss der TLSA-Record vorher passen. Das übliche Verfahren veröffentlicht den neuen Hash parallel zum alten, wechselt anschliessend das Zertifikat und entfernt danach den alten Eintrag. Wer diese Reihenfolge umdreht, macht den Mailserver für alle DANE-prüfenden Sender unerreichbar, und das sind unter anderem die grossen deutschsprachigen Provider. In der Schweiz ist DANE deutlich seltener anzutreffen als MTA-STS, was meist an der fehlenden DNSSEC-Signierung der Zone liegt.

## BIMI

BIMI zeigt das Markenlogo im Posteingang an und ist der einzige hier behandelte Mechanismus, der noch kein RFC ist, sondern weiterhin als Internet-Draft geführt wird.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Die Voraussetzungen sind hoch: eine durchgesetzte DMARC-Policy mit `quarantine` oder `reject`, ein Logo im Format SVG Tiny Portable/Secure und für die meisten Anbieter ein kostenpflichtiges Verified Mark Certificate. BIMI ist damit kein Sicherheitsmechanismus, sondern ein Sichtbarkeitsthema, und es gehört ans Ende der Reihenfolge, nicht an den Anfang.

## Weitere Records im Umfeld

**Autodiscover und SRV:** Exchange-Umgebungen nutzen `autodiscover.example.com` als CNAME oder einen SRV-Record `_autodiscover._tcp.example.com`. Beides betrifft die Client-Konfiguration und nicht den Mailfluss, wird aber beim Migrieren gern übersehen und sorgt dann für Profile, die sich nicht mehr einrichten lassen.

**CAA:** Hat mit Mail direkt nichts zu tun, entscheidet aber darüber, welche Zertifizierungsstelle für `mta-sts.example.com` oder den Mailserver-Namen ein Zertifikat ausstellen darf.

**Split-Horizon-Zonen:** Wo eine interne DNS-Zone denselben Namen trägt wie die öffentliche, existieren die Mail-Records intern oft nicht. Interne Systeme, die eine SPF- oder DKIM-Prüfung durchführen, kommen dann zu anderen Ergebnissen als die Aussenwelt. Bei jeder Änderung an Mail-Records gehört deshalb die Frage dazu, ob die interne Zone nachgeführt werden muss.

## Einige kurze Tests

Alle Abfragen bewusst gegen einen öffentlichen Resolver stellen, damit nicht der interne Cache oder eine Split-Horizon-Zone antwortet:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

Gegen den autoritativen Server, um Caching vollständig zu umgehen:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

Unter Windows ohne `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

Für die vollständige Auswertung inklusive SPF-Lookup-Zählung, DKIM-Selektorsuche und Alignment-Prüfung gibt es auf dieser Seite den [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), der eine Domain in einem Durchgang gegen alle hier beschriebenen Records prüft.

Der aussagekräftigste Test bleibt jedoch eine echte Nachricht. Schicken Sie eine Mail an ein Postfach bei einem grossen Anbieter und sehen Sie sich die `Authentication-Results`-Zeile im Kopf an. Sie zeigt in einer Zeile, was SPF, DKIM und DMARC tatsächlich ergeben haben, und ersetzt jede Theorie über die Zonendatei.

## Reihenfolge bei einer Migration

Wechselt der Mailanbieter, hat sich diese Abfolge bewährt:

1. TTL aller betroffenen Records auf 300 Sekunden senken, mindestens einen Tag vorher.
2. DKIM-Selektoren des neuen Anbieters publizieren, solange die alten noch stehen.
3. SPF um den neuen Anbieter erweitern, ohne den alten zu entfernen, und dabei das Lookup-Limit nachrechnen.
4. Bei MTA-STS die Policy auf die neuen MX-Namen anpassen und die `id` erhöhen, bevor die MX-Records wechseln.
5. MX umstellen und die Zustellung beobachten.
6. Erst nach einigen Tagen ohne Beanstandung die alten SPF-Includes und DKIM-Selektoren entfernen.
7. TTL zurücksetzen.

Das häufigste Problem in diesem Ablauf ist Schritt 6 zu früh: Alte Einträge werden zusammen mit der Umstellung gelöscht, und alles, was noch über den bisherigen Weg läuft, scheitert an der Absenderprüfung.

## Fazit

Mail-Records unterscheiden sich von allen anderen DNS-Einträgen darin, dass ein Fehler nicht zwangsläufig auffällt. Ein falscher A-Record führt binnen Minuten zu einem Ticket, ein doppelter SPF-Record oder ein DKIM-Schlüssel mit einem Zeichen zu viel dagegen zu einer Zustellrate, die über Wochen langsam sinkt.

Drei Regeln verhindern die meisten dieser Fälle. Erstens: vor jedem neuen Record prüfen, was bereits vorhanden ist, statt einen zweiten danebenzustellen. Zweitens: nach jeder Änderung gegen einen öffentlichen Resolver kontrollieren und den Wert zeichenweise gegen die Vorlage vergleichen, nicht nur optisch. Drittens: bei Umstellungen immer erst das Neue publizieren, dann umschalten, dann das Alte entfernen. Wer diese Reihenfolge einhält, hat bei Mail-Records jederzeit einen Rückweg.

## Quellen

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Definiert unter anderem die 255-Byte-Grenze einer einzelnen `character-string` in TXT-Records.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Hält in Abschnitt 10.3 fest, dass das Ziel eines MX-Records kein Alias sein darf.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Lookup-Limit von zehn Mechanismen, Void-Lookup-Grenze, Abschaffung des RR-Typs SPF und Abraten vom `ptr`-Mechanismus.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Aufbau des Schlüsselrecords unter `_domainkey`, Bedeutung des Selektors und des leeren `p=`.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Aktuelle DMARC-Spezifikation vom Mai 2026, löst RFC 7489 ab; Wegfall von `pct`, neues Tag `np`, Tree Walk statt Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format und Zustellung der Aggregatberichte, inklusive Autorisierung externer Empfängerdomains.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Kennzeichnung von Domains, die keine Mail annehmen.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): DNS-Record, Policy-Datei, Bedeutung der `id` und der Modi `testing` und `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Aufbau des `_smtp._tls`-Records und der Berichte.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): TLSA-Records für SMTP und die Voraussetzung einer DNSSEC-signierten Zone.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Aktueller Stand der BIMI-Spezifikation, weiterhin kein RFC.

12.  [Google: Richtlinien für E-Mail-Absender](https://support.google.com/a/answer/81126): Anforderungen an Absender, unter anderem PTR-Pflicht für sendende IPv6-Adressen und die seit Februar 2024 geltenden Vorgaben für Massenversender.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Anforderungen für Versender ab 5000 Nachrichten pro Tag, gültig seit Mai 2025.
