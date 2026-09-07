---
title: "Ghost Sender i Exchange Online: En MX-post er ikke en brannmur"
navTitle: "Ghost Sender"
description: "Direkte levering til Exchange Online omgår en forhåndskoblet gateway dersom leietakeren ikke uttrykkelig blokkerer den. Risikoen er reell, men årsaken er en ufullstendig konfigurert e-postflyt."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lesetid"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-i-exchange-online-en-mx-post-er-ikke-en-brannmur"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
translationId: article-d8dc8d1da6379d67
translationReview: required
translationSourceHash: 6a500f1ed53a180322afb3c86e44376100d68659eeb55ffae35937ab434c6b61
translatedAt: 2026-09-05T07:51:02.664Z
url: https://rafaelpfister.ch/no/blog/ghost-sender-i-exchange-online-en-mx-post-er-ikke-en-brannmur
translationModel: gpt-5.6-terra
---

# Ghost Sender i Exchange Online: En MX-post er ikke en brannmur

![En Ghost-admin holder døren ved siden av sikkerhetsporten åpen i datasenteret, mens e-poster kommer forbi filteret og direkte inn i postboksen.](../images/ghost-admin.png)

Angrepsmuligheten som InfoGuard Labs beskriver som «Ghost Sender», er reell: En angriper kan omgå en forhåndskoblet e-postgateway og levere direkte til Exchange Online. Forutsetningen er imidlertid at leietakeren fortsatt aksepterer denne direkte veien. Dette er ikke en universell sårbarhet i Exchange Online, men en ufullstendig sikret e-postflyttopologi.

En Mail Transfer Agent som betjener postbokser for et domene, tar som hovedregel imot SMTP-tilkoblinger fra internett. MX-posten viser vanlige avsendere ønsket leveringsvei. Den er verken en brannmurregel eller en tilgangsliste, og hindrer ingen i å kontakte et kjent Exchange Online-endepunkt direkte.

## Hva «Ghost Sender» faktisk viser

Scenarioet som [InfoGuard Labs beskriver](https://labs.infoguard.ch/posts/ghost-sender/) ser slik ut:

1. En organisasjon har postboksene sine i Exchange Online.
2. Den offentlige MX-posten peker mot en forhåndskoblet Secure Email Gateway.
3. Exchange Online-endepunktet under `*.mail.protection.outlook.com` forblir direkte tilgjengelig fra internett.
4. Administratoren har ikke begrenset Exchange Online slik at bare den forhåndskoblede gatewayen kan levere dit.
5. En angriper ignorerer MX-posten og leverer meldingen sin direkte til Exchange Online.

Den tiltenkte veien er altså:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Men denne veien står fortsatt åpen:

```text
Angreifer -> Exchange Online -> Postfach
```

Dette er en alvorlig feilkonfigurasjon. Det forhåndskoblede filteret kan omgås via denne veien; forfalskede avsendere, phishing og CEO-svindel blir dermed betydelig enklere. InfoGuard fortjener anerkjennelse for å ha synliggjort problemet, undersøkt utbredelsen og publisert en enkel test.

Men hvor er egentlig produktfeilen her?

Også medienes tilspissing hjelper lite med vurderingen. [Heise har overskriften at Exchange Online slipper forfalskede e-poster «uten videre gjennom»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), selv om det kun er bestemte, ufullstendig hardenede tredjeparts- og hybridkonfigurasjoner som rammes. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) formulerer det langt mer presist: ikke et sikkerhetshull i snever forstand, men et design- og konfigurasjonsproblem.

## «An MTA is doing MTA-Things»

Hver Exchange Online-leietaker har et offentlig SMTP-endepunkt. Dette endepunktet er ingen hemmelighet, og skal heller ikke være det. Microsoft forklarer selv at Exchange Online som standard godtar meldinger som er adressert direkte til postbokser som er driftet der: [Det er rett og slett slik e-post fungerer](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Også [SMTP selv beskriver MX-posten som en mekanisme for å finne det vanlige målsystemet](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Det innebærer ingen plikt for målserveren til å avvise tilkoblinger via enhver annen tilgjengelig vert. En angriper trenger ikke følge den skiltede veien. Hvis en annen MTA er tilgjengelig, kjenner mottakerdomenet og aksepterer meldingen, blir den forsøkt brukt, omtrent slik spammere i flere tiår har forsøkt å kontakte dårligere beskyttede backup-MX-systemer.

Den som kobler inn et tredjepartsfilter, endrer standardtopologien. «Exchange Online er min internett-e-postgateway» blir til «bare tredjeparts-gatewayen min kan overføre internett-e-post til Exchange Online». Denne nye `Trust-Border` oppstår ikke gjennom en DNS-oppføring. Den må håndheves uttrykkelig på mottakersystemet.

Microsoft dokumenterer nettopp dette: Ved en ekstern MX skal det opprettes en Inbound Connector av typen `Partner`, som for `SenderDomains *` bare godtar sertifikatet eller kilde-IP-adressene til den forhåndskoblede tjenesten. Meldinger som leveres direkte utenom gatewayen, blir da avvist. Dette står ordrett i Microsofts veiledning [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Også Frank Carius beskriver denne «sideinngangen» utførlig i [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM og DMARC er ikke dørvakter

InfoGuard viser meldinger der SPF, DKIM og DMARC feiler, men som likevel havner i postboksen. Det ser spektakulært ut, men er ingen kryptografisk «omgåelse» av disse mekanismene. E-postene slipper nettopp ikke gjennom med hell. De leverer `fail`. Det avgjørende er hvilken lokal handling mottakersystemet utleder av dette resultatet.

SPF kontrollerer om et system har lov til å sende for konvoluttavsenderen. DKIM kontrollerer en signatur. DMARC knytter disse resultatene til det synlige avsenderdomenet og publiserer ønsket behandling. Selv den gjeldende [DMARC-standarden RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) slår uttrykkelig fast at mottakeren kan ta hensyn til denne ønskede behandlingen, men ikke er forpliktet til det. DMARC er et viktig signal, men ingen nettverksbasert tilgangskontroll.

Ved en forhåndskoblet gateway kommer det i tillegg at Exchange Online først ser IP-adressen til denne gatewayen, og ikke den opprinnelige avsenderens. Til dette finnes [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Det rekonstruerer den opprinnelige kilden og forbedrer SPF-, DKIM-, DMARC-, anti-spoofing- og anti-phishing-evalueringer. Enhanced Filtering er imidlertid heller ingen dørlås. Det erstatter ikke den restriktive partner-connectoren.

Feilkonfigurasjonen blir særlig åpenbar når en administrator svekker EOP-kontrollen med SCL-bypass, eller fjerner den helt, fordi det forhåndskoblede produktet allerede skal filtrere, samtidig som direkte levering fra internett står åpen. Da har vedkommende ikke fått en beskyttelsesmekanisme «omgått», men bevisst ikke lenger sørget for effektiv beskyttelse ved én av to innganger.

Man kan absolutt kritisere Microsoft dersom en melding med en tydelig synlig autentiseringsfeil havner i innboksen uten advarsel. Man kan kritisere semantikken til connector-typene, dokumentasjonen og manglende advarsler i Configuration Analyzer. Alt dette er legitime punkter. Eksistensen av et offentlig tilgjengelig SMTP-endepunkt er imidlertid ingen sikkerhetssårbarhet.

## «Direct Send» er ikke det samme som «direkte levering»

To ting blandes sammen i diskusjonen:

- **Direct Send** betegner hos Microsoft anonyme meldinger der konvoluttavsenderen (`5321.MailFrom`) bruker leietakerens eget Accepted Domain.
- **Direkte levering til Exchange Online** betegner generelt en SMTP-melding som ignorerer den publiserte tredjeparts-MX-en og leveres direkte til Exchange-endepunktet. Avsenderen kan også bruke et vilkårlig eksternt domene.

For Direct Send finnes det en egen bryter:

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-RejectDirectSend $true` | Avviser anonyme direkteleveringer der konvoluttavsenderen bruker et Accepted Domain for leietakeren |

</details>

Bryteren er fornuftig dersom Direct Send ikke er nødvendig. Den forhindrer spoofing av interne domener via denne veien. Den stenger imidlertid ikke hele sideinngangen for vilkårlige eksterne avsendere. Microsoft beskriver det nøyaktige virkeområdet i [cmdlet-dokumentasjonen for `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Den som vil forhindre «Ghost Sender» fullstendig, trenger fortsatt tilgangsbegrensning via partner-connector eller en passende e-postflytregel.

## Må Microsoft virkelig gjøre alt for administratoren?

Nei. Den som setter inn et ekstra e-postfilter i en produksjonskjede for transport, overtar ansvaret for denne transportkjeden.

Leverandøren kan ikke pålitelig gjette om skannere, multifunksjonsenheter, SaaS-tjenester, hybridservere, partner-reléer eller andre legitime systemer i tillegg til den eksterne MX-en fortsatt må sende direkte til Exchange Online. En automatisk «MX-en peker et annet sted, så blokkerer jeg alt annet» ville avbrutt ønskede e-postflyter i en rekke reelle miljøer. Derfor må administratoren uttrykkelig definere ønsket tillitsgrense.

Likevel bør Microsoft gjøre det enklere for de ansvarlige. En god Configuration Analyzer bør gjenkjenne en ekstern MX uten restriktiv partner-connector og advare tydelig. Oppsettdialogen kunne forklare at en connector av typen «Din organisasjon» riktignok identifiserer passende tilkoblinger, men ikke automatisk avviser upassende tilkoblinger. Secure-by-default-brytere og bedre driftsrapporter ville også være velkomne.

Dette ville vært fornuftig produktherding. Det endrer imidlertid ikke den tekniske vurderingen: En usikker spesialtopologi forblir en usikker konfigurasjon og blir ikke en zero-day bare fordi den er utbredt.

## Slik stenges sideinngangen

For miljøer med forhåndskoblet filter bør minst disse punktene stå på sjekklisten:

1. **Dokumenter e-postflyten fullstendig.** Hvilke systemer har faktisk lov til å levere til Exchange Online? Dette omfatter også hybrid-, applikasjons- og nødveier.
2. **Opprett en restriktiv partner-connector.** Bruk `SenderDomains *` og begrens levering til et sertifikat (foretrukket) eller vedlikeholdte kilde-IP-områder. En connector av typen `OnPremises` eller «Din organisasjon» håndhever ikke denne default-deny-effekten (se for eksempel også: [E-postruting mellom Apache James og Exchange Online](/blog/totemomail-m365)).
3. **Konfigurer Enhanced Filtering korrekt.** Dersom EOP fortsatt skal filtrere, må opprinnelig IP og avsenderinformasjon rekonstrueres korrekt. Generelle SCL-`-1`-bypasser må vurderes kritisk.
4. **Deaktiver Direct Send dersom det ikke brukes.** Kontroller først med Message Trace eller tilgjengelige rapporter om skannere eller applikasjoner er avhengige av det.
5. **Ikke bytt ukritisk.** Test og overvåk deretter gateway-IP-områder, sertifikatbytter, hybrid e-postflyt samt `onmicrosoft.com`-, Teams- og andre spesialveier.

Et forenklet eksempel på den IP-baserte varianten er:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Name` | Visningsnavn for den nye Inbound Connector-en |
| `-ConnectorType Partner` | Connector-klasse for eksterne partnersystemer; bare denne typen håndhever avvisning av upassende tilkoblinger |
| `-SenderDomains *` | Connector-en gjelder e-post fra alle avsenderdomener |
| `-RestrictDomainsToIPAddresses $true` | Aktiverer sperringen: E-post fra de angitte domenene godtas nå bare fra adressene i `-SenderIpAddresses` |
| `-SenderIpAddresses` | Tillatte kilde-IP-adresser eller -områder for den forhåndskoblede gatewayen |
| `-RequireTls $true` | Krever TLS-kryptering for tilkoblinger via denne connector-en |

</details>

Der det er mulig, bør sertifikatbinding foretrekkes fremfor IP-allowlist. Endringer bør først gjennomføres i en kontrollert test, for en feilaktig allowlist gjør raskt den åpne sideinngangen til et fullstendig e-postutfall.

## Den enkle selvtesten

Testen som InfoGuard (og MSXFAQ) viser, er nyttig:

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-SmtpServer` | Målvert: leietakerens offentlige Exchange Online-endepunkt, bevisst utenom MX-en |
| `-To` | Mottakeradresse i leietakeren som skal testes |
| `-From` | Vilkårlig ekstern avsenderadresse; nettopp dette skal sideinngangen egentlig ikke lenger godta |
| `-Subject` | Emnelinje, for å finne den igjen i Message Trace |
| `-Body` | Meldingstekst |

</details>

Ved en korrekt begrenset partner-connector kan man forvente en SMTP-avvisning som `5.7.51 TenantInboundAttribution; Rejecting`. En alternativ transportregel kan først godta meldingen og deretter flytte den til karantene; derfor må både SMTP-svaret, Message Trace, karantene og postboks kontrolleres. `Send-MailMessage` (deprecated) brukes her kun som en lett forståelig illustrasjon. Ethvert kontrollert SMTP-testverktøy tjener samme formål.

## En nyttig test med misvisende etikett

«Ghost Sender» er ikke en ny SMTP-eksploit. Det er et slående navn for en åpen sideinngang, hvis sikring Microsoft har dokumentert lenge, og som administratoren har latt stå åpen.

Det ironiske er at InfoGuard selv omtaler problemet i eget bidrag som «widespread and systematic misconfiguration» og avslutter med setningen «Ghost-Sender is a misconfiguration». Microsofts Security Response Center klassifiserte også først rapporten som ingen sikkerhetssårbarhet. Faktaene finnes altså i artikkelen: Bare tittelen, test-e-posten og «Vulnerability»-branding antyder dessverre en mer dramatisk tolkning.

Den fornuftige delen av publiseringen er vekkeropet: Mange selskaper har tydeligvis ikke låst e-postflyten sin ordentlig. Den problematiske delen er påstanden om at Exchange Online har en universell sikkerhetssårbarhet for dette. Nei: Exchange Online oppfører seg her først og fremst som en MTA. Det blir usikkert gjennom en tillitsgrense som ikke er konfigurert ferdig.

Må man virkelig gjøre alt for administratoren? Nei. Men man må tydeligvis stadig minne om at DNS-ruting ikke erstatter tilgangskontroll.

## Kilder

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): Den opprinnelige undersøkelsen, inkludert utbredelsesanalyse og konklusjonen «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Nettbasert test publisert av InfoGuard for å kontrollere egen leietaker for den åpne sideinngangen.

3.  [MSXFAQ: Exchange Online som sideinngang for e-postmottak](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): Frank Carius' vurdering: ingen feil i Exchange Online, men en feilkonfigurasjon hos administratoren.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft forklarer at direkte mottak av e-post til driftsede postbokser er slik e-post fungerer, og avgrenser Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Den offisielle veiledningen med eget trinn for restriktiv partner-connector ved ekstern MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Rekonstruerer den opprinnelige avsenderkilden bak en gateway; forbedrer evalueringen, men erstatter ikke connector-en.

7.  [Heise: Ghost-Sender – Exchange Online slipper forfalskede e-poster uten videre gjennom](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Eksempel på tilspisset dekning som generaliserer bestemte feilkonfigurasjoner.

8.  [Crow in the Cloud: Åndene jeg ikke kalte på](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Presis vurdering som et design- og konfigurasjonsproblem, inkludert beskyttelsestiltak.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Beskriver MX-posten som en mekanisme for å finne det vanlige målsystemet, ikke som tilgangskontroll.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Slår fast at mottakeren kan ta hensyn til den publiserte DMARC-behandlingen, men ikke må.

---

## Er e-postflyten din sikker?

Usikker på om Exchange Online-leietakeren din også har en åpen sideinngang? **adeptio** kontrollerer hele e-postflyten din: fra MX-poster, connectors og tredjeparts-gatewayer til EOP, SPF, DKIM, DMARC og Direct Send. Praktisk, uavhengig og med konkrete anbefalinger.

De som ønsker å få kontrollert eller sikret e-postflyten sin ordentlig, kan gjerne avtale en uforpliktende rådgivningssamtale:

**[Bestill en rådgivningssamtale med adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
