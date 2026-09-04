---
title: "Hvor lenge forblir en SMTP-sesjon åpen? ConnectionTimeout 00:10:00 i Exchange og systemene der det er for kort"
navTitle: "Varighet for SMTP-sesjon"
description: "Exchange avslutter hver innkommende SMTP-sesjon etter ti minutter, også mens den overfører data. Hvilke avsendere som holder én forbindelse åpen så lenge, hvordan du kan lese den faktiske sesjonsvarigheten fra protokolloggen, og når ConnectionTimeout og ConnectionInactivityTimeout bør justeres på en relay-kobling."
date: "2026-09-03"
kategorie: "SMTP og e-postflyt"
timeToRead: "10 min lesetid"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "hvor-lenge-forblir-en-smtp-sesjon-apen-connectiontimeout-00-10-00-i-exchange-og-systemene-der"
translationId: "article-b40497933bbe0a88"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
translationOf: smtp-session-dauer-exchange-connectiontimeout
url: https://rafaelpfister.ch/no/blog/hvor-lenge-forblir-en-smtp-sesjon-apen-connectiontimeout-00-10-00-i-exchange-og-systemene-der
translationSourceHash: a107c4edd960dabb30ba1b6f263a693808a5edf6815747d81f5d446c103a7e79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:24:02.657Z
translationReview: automatic
---

# Hvor lenge forblir en SMTP-sesjon åpen? ConnectionTimeout 00:10:00 i Exchange og systemene der det er for kort

Kort oppsummert: En SMTP-sesjon har ingen naturlig slutt. RFC 5321 begrenser bare ventetiden på hvert neste trinn, og en klient kan levere flere meldinger over en åpen forbindelse så lenge serveren holder forbindelsen åpen. Exchange holder den som standard åpen i ti minutter på Receive-koblinger, og deretter avslutter serveren forbindelsen, uavhengig av om data overføres akkurat da. For Exchange-til-Exchange-trafikk og de fleste MTA-er er dette uten betydning, fordi disse avsenderne selv kobler til på nytt etter sekunder. For applikasjoner, gatewayer og lastgeneratorer som bruker én enkelt forbindelse for en hel utsendingsjobb, er verdien derimot årsaken til avbrudd som vises som tilkoblingsfeil i klienten og som `421 4.4.1 Connection timed out` i Exchange-protokolloggen.

## To tidsavbrudd med ulik betydning

En Receive-kobling har to tidsgrenser som ofte forveksles:

| Parameter | Betydning | Standard postboksserver | Standard Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | maksimal inaktiv tid uten klientaktivitet, deretter lukkes forbindelsen | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | maksimal samlet varighet for forbindelsen, også når den aktivt overfører data | 00:10:00 | 00:05:00 |

Begge verdiene godtar 1 sekund til 1 dag (`1.00:00:00`), og `ConnectionTimeout` må være større enn `ConnectionInactivityTimeout`. Verdiene gjelder per kobling, altså separat for den Internett-vendte `Default Frontend <Server>`, Transport-tjeneste-koblingen `Default <Server>` på port 2525 og hver selvopprettede relay-kobling.

Inaktivitetstidsavbruddet er ukritisk: Fem minutter tilsvarer nøyaktig minimumstiden RFC 5321 angir at en server skal vente på neste kommando, og en klient som ikke sender noe på fem minutter, har som regel selv glemt forbindelsen. Det samlede tidsavbruddet er særtrekket ved Exchange: Det teller fra forbindelsen opprettes og fortsetter mens klienten leverer melding etter melding. Etter ti minutter lukker Exchange forbindelsen på det punktet dialogen befinner seg, om nødvendig midt i en `DATA`-blokk.

På sendesiden finnes ikke motstykket: En Send-kobling har bare `ConnectionInactivityTimeOut` (standard ti minutter) og begrenser i stedet sesjoner via `SmtpMaxMessagesPerConnection`, som som standard er 20 meldinger. Exchange avslutter altså selv hver forbindelse som klient etter maksimalt 20 meldinger og oppretter en ny. Det er årsaken til at det samlede tidsavbruddet aldri merkes mellom Exchange-servere: Sesjonene varer sekunder.

## Hva RFC 5321 angir

Standarden definerer i avsnitt 4.5.3.2 minimumsventetider som en klient skal overholde per protokolltrinn før den gir opp forbindelsen:

| Trinn | Minimumstidsavbrudd på klientsiden |
|---|---|
| Vente på `220`-banneret | 5 minutter |
| Svar på `MAIL` | 5 minutter |
| Svar på `RCPT` | 5 minutter |
| Svar på `DATA` ( `354`) | 2 minutter |
| Sende en datablokk | 3 minutter |
| Svar på det avsluttende punktumet | 10 minutter |
| Server: Vente på neste kommando | minst 5 minutter |

Det finnes ingen øvre grense for en sesjons samlede varighet i RFC-en. En klient som leverer meldinger på samme forbindelse i tretti minutter og aldri er stille i mer enn noen få sekunder, oppfører seg standardmessig. Den siste klientverdien er påfallende: Ti minutters ventetid på svaret etter det avsluttende punktumet, fordi serveren i denne fasen mottar og overtar meldingen. Hvis klienten avbryter for tidlig her, er meldingen allerede levert og leveres en gang til ved neste forsøk. Den samme situasjonen oppstår speilvendt når serveren lukker forbindelsen i dette øyeblikket på grunn av det samlede tidsavbruddet.

Hvis en server lukker forbindelsen med `421`, skal klienten i henhold til avsnitt 3.8 behandle den pågående transaksjonen som om den hadde mottatt en `451`, altså som en midlertidig feil med nytt forsøk. En MTA med kø gjør nettopp det. En applikasjon uten kø rapporterer i stedet et unntak og overlater resten til kalleren.

## Hvor lenge avsendere faktisk holder sesjonene åpne

Sesjonsvarigheten bestemmes av klienten, og forskjellene mellom avsendertypene er store:

| Avsender | Typisk sesjonsvarighet | Begrenset av |
|---|---|---|
| Exchange Send-kobling | Sekunder | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix med tilkoblingsbuffer | maksimalt 5 minutter | `smtp_connection_reuse_time_limit` = 300s |
| Postfix uten tilkoblingsbuffer | én melding per forbindelse | Standardatferden til `smtp`-klienten |
| Applikasjon med `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | så lenge objektet lever: hele kjøringen ved en batchjobb | bare programkoden |
| Karantenevarsler fra e-postgatewayer | én sesjon per varslingskjøring | Produktatferd, delvis med `NOOP`-keepalive |
| Multifunksjonsenheter, skann-til-e-post | én melding per forbindelse, flere minutter ved store skanninger over langsomme linjer | Filstørrelse og båndbredde |
| Lastgeneratorer som `smtp-source -d` | til slutten av kjøringen | Kallparametere |

De to første radene forklarer hvorfor verdien ikke har blitt lagt merke til av noen i klassiske miljøer på mange år: MTA-er gjør selv forbindelsene korte. Postfix bruker for eksempel en bufret forbindelse i høyst fem minutter og åpner deretter en ny, og Exchange kobler fra etter 20 meldinger. Begge holder seg dermed under enhver standardverdi i Exchange.

Applikasjonsraden er det vanligste problemtilfellet. En batchjobb som sender fakturaer, lønnsslipper eller systemvarsler, oppretter typisk et klientobjekt, kaller sendemetoden på det i en løkke og lukker det til slutt. `System.Net.Mail.SmtpClient` bruker siden .NET Framework 4 samme forbindelse for påfølgende `Send`-kall og sender `QUIT` først ved `Dispose`; JavaMail oppfører seg på samme måte med en `Transport` som er åpnet én gang. Hvis jobben varer lenger enn ti minutter, oppstår `421` et sted midt i kjøringen, og jobben avbrytes med et unntak, i .NET for eksempel med teksten `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out`. Hvilken melding som rammes, avhenger av kjøretiden, og derfor virker feilen tilfeldig: Noen ganger er det 800 meldinger før avbruddet, andre ganger 1200, avhengig av meldingsstørrelse og serverlast.

Gateway-raden beskriver et dokumentert tilfelle: Symantec (nå Broadcom) Messaging Gateway sender spam-karantenevarsler via én enkelt forbindelse og sender `NOOP` som keepalive mellom meldingene. Exchange svarer på `NOOP` med Tarpit-forsinkelsen på fem sekunder, slik at maksimalt omtrent 120 varsler kommer gjennom på ti minutter før sesjonen avsluttes med `421 4.4.1` og gatewayen må koble til på nytt.

Skanner-raden er et størrelsesproblem fremfor et mengdeproblem: En skanning på 60 MB over en 2-Mbit/s-forbindelse trenger rundt fire minutter ren overføringstid, ved 100 MB nesten syv minutter. På en Edge Transport-server med fem minutters samlet tidsavbrudd er det allerede nok til et avbrudd, på en postboksserver er det fortsatt en reserve, men ikke mye.

## Hva som skjer ved avbruddet

Når det samlede tidsavbruddet utløper, skriver Exchange svaret `421 4.4.1 Connection timed out` til protokolloggen, sender det til klienten og lukker forbindelsen. For transaksjonen som pågår, gjelder følgende: Hvis det avsluttende punktumet ennå ikke er sendt, er meldingen ikke mottatt og må gjentas fullstendig. Hvis punktumet er sendt og forbindelsen lukkes før `250`-svaret, har klienten ingen informasjon om hvorvidt Exchange har overtatt meldingen; en korrekt implementert klient gjentar den, og mottakeren kan da få den to ganger. Sannsynligheten for dette er liten, men ved tusenvis av meldinger per kjøring er den ikke null.

Proxy-banen må også tas i betraktning: Front End Transport-tjenesten tar imot forbindelsen på port 25 og videresender den som en egen SMTP-sesjon til Transport-tjenesten på port 2525, der koblingen `Default <Server>` gjelder med de samme standardverdiene. En lang sesjon vises derfor i begge logger, og en justering må omfatte begge koblingene.

## Les den faktiske sesjonsvarigheten fra protokolloggen

Før du endrer en verdi, er det verdt å se på de reelle sesjonene. Forutsetningen er detaljert protokollogging på den berørte koblingen; på `Default Frontend <Server>` er dette allerede aktivt, men ikke på alle andre koblinger:

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

Loggene ligger under `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (Front End) og `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (Transport-tjenesten), og er navngitt etter UTC-time som `RECVyyyyMMddhh-nnnn.log`. Hver linje er en protokollhendelse med feltene `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data` og `context`. Alle linjer i en sesjon har samme `session-id`, så sesjonsvarigheten er differansen mellom første og siste tidsstempel for denne ID-en.

Følgende skript analyserer dagens nyeste loggfil for en kobling, sammenfatter linjene per sesjon og viser de 15 lengste sesjonene med antall meldinger, varighet og informasjon om hvorvidt Exchange avsluttet dem med `421 4.4.1`:

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Element | Effekt |
|---|---|
| `$logPfad` | Loggkatalog for Front End Transport-tjenesten; bruk `Hub` i stedet for `FrontEnd` for Transport-tjenesten |
| `$connector` | Navnebestanddel av koblingen; filtrerer via feltet `connector-id`, som logges som `Server\Name` |
| `$tag` | UTC-dato, fordi loggfilene er navngitt etter UTC-time |
| `-Filter "RECV$tag*.log"` | bare Receive-logger for dagens dato |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | den nyeste filen (høyeste time, høyeste instansnummer) |
| `$_ -notlike '#*'` | hopper over topptekstlinjene `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | deler opp CSV-linjen; feltene 0, 2, 5 og 7 som brukes, ligger før den første friteksten og er derfor stabile |
| `$c[2]` | `session-id`, grupperingsnøkkelen |
| `($c[5] -split ':')[0]` | IPv4-adresse fra `remote-endpoint` (ved IPv6-endepunkter må oppdelingen tilpasses) |
| `$c[0]` som `Start` og `Ende` | første og siste tidsstempel i sesjonen; `Ende` overskrives med hver linje |
| `$c[7] -like 'MAIL FROM*'` | teller meldinger via mottatt `MAIL FROM`-kommando |
| `$c[7] -like '421 4.4.1*'` | markerer sesjoner som Exchange avsluttet på grunn av det samlede tidsavbruddet |
| `Sort-Object Zeilen -Descending` | de mest aktive sesjonene først; alternativt sorter etter `Dauer_s` |
| `Dauer_s` | differanse mellom ISO-8601-tidsstemplene i sekunder, avrundet til én desimal |

</details>

I utdataene kan du kjenne igjen de berørte systemene ved at `Timeout` er satt til `True` og `Dauer_s` ligger like ved 600: Sesjonen har levd nøyaktig så lenge koblingen tillater. Sesjoner med mange meldinger og en varighet klart under 600 sekunder er ukritiske, selv om de for øyeblikket er de lengste. For en oversikt over hvilke kilder som er berørt, er det nok å gruppere de markerte sesjonene:

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

To begrensninger ved tilnærmingen: En sesjon som går over en timegrense, fordeles på to loggfiler og vises forkortet i enkeltfilen; for en dagsanalyse må du lese inn alle dagens filer. Og verdien `Mails` teller `MAIL FROM`-kommandoer, altså forsøk, ikke mottatte meldinger.

## Justere verdier: på hvilken kobling og hvor mye

Standardverdiene er en beskyttelse for den Internett-vendte koblingen, der vilkårlige motparter kan oppta forbindelser. Der forblir de uendret; en legitim ekstern MTA kobler uansett til på nytt. Det er den dedikerte koblingen som justeres, og som de identifiserte interne systemene leverer via. Hvis en slik kobling mangler, kan den opprettes begrenset til avsender-IP-ene med `RemoteIPRanges`; det er bedre enn å øke verdien på `Default Frontend`. Du får gjeldende status for alle koblinger med:

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

Selve justeringen, her med én times samlet varighet og uendret inaktivitetstidsavbrudd:

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Parameter | Effekt |
|---|---|
| `ConnectionTimeout` | samlet varighet for en forbindelse; tillatt 00:00:01 til 1.00:00:00, må være over `ConnectionInactivityTimeout` |
| `ConnectionInactivityTimeout` | inaktivitetstid før lukking; fem minutter tilsvarer RFC-minimumet og kan beholdes |
| `-Identity 'EX01\Relay Applikationen'` | Front End-koblingen for de interne avsenderne |
| `-Identity 'EX01\Default EX01'` | Transport-tjeneste-koblingen på port 2525, som Front End videresender sesjonen til |
| `@werte` | splatting: sender begge parameterne fra hashtabellen til cmdleten |

</details>

Følgende gjelder for verdien: Den skal være over den lengste legitime sesjonen analysen viste, med reserve for belastningstopper. Én time dekker de fleste batchjobber; for en nattlig kjøring på to timer trengs tilsvarende mer, opptil maksimumet på én dag. Verdien bør imidlertid heller ikke være vilkårlig høy på en intern kobling, fordi `MaxInboundConnectionPerSource` (standard 20) og `MaxInboundConnection` (standard 5000) også teller med: En klient som i tillegg til en fastlåst forbindelse stadig åpner nye, når grensen per kilde desto tidligere jo lenger de gamle forbindelsene forblir åpne.

For avsendere som sender `NOOP` mellom meldinger, bør `TarpitInterval` settes til `00:00:00` på samme kobling. Tarpit-forsinkelsen har ingen nytte for autentiserte eller IP-begrensede interne avsendere og forlenger hver sesjon kunstig.

Endringen på Exchange-siden løser symptomet. Den mer stabile løsningen ligger i klienten: Den oppretter forbindelsen på nytt etter et fast antall meldinger, slik Exchange gjør med 20 og Postfix med fem minutter. Med `.NET SmtpClient` betyr det å opprette og forkaste objektet per blokk på for eksempel 100 meldinger; i JavaMail lukkes og åpnes `Transport` på nytt tilsvarende. Da fungerer utsendingen også mot mål der tidsavbruddene ikke kan justeres, særlig Exchange Online, hvis innkommende koblinger ikke har timeout-parametere.

## Flere tidsgrenser på banen

Exchange-verdien er ikke den eneste grensen. Brannmurer og lastbalanserere har egne inaktivitetstidtakere for TCP-forbindelser: En FastL4-profil på en F5 BIG-IP er som standard satt til 300 sekunder, og en Azure Load Balancer til fire minutter. Disse tidtakerne måler inaktivitet, ikke samlet varighet, og slår derfor inn ved sendepauser, for eksempel når en batchjobb leser data fra databasen mellom to blokker. Det er alltid den minste verdien på hele banen som gjelder. Hvordan du dimensjonerer tidsavbrudd på en lastbalanserer for vedvarende SMTP-forbindelser, beskrives i artikkelen [F5 BIG-IP som utgående proxy for masseutsending av e-post](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand).

## Kilder

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector): Referanse med standardverdier og verdiområder for `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection` og `MaxInboundConnectionPerSource` for postboks- og Edge Transport-servere.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector): `ConnectionInactivityTimeOut` og `SmtpMaxMessagesPerConnection` på sendesiden.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Lagringssteder, filnavn og feltstruktur for SMTP-protokolloggene for Front End- og Transport-tjenesten.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Front End Transport-tjenesten som tilstandsløs proxy foran Transport-tjenesten.

5.  [RFC 5321, avsnitt 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2): Minimumsventetider per protokolltrinn, begrunnelsen for de ti minuttene etter det avsluttende punktumet og atferden ved `421` i avsnitt 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html): `smtp_connection_reuse_time_limit` (300s) og `smtpd_timeout` som eksempel på en MTA som selv holder sesjonene korte.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html): Dokumentert tilfelle av en gateway som havner i Exchanges samlede tidsavbrudd med `NOOP`-keepalive og Tarpit.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Gjenbruk av forbindelser over flere `Send`-kall og `QUIT` først ved `Dispose`.
