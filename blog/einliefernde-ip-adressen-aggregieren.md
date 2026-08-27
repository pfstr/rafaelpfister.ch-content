---
title: "Wer liefert eigentlich in Ihren Tenant ein? Einliefernde IP-Adressen aggregieren"
navTitle: "Einliefernde IPs"
description: "Eine einzige Auswertung zeigt, welche Systeme tatsächlich Mail in Ihren Tenant einliefern: vergessene Connectoren, direkt sendende Anwendungen und Dienstleister, die niemand dokumentiert hat, inklusive der typischen Auswertungsfehler bei Seitenlogik und Interpretation."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 Min. Lesezeit"
themen:
  - "microsoft-365-exchange"
  - "smtp-mailflow"
  - "exchange-onprem-hybrid"
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - "exchange-message-tracking-und-receive-connectoren-analysieren"
  - "ghost-sender-exchange-online-nebeneingang"
  - "mailflow-fehlersuche-kontrollierte-experimente"
slug: "einliefernde-ip-adressen-aggregieren"
translationId: "article-5879cc0eb17ed951"
url: "https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren"
draft: false
---

# Wer liefert eigentlich in Ihren Tenant ein? Einliefernde IP-Adressen aggregieren

Kaum eine Mailumgebung weiss noch vollständig, wer bei ihr einliefert. Über die Jahre sammeln sich Connectoren aus Migrationen, Anwendungen, die direkt senden, Dienstleister, deren Vertrag längst ausgelaufen ist, und Testaufbauten, die nie zurückgebaut wurden. Solange Mail fliesst, fällt das niemandem auf.

Eine einzige Auswertung schafft hier Klarheit: die Gruppierung aller eingehenden Nachrichten nach ihrer Quell-IP-Adresse. Sie ist in zwei Minuten gemacht, und die Ergebnisliste ist regelmässig überraschend. Dieser Artikel zeigt die Abfrage, erklärt, wie Sie sie **vollständig** bekommen, und vor allem, wie Sie die Zahlen richtig lesen. Denn die Interpretation ist der schwierigere Teil.

## Warum sich das lohnt

Die Liste beantwortet vier Fragen, die sonst mühsam einzeln zu klären sind. Welche Systeme senden überhaupt in Ihren Tenant? Läuft alles über die Wege, die Sie dokumentiert haben, oder gibt es einen zweiten Eingang? Wird ein Connector, den Sie abbauen wollen, noch genutzt? Und: Sendet eine Anwendung direkt an den Dienst vorbei an Ihrem Gateway, also unter Umgehung Ihrer Filterung?

Gerade die letzte Frage ist sicherheitsrelevant. Wer direkt einliefert, umgeht nicht nur die Filterung, sondern häufig auch die Protokollierung, auf die Sie sich im Ernstfall verlassen wollen.

## Die Abfrage

Im Tenant gruppieren Sie die Nachrichtenverfolgung nach `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Beginn des Abfragefensters, hier vor zwei Stunden |
| `-EndDate (Get-Date)` | Ende des Abfragefensters, der aktuelle Zeitpunkt |
| `-ResultSize 5000` | maximale Zeilenzahl pro Aufruf; 5000 ist zugleich der Höchstwert |
| `Group-Object FromIP` | gruppiert die Nachrichten nach der einliefernden IP-Adresse |
| `Sort-Object Count -Descending` | sortiert die Gruppen absteigend nach Nachrichtenzahl |
| `Format-Table Count, Name -AutoSize` | zweispaltige Ausgabe (Anzahl, IP-Adresse) mit automatischer Spaltenbreite |

</details>

Eine typische Ausgabe:

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Bevor Sie daraus Schlüsse ziehen, müssen zwei Dinge stimmen: Die Liste muss vollständig sein, und Sie müssen wissen, was die Einträge bedeuten.

## Fehlerquelle 1: Die Liste ist fast immer unvollständig

`Get-MessageTraceV2` liefert seitenweise, maximal 5000 Zeilen pro Aufruf. Bei hohem Aufkommen deckt eine Seite nur einen Bruchteil Ihres Zeitfensters ab. Sie gruppieren dann über einen Ausschnitt und halten das Ergebnis für die Gesamtheit.

Erkennbar ist das an dieser Warnung:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Erscheint diese Warnung, ist Ihre Auswertung wertlos.** Insbesondere darf ein fehlender Eintrag dann nicht als Abwesenheit gedeutet werden. Eine Adresse mit drei Nachrichten pro Tag taucht in einem Ausschnitt sowieso nicht auf.

Es gibt zwei Auswege. Der einfache: Verkleinern Sie das Zeitfenster, bis die Warnung ausbleibt. Bei 5000 Nachrichten pro Stunde sind das eben 55 Minuten und nicht sieben Tage. Für die Frage „welche Systeme senden überhaupt" genügt ein vollständiges kurzes Fenster meist völlig.

Der gründliche Weg blättert durch alle Seiten und sammelt ein:

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-StartDate` / `-EndDate` | Abfragefenster, hier die letzten 24 Stunden |
| `-StartingRecipientAddress` | Fortsetzungspunkt der Seitenlogik: die Empfängeradresse, ab der die nächste Seite beginnt |
| `-ResultSize 5000` | Seitengrösse; eine volle Seite signalisiert, dass weitere Ergebnisse folgen |
| `Group-Object FromIP` | gruppiert den Gesamtbestand nach der einliefernden IP-Adresse |
| `Sort-Object Count -Descending` | sortiert die Gruppen absteigend nach Nachrichtenzahl |
| `Format-Table Count, Name -AutoSize` | Ausgabe der Anzahl pro Adresse mit automatischer Spaltenbreite |

</details>

Die Schleife ruft so lange weitere Seiten ab, wie eine Seite mit genau 5000 Zeilen zurückkommt, und setzt dabei jeweils mit der letzten Empfängeradresse der Vorseite fort; erst der vollständige Bestand wird gruppiert.

Rechnen Sie bei 24 Stunden in einer mittleren Umgebung mit einigen Minuten Laufzeit. Für eine einmalige Bestandsaufnahme ist das gut investiert.

## Fehlerquelle 2: Die Zahlen bedeuten nicht, was sie zu bedeuten scheinen

Die Ergebnisliste enthält vier grundverschiedene Arten von Einträgen, und wer sie in einen Topf wirft, zieht falsche Schlüsse.

**`255.255.255.255` steht nicht für ein System.** Dieser Wert erscheint, wenn es zur Nachricht keine eingehende SMTP-Verbindung von aussen gab. Das betrifft im Dienst selbst erzeugte Nachrichten: Journalberichte, Unzustellbarkeitsmeldungen, Abwesenheitsnotizen, Nachrichten zwischen Postfächern desselben Tenants. In fast jeder Umgebung ist das der grösste Posten, und er ist völlig unauffällig.

**Private Adressen aus RFC 1918** stammen aus Ihrem eigenen Netz. In Hybrid-Umgebungen sehen Sie hier die lokalen Transportserver, denn deren interne Adresse wird bei der Übergabe an den Dienst erhalten. Das sind die grossen Zahlen in der Liste und in aller Regel der erwartete Hauptweg.

**Adressen von Sicherheits- und Filterdiensten** erkennen Sie am Betreiber, nicht am Zahlenwert. Cloud-Proxys, vorgelagerte Mailgateways und Web-Security-Dienste tauchen mit vielen benachbarten Adressen und mittleren Zahlen auf. Sie gehören meist dazu, sollten aber im Betriebshandbuch stehen.

**Öffentliche Einzeladressen mit kleinen Zahlen** sind die interessanten. Genau dort verstecken sich die vergessenen Anwendungen, alte Dienstleister und Systeme, von denen niemand mehr weiss.

## Die Auflösung: aus Adressen werden Namen

Für alles, was Sie nicht sofort zuordnen können, hilft die Rückwärtsauflösung. Sie ist nicht immer gesetzt und nicht immer verlässlich, aber sie liefert in der Mehrzahl der Fälle den entscheidenden Hinweis:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | fragt den Reverse-Eintrag (PTR) der jeweiligen IP-Adresse ab |
| `-ErrorAction Stop` | macht einen fehlenden Eintrag zu einem abfangbaren Fehler für den `try`/`catch`-Block |
| `[pscustomobject]@{ … }` | baut pro Adresse ein Objekt mit IP und aufgelöstem Namen für die Tabellenausgabe |
| `Format-Table -AutoSize` | Ausgabe mit automatischer Spaltenbreite |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

Ein fehlender PTR ist für sich genommen kein Hinweis auf ein Problem, aber er ist ein guter Grund, genauer hinzusehen. Nehmen Sie sich für solche Adressen die zugehörigen Nachrichten vor:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Abfragefenster und Seitengrösse wie in der Hauptabfrage |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtert clientseitig auf die fragliche Quelladresse |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | zeigt pro Nachricht Empfangszeit, Absender, Empfänger, Betreff und Zustellstatus |

</details>

Absender und Betreff sagen Ihnen in aller Regel sofort, welche Anwendung dahintersteckt.

## Der Abgleich: welche Adresse gehört zu welchem Connector?

Stellen Sie Ihre Ergebnisliste den konfigurierten Connectoren gegenüber:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-InboundConnector` | listet alle eingehenden Connectoren des Tenants; hier bewusst ohne einschränkende Parameter |
| `Format-List <Eigenschaften>` | Ausgabe als Liste der genannten Eigenschaften, eine pro Zeile |
| `@{n='…'; e={…}}` | berechnete Eigenschaft mit Name (`n`) und Ausdruck (`e`) |
| `-join ', '` | macht aus dem Array der Adressen bzw. Domänen eine lesbare, kommagetrennte Zeile |

</details>

Drei Konstellationen sind aufschlussreich.

**Eine Adresse liefert ein, ist aber in keinem Connector genannt.** Dann kommt die Mail als gewöhnliche Internet-Mail herein. Das ist zulässig, heisst aber: Diese Anwendung geniesst keinerlei Sonderbehandlung, und ihre Nachrichten unterliegen der vollen Filterung. Wenn jemand behauptet, für dieses System sei ein Connector eingerichtet, stimmt das offenkundig nicht mehr.

**Ein Connector nennt Adressen, von denen nichts kommt.** Der Kandidat für den Rückbau. Prüfen Sie, bevor Sie löschen, ob es sich um saisonale oder seltene Systeme handelt, und deaktivieren Sie zuerst, statt gleich zu entfernen.

**Ein Connector setzt `TreatMessagesAsInternal` oder `CloudServicesMailEnabled` auf wahr.** Hier lohnt genaues Hinsehen. Beide Einstellungen führen dazu, dass Nachrichten über diesen Weg als organisationsintern behandelt werden. Kommt darüber Mail aus dem Internet herein, umgeht sie damit Prüfungen, die für externe Nachrichten gedacht sind, unter anderem den Schutz vor gefälschten Absendern aus der eigenen Domäne. Für einen reinen Hybrid-Connector ist das richtig; für einen Connector, über den beliebige Systeme einliefern, ist es ein Befund.

## Was Sie typischerweise finden

Aus der Praxis, ohne Anspruch auf Vollständigkeit: einen Testconnector aus einer Migration, der seit Jahren aktiv ist. Eine Fachanwendung, die direkt an den Dienst sendet, obwohl alle glauben, sie laufe über das Gateway. Einen Newsletter-Dienstleister, dessen Vertrag ausgelaufen ist, der aber weiterhin zustellen darf. Und regelmässig einen Connector mit weit offenen Bedingungen, den einmal jemand angelegt hat, um ein akutes Problem zu lösen.

Keiner dieser Funde ist dramatisch für sich genommen. Zusammen ergeben sie das Bild einer Umgebung, die niemand mehr vollständig überblickt, und genau das ist das eigentliche Risiko.

## Grenzen der Methode

Drei Einschränkungen sollten Sie kennen.

Die Nachrichtenverfolgung reicht über das Cmdlet nur rund zehn Tage zurück. Für längere Zeiträume brauchen Sie die historische Suche, die asynchron läuft und bis zu 90 Tage abdeckt. Seltene Systeme, die monatlich senden, entgehen Ihnen sonst.

`FromIP` bedeutet nicht überall dasselbe. Bei Mail aus dem Internet ist es die Adresse des einliefernden Servers. Bei Hybrid-Mail ist es die Adresse Ihres lokalen Transportservers, nicht die des ursprünglichen Absenders. Die Auswertung zeigt Ihnen also die **letzte Station vor dem Dienst**, nicht die Herkunft.

Und die Zuordnung zu einem Connector ist im Tenant nicht direkt sichtbar. Sie schliessen darauf über Adresse, Zertifikat und Absenderdomäne. Für eine belastbare Aussage über die Nutzung eines einzelnen Connectors ist der Connector-Bericht im Exchange Admin Center unter Berichte und Mailfluss die bessere Quelle, weil er serverseitig über längere Zeiträume aggregiert.

## Als wiederkehrende Prüfung

Diese Auswertung eignet sich gut als Quartalsroutine. Legen Sie das Ergebnis ab und vergleichen Sie beim nächsten Durchgang. Neue Adressen in der Liste sind entweder dokumentierte Änderungen oder etwas, das Sie wissen wollen.

Wenn Sie dabei ohnehin die Mailkonfiguration Ihrer Domänen prüfen: Der [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) zeigt SPF, DKIM, DMARC und die übrigen Mailstandards für beliebige Domänen direkt im Browser, auch für Neben- und Marketingdomänen, die bei solchen Bestandsaufnahmen erfahrungsgemäss vergessen gehen. Und für die Abfragen selbst liefert der [Befehls-Generator](https://rafaelpfister.ch/tools/command-builder) fertige Bausteine für PowerShell und Unix-Shell.

Wie Sie einzelne auffällige Nachrichten weiterverfolgen, steht in [Exchange-Mailfluss analysieren: Message Tracking, SMTP-Protokolle und Receive-Connectoren](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Quellen

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Feldliste inklusive FromIP und ToIP sowie die Seitenlogik mit StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynchrone Nachrichtenverfolgung über bis zu 90 Tage für ältere Zeiträume.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): Parameter der eingehenden Connectoren, unter anderem SenderIPAddresses und TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Zusammenspiel der Connector-Typen und wann welcher greift.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): definiert die privaten Adressbereiche, die Sie in der Auswertung von öffentlichen Adressen unterscheiden müssen.
