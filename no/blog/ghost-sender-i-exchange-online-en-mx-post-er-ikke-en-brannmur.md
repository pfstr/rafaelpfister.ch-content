---
title: "Ghost Sender i Exchange Online: En MX-record er ikke en brannmur"
navTitle: "Ghost Sender"
description: "Direkte levering til Exchange Online omgår en foranstilt gateway dersom tenant-en ikke uttrykkelig blokkerer den. Risikoen er reell, og årsaken er en ufullstendig konfigurert e-postflyt."
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
translationSourceHash: da918db49e8e0b5243e6a3c354324682ddb6bda54ec743be474f3011df3f9dc6
translatedAt: 2026-10-14T08:56:53.660Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/no/blog/ghost-sender-i-exchange-online-en-mx-post-er-ikke-en-brannmur
---

# Ghost Sender i Exchange Online: En MX-record er ikke en brannmur

![En Ghost-admin holder døren ved siden av sikkerhetsporten åpen i datasenteret, mens e-poster passerer filteret og kommer direkte inn i postkassen.](../images/ghost-admin.png)

Angrepsmuligheten som InfoGuard Labs beskriver som «Ghost Sender», er reell: En angriper kan omgå en foranstilt e-postgateway og levere direkte til Exchange Online. Forutsetningen er imidlertid at tenant-en fortsatt aksepterer denne direkte veien. Dette er ikke en universell sårbarhet i Exchange Online, men en ufullstendig sikret e-postflyttopologi.

En Mail Transfer Agent som betjener postkasser for et domene, tar i utgangspunktet imot SMTP-forbindelser fra internett. MX-recorden viser vanlige avsendere den ønskede leveringsveien. Den er verken en brannmurregel eller en tilgangsliste, og hindrer ingen i å kontakte et kjent Exchange Online-endepunkt direkte.

## Hva «Ghost Sender» faktisk viser

Scenariet som [InfoGuard Labs beskriver](https://labs.infoguard.ch/posts/ghost-sender/), ser slik ut:

1. En organisasjon har postkassene sine i Exchange Online.
2. Den offentlige MX-recorden peker til en foranstilt Secure Email Gateway.
3. Exchange Online-endepunktet på `*.mail.protection.outlook.com` er fortsatt direkte tilgjengelig fra internett.
4. Administratoren har ikke begrenset Exchange Online slik at bare den foranstilte gatewayen kan levere der.
5. En angriper ignorerer MX-recorden og leverer meldingen direkte til Exchange Online.

Den tiltenkte veien er dermed:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Denne veien er imidlertid fortsatt åpen:

```text
Angreifer -> Exchange Online -> Postfach
```

Dette er en alvorlig feilkonfigurasjon. Det foranstilte filteret kan omgås på denne veien; forfalskede avsendere, phishing og CEO-svindel blir dermed betydelig enklere. InfoGuard fortjener anerkjennelse for å ha synliggjort problemet, undersøkt utbredelsen og publisert en enkel test.

Men hvor er egentlig produktfeilen her?

Også medienes spissing hjelper lite med å sette dette i perspektiv. [Heise har overskriften om at Exchange Online slipper forfalskede e-poster «uten videre gjennom»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), selv om bare bestemte, ufullstendig herdede tredjeparts- og hybridkonfigurasjoner er berørt. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) formulerer det langt mer presist: ikke et sikkerhetshull i snever forstand, men et design- og konfigurasjonsproblem.

## «An MTA is doing MTA-Things»

Hver Exchange Online-tenant har et offentlig SMTP-endepunkt. Dette endepunktet er ingen hemmelighet, og det skal heller ikke være det. Microsoft forklarer selv at Exchange Online som standard tar imot meldinger som er adressert direkte til postkasser som driftes der: [Dette er rett og slett slik e-post fungerer](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Også [SMTP selv beskriver MX-recorden som en mekanisme for å finne det ordinære målsystemet](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Dette innebærer ingen plikt for målserveren til å avvise forbindelser via enhver annen tilgjengelig vert. En angriper må ikke følge den skilte veien. Hvis en annen MTA er tilgjengelig, kjenner mottakerdomenet og aksepterer meldingen, vil den bli prøvd – på samme måte som spammere i flere tiår har forsøkt å kontakte dårligere beskyttede backup-MX-systemer.

Den som setter et tredjepartsfilter foran, endrer standardtopologien. «Exchange Online er min internett-e-postgateway» blir til «bare tredjepartsgatewayen min kan overføre internett-e-post til Exchange Online». Denne nye `Trust-Border` oppstår ikke gjennom en DNS-oppføring. Den må håndheves uttrykkelig på det mottakende systemet.

Microsoft dokumenterer nettopp dette: Ved en ekstern MX skal det opprettes en Inbound Connector av typen `Partner`, som for `SenderDomains *` bare godtar sertifikatet eller kilde-IP-adressene til den foranstilte tjenesten. Meldinger som leveres direkte forbi gatewayen, blir da avvist. Dette står ordrett i Microsofts veiledning [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius beskriver også denne «sideinngangen» inngående i [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM og DMARC er ikke dørvakter

InfoGuard viser meldinger der SPF, DKIM og DMARC feiler, men som likevel havner i postkassen. Det ser spektakulært ut, men er ingen kryptografisk «bypass» av disse mekanismene. E-postene kommer nettopp ikke vellykket gjennom. De leverer `fail`. Det avgjørende er hvilken lokal handling det mottakende systemet utleder av dette resultatet.

SPF kontrollerer om et system har lov til å sende for konvoluttavsenderen. DKIM kontrollerer en signatur. DMARC kobler disse resultatene til det synlige avsenderdomenet og publiserer ønsket behandling. Selv den gjeldende [DMARC-standarden RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) fastslår uttrykkelig at mottakeren kan ta hensyn til denne ønskede behandlingen, men ikke er forpliktet til det. DMARC er et viktig signal, men ingen nettverksbasert tilgangskontroll.

Ved en foranstilt gateway kommer det i tillegg at Exchange Online først ser IP-adressen til denne gatewayen, ikke den opprinnelige avsenderens. Til dette finnes [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Den rekonstruerer den opprinnelige kilden og forbedrer SPF-, DKIM-, DMARC-, anti-spoofing- og anti-phishing-vurderinger. Enhanced Filtering er imidlertid heller ingen dørlås. Den erstatter ikke den restriktive Partner Connector.

Feilkonfigurasjonen blir særlig tydelig når en administrator svekker eller helt omgår EOP-kontrollen via SCL-bypass fordi det foranstilte produktet allerede skal filtrere, samtidig som direkte internettlevering står åpen. Da har vedkommende ikke fått en beskyttelsesmekanisme «omgått», men bevisst unnlatt å sørge for effektiv beskyttelse ved én av to innganger.

Man kan absolutt kritisere Microsoft dersom en melding med tydelig synlig autentiseringsfeil havner i innboksen uten advarsel. Man kan kritisere semantikken i connector-typene, dokumentasjonen og manglende varsler i Configuration Analyzer. Alt dette er legitime punkter. Men eksistensen av et offentlig tilgjengelig SMTP-endepunkt er ingen sikkerhetssårbarhet.

## «Direct Send» er ikke det samme som «direkte levering»

To ting blandes sammen i diskusjonen:

- **Direct Send** hos Microsoft betegner anonyme meldinger der konvoluttavsenderen (`5321.MailFrom`) bruker et eget Accepted Domain for tenant-en.
- **Direkte levering til Exchange Online** betegner generelt en SMTP-melding som ignorerer den publiserte tredjeparts-MX-en og leveres direkte til Exchange-endepunktet. Avsenderen kan da også bruke et vilkårlig eksternt domene.

For Direct Send finnes det en egen innstilling:

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-RejectDirectSend $true` | Avviser anonyme direkteleveringer der konvoluttavsenderen bruker et Accepted Domain for tenant-en |

</details>

Innstillingen er fornuftig dersom Direct Send ikke behøves. Den forhindrer spoofing av interne domener via denne veien. Den stenger imidlertid ikke hele sideinngangen for vilkårlige eksterne avsendere. Microsoft beskriver det nøyaktige virkeområdet i [cmdlet-dokumentasjonen for `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Den som vil forhindre «Ghost Sender» fullstendig, trenger fortsatt tilgangsbegrensning via Partner Connector eller en passende e-postflytregel.

## Må Microsoft virkelig gjøre alt for administratoren?

Nei. Den som legger et ekstra e-postfilter inn i en produksjonstransportkjede, overtar ansvaret for denne transportkjeden.

Leverandøren kan ikke pålitelig gjette om det i tillegg til den eksterne MX-en finnes skannere, multifunksjonsenheter, SaaS-tjenester, hybridservere, partnerreléer eller andre legitime systemer som må sende direkte til Exchange Online. En automatisk «MX-en peker et annet sted, så blokker alt annet» ville avbryte ønskede e-postflyter i en rekke reelle miljøer. Derfor må administratoren definere den ønskede tillitsgrensen eksplisitt.

Likevel bør Microsoft gjøre det enklere for de ansvarlige. En god Configuration Analyzer bør oppdage en ekstern MX uten restriktiv Partner Connector og varsle tydelig. Oppsettsdialogen kunne forklare at en connector av typen «Din organisasjon» riktignok identifiserer passende forbindelser, men ikke automatisk avviser upassende forbindelser. Secure-by-default-innstillinger og bedre driftsrapporter ville også være velkomne.

Dette ville være fornuftig produktherding. Det endrer imidlertid ikke den tekniske vurderingen: En usikker spesialtopologi forblir en usikker konfigurasjon og blir ikke en zero-day bare fordi den er utbredt.

## Slik stenges sideinngangen

For miljøer med et foranstilt filter bør minst disse punktene stå på sjekklisten:

1. **Dokumenter e-postflyten fullstendig.** Hvilke systemer har faktisk lov til å levere til Exchange Online? Dette omfatter også hybrid-, applikasjons- og nødveier.
2. **Opprett en restriktiv Partner Connector.** Bruk `SenderDomains *` og begrens leveringen til et sertifikat (foretrukket) eller vedlikeholdte kilde-IP-områder. En connector av typen `OnPremises` eller «Din organisasjon» håndhever ikke denne default-deny-effekten (se for eksempel også: [E-postruting mellom Apache James og Exchange Online](/blog/totemomail-m365)).
3. **Konfigurer Enhanced Filtering riktig.** Dersom EOP fortsatt skal filtrere, må opprinnelig IP og avsenderinformasjon rekonstrueres korrekt. Generelle SCL-`-1`-bypasser må vurderes kritisk.
4. **Deaktiver Direct Send dersom det ikke brukes.** Kontroller først med Message Trace eller tilgjengelige rapporter om skannere eller applikasjoner avhenger av det.
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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-Name` | Visningsnavn for den nye Inbound Connector |
| `-ConnectorType Partner` | Connector-klasse for eksterne partnersystemer; bare denne typen håndhever avvisning av upassende forbindelser |
| `-SenderDomains *` | Connectoren gjelder for e-post fra alle avsenderdomener |
| `-RestrictDomainsToIPAddresses $true` | Aktiverer sperren: E-post for de angitte domenene godtas nå bare fra adressene i `-SenderIpAddresses` |
| `-SenderIpAddresses` | Tillatte kilde-IP-adresser eller -områder for den foranstilte gatewayen |
| `-RequireTls $true` | Krever TLS-kryptering for forbindelser via denne connectoren |

</details>

Der det er mulig, bør sertifikatbinding foretrekkes fremfor en IP-tillatelsesliste. Endringer bør først gjøres i en kontrollert test, for en feilaktig tillatelsesliste gjør svært raskt den åpne sideinngangen til et fullstendig e-postutfall.

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-SmtpServer` | Målvert: tenant-ens offentlige Exchange Online-endepunkt, bevisst utenom MX-en |
| `-To` | Mottakeradresse i tenant-en som testes |
| `-From` | Vilkårlig ekstern avsenderadresse; det er nettopp dette sideinngangen ikke lenger skal akseptere |
| `-Subject` | Emnelinje, for å kunne finne den igjen i Message Trace |
| `-Body` | Meldingstekst |

</details>

Med en korrekt begrenset Partner Connector forventes en SMTP-avvisning som `5.7.51 TenantInboundAttribution; Rejecting`. En alternativ transportregel kan først godta meldingen og deretter flytte den til karantene; derfor må Message Trace, karantene og postkasse kontrolleres i tillegg til SMTP-svaret. `Send-MailMessage` (deprecated) brukes her bare som en lett forståelig illustrasjon. Ethvert kontrollert SMTP-testverktøy tjener samme formål.

## En nyttig test med en misvisende etikett

«Ghost Sender» er ikke en ny SMTP-utnyttelse. Det er et treffende navn på en åpen sideinngang som Microsoft lenge har dokumentert sikringen av, og som administratoren har latt stå åpen.

Det ironiske er at InfoGuard i sitt eget bidrag selv omtaler problemet som «widespread and systematic misconfiguration» og avslutter med setningen «Ghost-Sender is a misconfiguration». Microsofts Security Response Center klassifiserte også først rapporten som ikke en sikkerhetssårbarhet. Faktaene finnes altså i artikkelen: Bare tittelen, test-e-posten og «Vulnerability»-merkingen antyder dessverre en mer dramatisk tolkning.

Den nyttige delen av publiseringen er vekkeropet: Mange selskaper har tilsynelatende ikke låst e-postflyten sin ordentlig. Den problematiske delen er påstanden om at Exchange Online har en universell sikkerhetssårbarhet. Nei: Exchange Online oppfører seg her først og fremst som en MTA. Det blir usikkert gjennom en tillitsgrense som ikke er ferdig konfigurert.

Må man virkelig gjøre alt for administratoren? Nei. Men man må tydeligvis stadig minne om at DNS-ruting ikke erstatter tilgangskontroll.

## Kilder

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): Den opprinnelige undersøkelsen, inkludert utbredelsesanalyse og konklusjonen de selv trekker: «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Nettbasert test publisert av InfoGuard for å undersøke egen tenant for den åpne sideinngangen.

3.  [MSXFAQ: Exchange Online som sideinngang for e-postmottak](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): Frank Carius' vurdering: ingen feil i Exchange Online, men en feilkonfigurasjon hos administratoren.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft forklarer at direkte mottak av e-post til driftede postkasser er slik e-post fungerer, og avgrenser Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Den offisielle veiledningen med eget trinn for restriktiv Partner Connector ved ekstern MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Rekonstruerer den opprinnelige avsenderkilden bak en gateway, forbedrer vurderingen, men erstatter ikke connectoren.

7.  [Heise: Ghost-Sender – Exchange Online slipper forfalskede e-poster uten videre gjennom](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Eksempel på spisset rapportering som generaliserer bestemte feilkonfigurasjoner.

8.  [Crow in the Cloud: The ghosts I did not call](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Treffende vurdering som et design- og konfigurasjonsproblem, inkludert beskyttelsestiltak.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Beskriver MX-recorden som en mekanisme for å finne det ordinære målsystemet, ikke som tilgangskontroll.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Fastslår at mottakeren kan ta hensyn til den publiserte DMARC-behandlingen, men ikke må.

---

## Er e-postflyten din sikker?

Usikker på om Exchange Online-tenant-en din også har en åpen sideinngang? **adeptio** kontrollerer hele e-postflyten din: fra MX-records, connectors og tredjepartsgatewayer til EOP, SPF, DKIM, DMARC og Direct Send. Praktisk, uavhengig og med konkrete anbefalinger.

Den som ønsker å få kontrollert eller sikret e-postflyten sin på en god måte, kan gjerne avtale en uforpliktende rådgivningssamtale:

**[Bestill en rådgivningssamtale med adeptio](/kontakt)**  
[adeptio.ch](https://adeptio.ch/)
