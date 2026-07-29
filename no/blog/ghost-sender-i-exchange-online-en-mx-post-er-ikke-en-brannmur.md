---
title: "Ghost Sender i Exchange Online: En MX-post er ikke en brannmur"
navTitle: "Ghost Sender"
description: "Direkte levering til Exchange Online omgår en forhåndskoblet gateway dersom leietakeren ikke uttrykkelig blokkerer den. Risikoen er reell, men årsaken er en ufullstendig konfigurert e-postflyt."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lesetid"
themen:
  - "microsoft-365-exchange"
slug: "ghost-sender-i-exchange-online-en-mx-post-er-ikke-en-brannmur"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/no/blog/ghost-sender-i-exchange-online-en-mx-post-er-ikke-en-brannmur"
---

# Ghost Sender i Exchange Online: En MX-post er ikke en brannmur

![En spøkelsesadmin holder døren ved siden av sikkerhetsporten åpen i datasenteret, mens e-poster kommer forbi filteret og direkte inn i innboksen.](../images/ghost-admin.png)

Angrepsmuligheten som InfoGuard Labs beskriver som «Ghost Sender», er reell: En angriper kan omgå en forhåndskoblet e-postgateway og levere direkte til Exchange Online. Forutsetningen er imidlertid at leietakeren fortsatt aksepterer denne direkte veien. Dette er ikke en universell sårbarhet i Exchange Online, men en ufullstendig sikret e-postflyttopologi.

En Mail Transfer Agent som betjener postbokser for et domene, mottar i utgangspunktet SMTP-forbindelser fra internett. MX-posten viser vanlige avsendere den ønskede leveringsveien. Den er verken en brannmurregel eller en tilgangsliste, og hindrer ingen i å henvende seg direkte til et kjent Exchange Online-endepunkt.

## Hva «Ghost Sender» faktisk viser

Scenarioet som [InfoGuard Labs beskriver](https://labs.infoguard.ch/posts/ghost-sender/), ser slik ut:

1. En organisasjon har postboksene sine i Exchange Online.
2. Den offentlige MX-posten peker til en forhåndskoblet Secure Email Gateway.
3. Exchange Online-endepunktet under `*.mail.protection.outlook.com` er fortsatt direkte tilgjengelig fra internett.
4. Administratoren har ikke begrenset Exchange Online slik at bare den forhåndskoblede gatewayen kan levere dit.
5. En angriper ignorerer MX-posten og leverer meldingen sin direkte til Exchange Online.

Den tiltenkte veien er dermed:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Denne veien er imidlertid fortsatt åpen:

```text
Angreifer -> Exchange Online -> Postfach
```

Dette er en alvorlig feilkonfigurasjon. Det forhåndskoblede filteret kan omgås denne veien; forfalskede avsendere, phishing og CEO-svindel blir dermed betydelig enklere. InfoGuard fortjener anerkjennelse for å ha synliggjort problemet, undersøkt utbredelsen og publisert en enkel test.

Men hvor er egentlig produktfeilen her?

Også medienes spissformuleringer hjelper lite med å sette dette i sammenheng. [Heise har overskriften at Exchange Online slipper forfalskede e-poster «uhindret gjennom»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), selv om bare bestemte, ufullstendig herdede tredjeparts- og hybridkonfigurasjoner er berørt. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) formulerer det langt mer treffende: ikke et sikkerhetshull i snever forstand, men et design- og konfigurasjonsproblem.

## «An MTA is doing MTA-Things»

Hver Exchange Online-leietaker har et offentlig SMTP-endepunkt. Dette endepunktet er ingen hemmelighet, og det skal heller ikke være det. Microsoft forklarer selv at Exchange Online som standard tar imot meldinger som er direkte adressert til postbokser som driftes der: [Dette er ganske enkelt slik e-post fungerer](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Også [SMTP beskriver selv MX-posten som en mekanisme for å finne det vanlige målsystemet](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Det følger ingen forpliktelse for målserveren til å avvise forbindelser via enhver annen tilgjengelig vert. En angriper trenger ikke følge den skiltede veien. Hvis en annen MTA er tilgjengelig, kjenner mottakerdomenet og aksepterer meldingen, vil den bli prøvd – på samme måte som spammere i flere tiår har forsøkt å nå dårligere beskyttede backup-MX-systemer.

Den som setter et tredjepartsfilter foran, endrer standardtopologien. «Exchange Online er min internett-e-postgateway» blir til «bare min tredjepartsgateway kan overføre internett-e-post til Exchange Online». Denne nye `Trust-Border` oppstår ikke gjennom en DNS-oppføring. Den må håndheves uttrykkelig på det mottakende systemet.

Nettopp dette dokumenterer Microsoft: Ved en ekstern MX skal det opprettes en innkommende connector av typen `Partner`, som for `SenderDomains *` bare aksepterer sertifikatet eller kilde-IP-adressene til den forhåndskoblede tjenesten. Meldinger levert direkte forbi gatewayen blir da avvist. Dette står ordrett i Microsofts veiledning [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius beskriver også denne «sideinngangen» utførlig i [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM og DMARC er ikke dørvakter

InfoGuard viser meldinger der SPF, DKIM og DMARC feiler, men som likevel havner i postboksen. Det ser spektakulært ut, men er ingen kryptografisk «bypass» av disse mekanismene. E-postene slipper nettopp ikke vellykket gjennom. De leverer `fail`. Det avgjørende er hvilken lokal handling det mottakende systemet utleder fra dette resultatet.

SPF kontrollerer om et system har tillatelse til å sende for konvoluttavsenderen. DKIM kontrollerer en signatur. DMARC knytter disse resultatene til det synlige avsenderdomenet og publiserer en ønsket behandling. Selv gjeldende [DMARC-standard RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) slår uttrykkelig fast at mottakeren kan ta hensyn til denne ønskede behandlingen, men ikke er forpliktet til det. DMARC er et viktig signal, men ikke en nettverksbasert tilgangskontroll.

Med en forhåndskoblet gateway kommer det i tillegg at Exchange Online først ser IP-adressen til denne gatewayen, og ikke den opprinnelige avsenderens. Til dette finnes [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Det rekonstruerer den opprinnelige kilden og forbedrer SPF-, DKIM-, DMARC-, anti-spoofing- og anti-phishing-vurderinger. Enhanced Filtering er imidlertid heller ikke en dørlås. Det erstatter ikke den restriktive partner-connectoren.

Feilkonfigurasjonen blir særlig tydelig når en administrator svekker eller helt omgår EOP-kontrollen med SCL-bypass, fordi det forhåndskoblede produktet allerede skal filtrere, samtidig som direkte internettlevering holdes åpen. Da har vedkommende ikke fått en beskyttelsesmekanisme «omgått», men bevisst unnlatt å sørge for effektiv beskyttelse ved én av to innganger.

Man kan absolutt kritisere Microsoft dersom en melding med tydelig synlig autentiseringsfeil havner i innboksen uten advarsel. Man kan kritisere semantikken til connectortypene, dokumentasjonen og manglende varsler i Configuration Analyzer. Alt dette er legitime poenger. Eksistensen av et offentlig tilgjengelig SMTP-endepunkt er imidlertid ikke en sikkerhetssårbarhet.

## «Direct Send» er ikke det samme som «direkte levering»

To ting blandes sammen i diskusjonen:

- **Direct Send** betegner hos Microsoft anonyme meldinger der konvoluttavsenderen (`5321.MailFrom`) bruker et eget Accepted Domain for leietakeren.
- **Direkte levering til Exchange Online** betegner generelt en SMTP-melding som ignorerer den publiserte tredjeparts-MX-en og leveres direkte til Exchange-endepunktet. Avsenderen kan også bruke et vilkårlig eksternt domene.

Bryteren

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

er nyttig dersom Direct Send ikke behøves. Den forhindrer spoofing av interne domener via denne veien. Den stenger imidlertid ikke hele sideinngangen for vilkårlige eksterne avsendere. Microsoft beskriver det nøyaktige virkeområdet i [cmdlet-dokumentasjonen for `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Den som vil forhindre «Ghost Sender» fullstendig, trenger fortsatt tilgangsbegrensning via en partner-connector eller en passende regel for e-postflyt.

## Må Microsoft virkelig gjøre alt for administratoren?

Nei. Den som legger et ekstra e-postfilter inn i en produksjonsklar transportkjede, tar ansvar for denne transportkjeden.

Leverandøren kan ikke pålitelig gjette om skannere, multifunksjonsenheter, SaaS-tjenester, hybridservere, partnerreléer eller andre legitime systemer fortsatt må sende direkte til Exchange Online ved siden av den eksterne MX-en. Et automatisk «MX-en peker et annet sted, så blokker alt annet» ville avbryte ønskede e-postflyter i mange reelle miljøer. Derfor må administratoren uttrykkelig definere den ønskede tillitsgrensen.

Likevel bør Microsoft gjøre det enklere for de ansvarlige. En god Configuration Analyzer burde oppdage en ekstern MX uten restriktiv partner-connector og advare tydelig. Oppsettsdialogen kunne forklare at en connector av typen «Din organisasjon» riktignok identifiserer passende forbindelser, men ikke automatisk avviser upassende forbindelser. Secure-by-default-brytere og bedre driftsrapporter ville også vært velkomne.

Dette ville vært meningsfull produktherding. Men det endrer ikke den tekniske vurderingen: En usikker spesialtopologi forblir en usikker konfigurasjon og blir ikke en zero-day bare fordi den er vidt utbredt.

## Slik stenges sideinngangen

For miljøer med forhåndskoblet filter bør minst disse punktene stå på sjekklisten:

1. **Dokumenter e-postflyten fullstendig.** Hvilke systemer har faktisk tillatelse til å levere til Exchange Online? Dette omfatter også hybrid-, applikasjons- og nødveier.
2. **Opprett en restriktiv partner-connector.** Bruk `SenderDomains *` og begrens levering til et sertifikat (foretrukket) eller vedlikeholdte kilde-IP-områder. En connector av typen `OnPremises` eller «Din organisasjon» håndhever ikke denne standard-avvis-effekten (se for eksempel også: [E-postruting mellom Apache James og Exchange Online](/blog/totemomail-m365)).
3. **Konfigurer Enhanced Filtering korrekt.** Dersom EOP fortsatt skal filtrere, må opprinnelig IP og avsenderinformasjon rekonstrueres riktig. Generelle SCL-`-1`-bypasser må vurderes kritisk.
4. **Deaktiver Direct Send dersom det ikke brukes.** Kontroller først med Message Trace eller tilgjengelige rapporter om skannere eller applikasjoner er avhengige av dette.
5. **Ikke bytt blindt.** Test og overvåk deretter gateway-IP-områder, sertifikatendringer, hybrid e-postflyt samt `onmicrosoft.com`-, Teams- og andre spesialveier.

Et forenklet eksempel på den IP-baserte varianten er:

```powershell
New-InboundConnector `
  -Name "Bare fra overordnet e-postgateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-områder-for-gatewayen> `
  -RequireTls $true
```

Der det er mulig, bør sertifikatbinding foretrekkes fremfor IP-tillatelseslisten. Endringer bør først gjennomføres i en kontrollert test, for en feilaktig tillatelsesliste gjør raskt den åpne sideinngangen til et fullstendig e-postavbrudd.

## Den enkle selvtesten

Testen som InfoGuard (og MSXFAQ) viser, er nyttig:

```powershell
Send-MailMessage `
  -SmtpServer <leietakernavn>.mail.protection.outlook.com `
  -To admin@<leietakerdomene> `
  -From noreply@example.com `
  -Subject "EXO-sideinngang" `
  -Body "Test-e-post direkte til leietakeren"
```

Med en korrekt begrenset partner-connector kan man forvente en SMTP-avvisning som `5.7.51 TenantInboundAttribution; Rejecting`. En alternativ transportregel kan først akseptere meldingen og deretter flytte den til karantene; derfor må man i tillegg til SMTP-svaret også kontrollere Message Trace, karantene og postboks. `Send-MailMessage` (utdatert) brukes her bare som en lett forståelig illustrasjon. Ethvert kontrollert SMTP-testverktøy oppfyller samme formål.

## En nyttig test med misvisende etikett

«Ghost Sender» er ikke en ny SMTP-utnyttelse. Det er et treffende navn på en åpen sideinngang der sikringen lenge har vært dokumentert av Microsoft, men som administratoren har latt stå åpen.

Det ironiske er at InfoGuard i sitt eget innlegg selv betegner problemet som «widespread and systematic misconfiguration» og avslutter med setningen «Ghost-Sender is a misconfiguration». Microsofts Security Response Center klassifiserte også først rapporten som ikke en sikkerhetssårbarhet. Faktaene er altså til stede i artikkelen: Bare tittelen, test-e-posten og «Vulnerability»-merkingen forteller dessverre en mer dramatisk historie.

Den nyttige delen av publiseringen er vekkerklokken: Mange selskaper har tilsynelatende ikke låst e-postflyten sin ordentlig. Den problematiske delen er påstanden om at Exchange Online har en universell sikkerhetssårbarhet av denne grunn. Nei: Exchange Online oppfører seg her først og fremst som en MTA. Det blir usikkert når tillitsgrensen ikke er ferdig konfigurert.

Må man virkelig gjøre alt for administratoren? Nei. Men man må tydeligvis stadig minne om at DNS-ruting ikke erstatter tilgangskontroll.

## Kilder

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): Den opprinnelige undersøkelsen, inkludert utbredelsesanalysen og den selvtrukne konklusjonen «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Nettesten publisert av InfoGuard for å kontrollere egen leietaker for den åpne sideinngangen.

3.  [MSXFAQ: Exchange Online som sideinngang for e-postmottak](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): Frank Carius’ vurdering: ingen feil i Exchange Online, men en feilkonfigurasjon hos administratoren.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft forklarer at direkte mottak av e-post til driftede postbokser er slik e-post fungerer, og avgrenser Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Den offisielle veiledningen med et eget trinn for restriktiv partner-connector ved ekstern MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Rekonstruerer den opprinnelige avsenderkilden bak en gateway; forbedrer vurderingen, men erstatter ikke connectoren.

7.  [Heise: Ghost-Sender – Exchange Online slipper forfalskede e-poster uhindret gjennom](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Eksempel på spissformulert dekning som generaliserer bestemte feilkonfigurasjoner.

8.  [Crow in the Cloud: Spøkelsene jeg ikke påkalte](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Treffende vurdering som et design- og konfigurasjonsproblem, inkludert beskyttelsestiltak.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Beskriver MX-posten som en mekanisme for å finne det vanlige målsystemet, ikke som tilgangskontroll.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Slår fast at mottakeren kan ta hensyn til den publiserte DMARC-behandlingen, men ikke må.

---

## Er e-postflyten din sikker?

Usikker på om Exchange Online-leietakeren din også har en åpen sideinngang? **adeptio** kontrollerer hele e-postflyten din: fra MX-poster, connectore og tredjepartsgatewayer til EOP, SPF, DKIM, DMARC og Direct Send. Praktisk, uavhengig og med konkrete anbefalinger.

De som vil kontrollere eller sikre e-postflyten sin ordentlig, kan gjerne avtale en uforpliktende konsultasjon:

**[Bestill en konsultasjon med adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
