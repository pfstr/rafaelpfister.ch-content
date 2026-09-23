---
title: "Mail-Schlaufe mit Verschlüsselungsgateway hinter EXO - Spoofing-Probleme verhindern"
navTitle: "Gateway-Schlaufe"
description: "Steht das Verschlüsselungsgateway (im Beispiel HIN) hinter Exchange Online, kommt jede eingehende Nachricht ein zweites Mal bei EOP an: mit fremder Absenderdomäne, ungültiger DKIM-Signatur und der Gateway-IP als Quelle. Das Ergebnis ist ein Spoof-Verdikt und Junk. Vier Einstellungen verhindern das, ohne die Filterung per SCL -1 abzuschalten: PTR-Record, Enhanced Filtering aus, Spoof-Ausnahme für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren."
date: "2026-09-23"
kategorie: "Mailfluss und SMTP"
timeToRead: "9 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "microsoft-365-exchange"
  - "hin-gateway"
  - "e-mail-verschluesselung"
produkte:
  - "exchange-online"
  - "hybrid-mailfluss"
  - "hin"
protokolle:
  - "mail-auth"
  - "smtp"
  - "powershell"
slug: "verschluesselungsgateway-hinter-exchange-online"
translationId: "article-b773c8d303aee87a"
url: "https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online"
aiPrompt: |
  Du bist mein Exchange-Online-Assistent. Ich betreibe ein Verschlüsselungsgateway hinter Exchange Online (Mail-Schlaufe: EXO, Gateway, EXO). Prüfe mit mir die vier Einstellungen: PTR-Record der Gateway-IP, Enhanced Filtering auf dem Inbound-Connector, Spoof-Ausnahme in der Tenant Allow/Block List für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren. Erkläre mir zu jeder Einstellung, warum sie nötig ist, und hilf mir, das Ergebnis anhand der Authentication-Results-Header einer Testnachricht zu verifizieren.
---
# Mail-Schlaufe mit Verschlüsselungsgateway hinter EXO - Spoofing-Probleme verhindern

Viele Organisationen im Schweizer Gesundheitswesen betreiben ein HIN-Gateway, andere ein SEPPmail oder totemomail für S/MIME und PGP. Zeigt der MX-Eintrag auf Microsoft und steht das Gateway hinter Exchange Online, durchläuft jede eingehende Nachricht die Filterung zweimal. Im zweiten Durchlauf sieht EOP eine Nachricht mit fremder Absenderdomäne, ungültiger DKIM-Signatur und der Gateway-IP als Quelle. Das ist aus Sicht des Filters das Muster einer Absenderfälschung, und legitime Post landet im Junk-Ordner. Ich habe diesen Aufbau mehrfach eingerichtet und beschreibe hier die vier Einstellungen, mit denen die Schlaufe ohne Spoof-Verdikt läuft, und warum jede einzelne davon nötig ist. Beispielhaft steht das HIN-Gateway; der Mechanismus ist bei jedem anderen Gateway in derselben Position identisch.

## Der Aufbau

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

Eine Transportregel leitet eingehende Nachrichten über einen Outbound-Connector an das Gateway. Das Gateway entschlüsselt, prüft Signaturen und liefert die Nachricht über einen Inbound-Connector wieder bei Exchange Online ein. Ein vom Gateway gesetztes Kopfzeilenfeld verhindert, dass die Regel erneut greift.

Der Aufbau hat gute Gründe: Microsoft filtert zuerst, das Gateway bekommt nur vorgeprüfte Post, und der Betrieb muss keinen eigenen MX ins Internet stellen. Der Preis ist der zweite Durchlauf, und der lässt sich nicht abschalten, nur richtig konfigurieren.

## Warum SCL -1 die falsche Antwort ist

Die verbreitete Abhilfe ist eine Transportregel auf dem Rückweg, die den Spam Confidence Level auf `-1` setzt. Damit überspringt Exchange Online die Inhaltsprüfung im zweiten Durchlauf vollständig. Das wirkt sofort und hat zwei Nachteile: Der zweite Durchlauf leistet danach nichts mehr, und die Regel hängt an einer Bedingung (Connector oder IP), die sich bei jeder Änderung verschiebt. Microsoft beschreibt SCL -1 zudem als Eingabe für die Filterung, nicht als endgültige Entscheidung; der gestempelte Wert kann abweichen. Die vier Einstellungen unten lösen das Problem an der Stelle, an der es entsteht: bei der Bewertung der einliefernden Infrastruktur.

## Die vier Einstellungen

1. Im öffentlichen DNS einen PTR-Record für die öffentliche IP des HIN-Gateways veröffentlichen. Ohne diesen Eintrag funktioniert der ganze Weg nicht.
2. Enhanced Filtering auf dem Inbound-Connector, über den HIN nach Exchange Online einliefert, vollständig abschalten.
3. Eine Spoof-Ausnahme für die einliefernde HIN-Infrastruktur in der Tenant Allow/Block List anlegen.
4. `CloudServicesMailEnabled` auf dem Outbound-Connector zu HIN und auf dem Inbound-Connector von HIN aktivieren.

Zu Punkt 2 zuerst prüfen, welche Connectoren betroffen sind:

```powershell
Get-InboundConnector |
    Select-Object Name, Enabled, ConnectorType, EFSkipLastIP, EFSkipIPs, EFUsers, EFTestMode |
    Format-List
```

Dann auf dem HIN-Connector abschalten:

```powershell
$efAus = @{
    Identity     = "<Inbound-Connector HIN>"
    EFSkipLastIP = $false
    EFSkipIPs    = $null
    EFUsers      = $null
}
Set-InboundConnector @efAus
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Parameter | Wirkung |
|---|---|
| `EFSkipLastIP = $false` | Der letzte Hop wird nicht mehr automatisch übersprungen. Steht gleichzeitig keine IP in `EFSkipIPs`, ist Enhanced Filtering auf dem Connector deaktiviert. |
| `EFSkipIPs = $null` | Leert die Liste der zu überspringenden IP-Adressen. |
| `EFUsers = $null` | Entfernt die Beschränkung auf einzelne Empfänger; ohne aktives Enhanced Filtering ist der Wert ohne Bedeutung, bleibt so aber eindeutig. |
| `EFTestMode` | Nur in der Abfrage: zeigt, ob der Connector im Testmodus steht. Microsoft führt den Parameter als intern, er ist aber lesbar. |

</details>

Punkt 3, die Spoof-Ausnahme:

```powershell
$spoof = @{
    Identity              = "Default"
    Action                = "Allow"
    SpoofedUser           = "*"
    SendingInfrastructure = "gateway.example.com"
    SpoofType             = "External"
}
New-TenantAllowBlockListSpoofItems @spoof
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Parameter | Wirkung |
|---|---|
| `Identity = "Default"` | Die Liste selbst; es gibt nur diese eine. |
| `Action = "Allow"` | Erlaubt die Kombination. `Block` stuft sie stattdessen als Phishing ein. |
| `SpoofedUser = "*"` | Die im Von-Feld sichtbare Adresse. Der Platzhalter steht für beliebige Absender. Zulässig ist ein Platzhalter auf einer Seite des Paares, nicht auf beiden. |
| `SendingInfrastructure` | Die Quelle: die Domäne aus dem PTR-Record der einliefernden IP (Punkt 1). Ohne PTR-Record akzeptiert die Liste nur `<IP>/24`. |
| `SpoofType = "External"` | Gilt für fremde Absenderdomänen. `Internal` deckt die eigenen akzeptierten Domänen ab und braucht einen zweiten Eintrag. |

</details>

Punkt 4, die Cross-Premises-Header:

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

Warum beim Inbound-Connector `TreatMessagesAsInternal` im selben Befehl steht, erkläre ich unten bei Punkt 4.

## Zu 1: PTR-Record

EOP identifiziert die einliefernde Infrastruktur über den Reverse-Lookup der Quell-IP. Der PTR-Wert erscheint im `Authentication-Results`-Header als sending infrastructure, und genau auf diesen Wert bezieht sich die Spoof-Ausnahme aus Punkt 3. Fehlt der PTR-Record, bewertet Exchange Online jede Nachricht auf dem Rückweg mit `PTR:InfoDomainNonexistent`, und die Ausnahme muss auf ein ganzes `/24` lauten statt auf einen Namen. Ein fehlender Reverse-DNS-Eintrag ist zudem ein seit langem etabliertes Negativsignal in jedem Spamfilter. Der Eintrag sollte vorwärts wieder auf dieselbe IP auflösen.

Bei HIN ist die einliefernde IP diejenige des HIN-Mailgateways, über das die Nachrichten nach Exchange Online zurückkommen. Wer den PTR-Record für diese IP verwaltet, hängt vom Betriebsmodell ab; die Zuständigkeit gehört vor der Umstellung geklärt.

## Zu 2: Enhanced Filtering abschalten

Enhanced Filtering for Connectors (Skip Listing) ist für den umgekehrten Aufbau gebaut: Gateway vor Microsoft 365, MX auf das Gateway. Dort überspringt EOP den letzten Hop und bewertet den echten Ursprung der Nachricht.

Im Schlaufen-Setup funktioniert das nicht. Microsoft hält in der Dokumentation fest, dass Enhanced Filtering nicht für Dienste gedacht ist, die Mail nach Microsoft 365 verarbeiten, und führt nichtlineares Routing (Internet, Microsoft 365, externes System, Microsoft 365) als nicht unterstützt auf. Als Folge nennt die Dokumentation genau das Symptom aus der Praxis: Microsoft 365 prüft die zurückkommende Post erneut, vergibt einen `compauth`-Wert, und die Nachricht kann als Spam eingestuft werden.

Bleibt Enhanced Filtering aktiv, überspringt EOP die HIN-IP und bewertet die IP davor. Das ist in der Schlaufe entweder eine Microsoft-eigene IP aus dem ersten Durchlauf oder der ursprüngliche externe Absender. Beides ist falsch:

* Die Bewertung greift auf einer IP, die im zweiten Durchlauf keine Aussage mehr über die Nachricht erlaubt.
* Die Spoof-Ausnahme aus Punkt 3 greift nie, weil sie an der HIN-Infrastruktur hängt, die EOP gerade übersprungen hat.

Deshalb muss Enhanced Filtering auf diesem Connector aus sein: erst dann ist HIN als einliefernde Infrastruktur sichtbar und adressierbar.

## Zu 3: Spoof-Ausnahme

Ein Spoof-Eintrag ist immer ein Paar aus Spoofed user (die From-Adresse oder deren Domäne) und Sending infrastructure (die Quelle). Erlaubt wird ausschliesslich diese Kombination.

`SpoofedUser = "*"` zusammen mit der HIN-Infrastruktur heisst also: Jede From-Adresse darf über HIN einliefern, ohne dass das Spoof-Verdikt greift. Jede andere Quelle, die dieselben Absender verwendet, wird weiterhin geprüft. Die Ausnahme ist kein Spamfilter-Bypass: Spam-, Inhalts- und Bedrohungsprüfung laufen im zweiten Durchlauf unverändert weiter. Eine Nachricht darf also weiterhin wegen ihres Inhalts aussortiert werden, sie gilt nur nicht mehr als Fälschung.

Im Befehl oben steht bewusst nur `SpoofType = "External"`. Eigene Mails, also Nachrichten mit Absender aus den eigenen akzeptierten Domänen, sollten nicht über das Gateway zurückkommen. Sie gehen von internen Systemen oder dem OnPrem-Exchange direkt nach Exchange Online, ohne HIN zu berühren. Ob das in Ihrer Umgebung tatsächlich so ist, zeigt die Spoof-Intelligence-Auswertung:

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

Tauchen dort eigene Domänen mit der HIN-Infrastruktur und `SpoofType Internal` auf, braucht es einen zweiten Eintrag mit `SpoofType = "Internal"`, oder besser: die Ursache, warum interne Post über das Gateway läuft.

Spoof-Einträge verfallen nicht von selbst. Fällt das Gateway weg oder ändert sich seine Infrastruktur, gehört der Eintrag entfernt.

## Zu 4: CloudServicesMailEnabled

Dieser Punkt ist der einzige, den Microsoft für dieses Szenario nicht ausdrücklich dokumentiert. Er ist meine Ableitung aus dem dokumentierten Verhalten des Parameters: Er erhält die Cross-Premises-Header über die Schlaufe hinweg.

Der Parameter steuert die Behandlung der internen `X-MS-Exchange-Organization-*`-Header. Auf dem Outbound-Connector werden sie in `X-MS-Exchange-CrossPremises-*` umgewandelt und überstehen so die Strecke über HIN. Auf dem Inbound-Connector werden sie wieder zu `X-MS-Exchange-Organization-*` zurückgeschrieben und ersetzen dabei gleichnamige Header, die bereits in der Nachricht stehen. Steht der Parameter auf `$false`, entfernt der Connector diese Header.

Praktisch heisst das: Was Exchange Online im ersten Durchlauf ermittelt hat, unter anderem der Authentifizierungsstatus und die interne Kennzeichnung, überlebt die Schlaufe, statt beim Wiedereintritt gestrippt zu werden. In den Kopfzeilen der zugestellten Nachricht steht dann `X-CrossPremisesHeadersPromoted`, und die ursprünglichen Prüfergebnisse bleiben unter `Authentication-Results-Original` erhalten. Das allein verhindert kein Spoof-Verdikt (dafür sind Punkt 1 bis 3 zuständig), es erhält aber das Urteil des ersten Durchlaufs.

Drei Dinge sind dabei zu beachten:

* Wurden die Connectoren vom Hybrid Configuration Wizard erstellt (`ConnectorSource: HybridWizard`), überschreibt ein späterer HCW-Lauf die Einstellung. Die HIN-Connectoren sollten deshalb eigene, manuell erstellte Connectoren sein.
* Auf dem Inbound-Connector schliessen sich `CloudServicesMailEnabled $true` und `TreatMessagesAsInternal $true` gegenseitig aus. Steht `TreatMessagesAsInternal` bereits auf `$true`, lehnt Exchange Online den Befehl ab. Deshalb gehören beide Parameter in denselben `Set-InboundConnector`-Aufruf, wie oben gezeigt.
* Microsoft empfiehlt, den Parameter nur auf Anweisung des Supports oder einer Produktdokumentation zu setzen. Für die Schlaufe gibt es diese Dokumentation nicht; die Entscheidung liegt bei Ihnen, und sie gehört dokumentiert.

## Einlieferung absichern

Weil Exchange Online nach Punkt 4 Header aus der Rückstrecke übernimmt und nach Punkt 3 jede From-Adresse aus der HIN-Infrastruktur akzeptiert, muss der Inbound-Connector so eingeschränkt sein, dass nur HIN darüber einliefern kann. Bei einem Connector vom Typ `Partner` sind das `RestrictDomainsToCertificate` mit `TlsSenderCertificateName` oder `RestrictDomainsToIPAddresses` mit `SenderIPAddresses`, jeweils zusammen mit `RequireTls`. Ohne diese Einschränkung könnte jede Quelle, die den Connector trifft, mit beliebigen Absendern und übernommenen Organisations-Headern einliefern.

```powershell
$absicherung = @{
    Identity                     = "<Inbound-Connector HIN>"
    RequireTls                   = $true
    RestrictDomainsToCertificate = $true
    TlsSenderCertificateName     = "gateway.example.com"
}
Set-InboundConnector @absicherung
```

## Kontrolle

Nach der Umstellung eine Testnachricht von einer externen Domäne mit DMARC-Richtlinie über das Gateway schicken und die Kopfzeilen der zugestellten Nachricht prüfen. Im `Authentication-Results`-Header des zweiten Durchlaufs sollte die HIN-Domäne als sending infrastructure stehen und `compauth` auf `pass` mit einem Grund aus dem Bereich der Tenant-Allow-Einträge, nicht mehr auf `fail reason=001`. `X-CrossPremisesHeadersPromoted` und `Authentication-Results-Original` zeigen, dass Punkt 4 wirkt. Der [Mail-Header-Analyzer](/tools/header-analyzer) auf dieser Site stellt beide Durchläufe als Flussgrafik dar und markiert die Sprünge zwischen Exchange Online und dem Gateway.

## Quellen

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Abgrenzung von linearem und nichtlinearem Routing, Hinweis auf Dienste hinter Microsoft 365, Folge `compauth` und Spam-Einstufung, SCL -1 als Eingabe statt Entscheidung, PowerShell-Parameter `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): Verhalten von `CloudServicesMailEnabled` (Umwandlung und Promotion der Cross-Premises-Header, Entfernen bei `$false`), Ausschluss mit `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` und `RestrictDomainsToIPAddresses` für Partner-Connectoren.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` auf dem ausgehenden Connector.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): Syntax der Spoof-Einträge, Platzhalterregeln, Bestimmung der Sendeinfrastruktur über PTR-Record oder `/24`, Abdeckung DMARC-bedingter Spoof-Verdikte.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): Parameter `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): Auswertung der erkannten Spoof-Paare der letzten sieben Tage.
