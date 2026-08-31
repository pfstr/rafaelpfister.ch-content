---
title: "NDR, DSN, Bounce: Slik skiller du korrekt mellom meldinger om mislykket levering"
navTitle: "NDR og bouncer"
description: "NDR, DSN, bounce, reject, backscatter: Begrepene rundt mislykket levering brukes ofte om hverandre, men betegner ulike ting. Hva RFC-ene definerer, hvem som genererer hvilke meldinger, hvordan en DSN er bygget opp, og hvorfor forskjellen mellom reject og bounce avgjør backscatter."
date: "2026-08-28"
kategorie: "SMTP og e-postflyt"
timeToRead: "10 min. lesetid"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-slik-skiller-du-korrekt-mellom-meldinger-om-mislykket-levering"
translationId: "article-5c5164049a129fa4"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich füge dir gleich eine Unzustellbarkeitsmeldung (NDR/DSN) ein. Analysiere sie Schritt für Schritt: 1. Welcher Server hat die Meldung erzeugt (Reporting-MTA bzw. Generating server)? 2. Wurde die Mail in der SMTP-Session abgewiesen oder nach Annahme zurückgeschickt? 3. Was bedeuten SMTP-Antwortcode und Enhanced Status Code (RFC 3463) konkret? 4. Liegt die Ursache beim Absender, beim Empfänger oder auf dem Transportweg? 5. Welche nächsten Diagnose-Schritte empfiehlst du?
translationOf: ndr-dsn-bounce-unterschiede
url: https://rafaelpfister.ch/no/blog/ndr-dsn-bounce-slik-skiller-du-korrekt-mellom-meldinger-om-mislykket-levering
translationSourceHash: e526de6f4a454b4f4975eac3e8a406ab5b30314c624bf12c69f87bec99fdd0e7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:34:58.041Z
translationReview: automatic
---

# NDR, DSN, Bounce: Slik skiller du korrekt mellom meldinger om mislykket levering

En e-post kommer ikke frem, og i saken står det enten «Bounce», «NDR», «Mailer-Daemon» eller «feilmelding fra serveren». I administrasjonshverdagen brukes disse begrepene om hverandre, selv om de betegner ulike ting: En reject i SMTP-økten er ikke en retur-e-post, en forsinkelsesmelding er ikke en leveringsfeil, og en lesebekreftelse har ingenting med manglende levering å gjøre. Den som skiller begrepene klart, finner årsaken raskere, fordi hver meldingstype sier noe forskjellig om hvor i transportveien problemet ligger, og hvem som kan løse det.

## DSN: overbegrepet fra RFC-ene

Det formelle overbegrepet heter Delivery Status Notification (DSN), definert i RFC 3461 til 3464. En DSN er en maskinelt generert e-post som informerer avsenderen om leveringsstatusen til meldingen. Viktig: En DSN rapporterer ikke bare feil. Feltet `Action` i den maskinlesbare delen har fem verdier:

| Action | Betydning |
|---|---|
| `failed` | Levering har mislyktes endelig; e-posten forsøkes ikke på nytt |
| `delayed` | Leveringen er forsinket; serveren fortsetter å forsøke |
| `delivered` | Levert uten feil (leveringsbekreftelse, kun ved eksplisitt forespørsel) |
| `relayed` | Videresendt til en server som selv ikke genererer DSN-er |
| `expanded` | Overlevert til en distribusjonsliste og splittet opp |

Meldingen om mislykket levering er altså bare et spesialtilfelle: en DSN med `Action: failed`. Microsoft kaller nettopp dette spesialtilfellet en Non-Delivery Report (NDR). Begrepet NDR stammer fra Exchange-verdenen, men brukes nå på tvers av leverandører. Presist formulert: Alle NDR-er er DSN-er, men ikke alle DSN-er er NDR-er.

Forsinkelsesmeldingen (`Action: delayed`) fortjener særlig oppmerksomhet, fordi den regelmessig misforstås som en leveringsfeil i supportarbeid. Et typisk emne er «Delivery delayed» eller «Levering forsinket». E-posten ligger da fortsatt i køen på den sendende serveren, som fortsetter å forsøke, vanligvis i én til to dager. Først når køens levetid utløper, følger den endelige NDR-en. En bruker som sender e-posten på nytt etter en forsinkelsesmelding, oppretter duplikater så snart målsystemet er tilgjengelig igjen.

## Reject eller bounce: det viktigste skillet

Før de øvrige begrepene følger, må det sentrale tekniske veiskillet forklares, siden det avgjør hvilken server som genererer en melding.

**Reject (avvisning i økten):** Den mottakende serveren avviser e-posten allerede under SMTP-økten, med en 5xx-svarkode på `RCPT TO` eller etter `DATA`. Den tar aldri imot e-posten og genererer heller ingen egen returmelding. Plikten til å informere avsenderen ligger hos den innleverende serveren: Den sendende MTA-en ser 5xx-svaret og genererer deretter NDR-en for sin lokale bruker. NDR-en brukeren leser, kommer i dette tilfellet fra egen server, men siterer feilmeldingen fra motparten.

**Bounce (aksept med senere returmelding):** Den mottakende serveren aksepterer e-posten med `250 OK` og fastslår først etterpå at den ikke kan levere den, for eksempel fordi postboksen ikke finnes, kvoten er full eller en underordnet server avviser den. Nå har den ansvaret for meldingen og må selv sende en DSN til avsenderen. Denne påfølgende retur-e-posten er en bounce i snever forstand.

For feilsøking kan forskjellen brukes direkte: Hvis egen server står oppført som genererende system i NDR-en, ble e-posten avvist i økten eller kom aldri ut. Hvis en ekstern server står som avsender av meldingen, har motparten først akseptert e-posten, og problemet ligger bak dens akseptpunkt, usynlig for avsenderen.

To ytterligere bounce-begreper kommer fra markedsføringsmiljøet og står ikke i noen RFC: Hard Bounce for endelige feil (5xx, `Action: failed`) og Soft Bounce for midlertidige feil (4xx, `Action: delayed`). For e-postplattformene er skillet sentralt, fordi Hard Bounces bør føre til umiddelbar opprydding i lister. Teknisk sett er dette de samme mekanismene som ovenfor.

## Begrepene i oversikt

| Begrep | Hva det er | Hvem genererer meldingen | Standard |
|---|---|---|---|
| DSN | Overbegrep: statusmelding om levering (failed, delayed, delivered, relayed, expanded) | MTA-en som har ansvaret for e-posten | RFC 3461 til 3464 |
| NDR | DSN med `Action: failed`; Microsoft-begrep for meldingen om mislykket levering | Sendende MTA (etter reject) eller mottakende MTA (etter aksept) | RFC 3464, Microsoft-dokumentasjon |
| Reject | 5xx-avvisning i en pågående SMTP-økt; ingen egen e-post | Ingen; den sendende MTA-en former en NDR av dette | RFC 5321 |
| Bounce | Retur-e-post etter at e-posten allerede er akseptert | Mottakende MTA | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Markedsføringsinndeling: endelig (5xx) kontra midlertidig (4xx) | som bounce | ingen RFC |
| Forsinkelsesmelding | DSN med `Action: delayed`; e-posten ligger fortsatt i køen | Sendende eller relayende MTA | RFC 3464 |
| Backscatter | NDR-er til forfalskede avsenderadresser, vanligvis utløst av spam | Feilkonfigurerte mottakende MTA-er | ingen RFC, anti-misbruksbegrep |
| MDN / lesebekreftelse | Melding om visning eller sletting hos mottakeren | Mottakerens e-postklient | RFC 8098 |
| Fraværsmelding | Automatisk svar fra en nådd postboks | Postboks- eller groupware-server | RFC 3834 |

## Slik er en DSN bygget opp

Standardkonforme DSN-er bruker MIME-typen `multipart/report; report-type=delivery-status` med tre deler: en menneskelesbar forklaring, en maskinlesbar del av typen `message/delivery-status` og eventuelt originalmeldingen eller headerne dens. Den maskinlesbare delen er den mest verdifulle for diagnostikk, fordi feltene er standardiserte:

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Felt | Betydning |
|---|---|
| `Reporting-MTA` | Serveren som genererte denne DSN-en; første indikasjon på ansvar |
| `Final-Recipient` | Mottakeradressen som statusen gjelder for (én blokk per mottaker) |
| `Action` | En av de fem statusverdiene (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code etter RFC 3463, f.eks. `5.1.1` |
| `Remote-MTA` | Motparten som Reporting-MTA-en kommuniserte med |
| `Diagnostic-Code` | Motpartens ordrette SMTP-svar; ofte den mest informative linjen |

En DSN sendes alltid med tom envelope-avsender (`MAIL FROM:<>`). Dette er ikke en forsømmelse, men et krav i RFC 5321: Den tomme avsenderen hindrer at en DSN for en ikke-leverbar DSN følger, og at to servere sender feilmeldinger til hverandre uten ende. Dette gir en konfigurasjonsregel: Et e-postsystem må ikke avvise e-post med tom envelope-avsender generelt, ellers kommer legitime meldinger om mislykket levering aldri frem til egne brukere.

Exchange og Exchange Online følger standarden for formatet, men pakker innholdet i sin egen presentasjon: Brukeren ser en klargjort side med forklaring i klartekst, etterfulgt av «Generating server» (tilsvarer `Reporting-MTA`) og rådataene. For diagnostikk er det alltid verdt å se på denne nedre, tekniske delen.

## Slik leser du Enhanced Status Codes

I feltet `Status` og vanligvis også i `Diagnostic-Code` står det en tredelt kode etter RFC 3463: klasse.emne.detalj. Klassen angir bindende status, mens emne og detalj angir årsaken:

| Kodeområde | Betydning |
|---|---|
| `2.x.x` | Suksess (kun i leveringsbekreftelser) |
| `4.x.x` | Midlertidig feil; serveren forsøker på nytt |
| `5.x.x` | Endelig feil; ingen flere forsøk |
| `x.1.x` | Adresseringsproblem, f.eks. `5.1.1` ukjent mottaker, `5.1.10` domene uten MX |
| `x.2.x` | Postboksproblem, f.eks. `5.2.2` postboksen er full, `5.2.3` meldingen er for stor for postboksen |
| `x.3.x` | Problem i målsystemet, f.eks. `4.3.2` systemet tar for øyeblikket ikke imot noe |
| `x.4.x` | Nettverk og ruting, f.eks. `4.4.1` intet svar, `4.4.7` køens levetid er utløpt |
| `x.5.x` | Protokollfeil i SMTP-dialogen |
| `x.7.x` | Retningslinjer og sikkerhet, f.eks. `5.7.1` relay nektet eller avvisning på grunn av retningslinjer, `5.7.26` manglende autentisering (SPF/DKIM/DMARC) |

Den klassiske tresifrede SMTP-svarkoden (for eksempel `550`) og Enhanced Status Code står ofte sammen på én linje: `550 5.7.1 ...`. Den tresifrede koden styrer protokollatferden til den sendende serveren, mens den utvidede koden gir den diagnostiske informasjonen. Ved motsetninger mellom kode og fritekst er motpartens fritekst ofte den mer presise kilden, fordi mange systemer setter generiske koder og skriver den faktiske årsaken i kommentaren, inkludert referanse-ID-er for support hos motparten.

Merk: Avvisninger med `5.7.x` fra omdømme- og innholdsfiltre sier ofte bevisst lite. Den som bare ser på koden her, leter på feil sted; blokkeringslisten eller filterprodusenten i friteksten leder raskere til målet.

## Backscatter: den skadelige typen bounce

Backscatter oppstår når en server først aksepterer spam med forfalsket avsender og deretter sender en NDR til den forfalskede adressen. NDR-en treffer dermed en uvedkommende hvis adresse spammeren har misbrukt. Ved store spamkampanjer mottar de berørte tusenvis av NDR-er for e-poster de aldri har sendt, og servere som genererer slike NDR-er i stort omfang, havner selv på blokkeringslister (for eksempel Backscatterer-listen fra UCEPROTECT).

Løsningen følger direkte av skillet mellom reject og bounce: Alt som kan avvises, skal avvises i SMTP-økten, ikke sendes tilbake etter aksept. Konkret betyr det mottakervalidering ved det ytterste akseptpunktet (edge-gatewayen kjenner de gyldige adressene, via katalogoppslag eller Recipient Callout, i stedet for å akseptere alt og la det feile internt), avvisning av spam og skadelig programvare under økten i stedet for karantene-NDR-er, og å avstå fra NDR-er for meldinger klassifisert som spam. En reject skaper ikke backscatter, for med forfalsket avsender mottar spammerens server 5xx-svaret og lager ingen NDR til offeret av det.

## Hva som ikke er en melding om mislykket levering

Tre meldingstyper havner regelmessig i samme kategori i saker, men hører ikke hjemme der:

**MDN (Message Disposition Notification, RFC 8098):** Lesebekreftelsen. Den genereres ikke av transportsystemet, men av mottakerens e-postklient, og rapporterer visning eller sletting av meldingen, ikke levering. MIME-typen heter derfor `multipart/report; report-type=disposition-notification`. En uteblitt lesebekreftelse sier ingenting om leveringen; de fleste klienter spør brukeren eller undertrykker MDN-er helt.

**Fraværsmeldinger og autosvar (RFC 3834):** En fraværsmelding beviser det motsatte av en leveringsfeil, fordi den forutsetter at e-posten har nådd postboksen. I saksbeskrivelser («jeg får et automatisk svar, kommer e-posten min frem?») er det verdt å spørre hvilken melding som faktisk foreligger.

**Karantenevarsler:** Meldinger som karantenesammendraget fra Microsoft 365 eller en gateway informerer mottakeren om tilbakeholdte e-poster. De går til mottakeren, ikke avsenderen, og følger ingen DSN-standard. Avsenderen mottar ofte ingenting i dette scenariet, noe som forklarer tilfellene der en e-post «forsvinner uten feilmelding».

## Sjekkliste for diagnostikk

Hvis en melding foreligger, avklar dette i følgende rekkefølge:

1. Hvilken type er det: NDR (`Action: failed`), forsinkelse (`Action: delayed`), MDN, autosvar eller karantenevarsel? Ved en forsinkelsesmelding: vent, ikke send på nytt.
2. Hvem genererte meldingen (`Reporting-MTA` eller «Generating server»)? Egen server betyr reject eller intern feil, en ekstern server betyr aksept med senere feil hos motparten.
3. Hva sier status- og Diagnostic-koden? Klasse 4 kontra klasse 5 skiller midlertidig fra endelig, emnet (`x.1` adresse, `x.2` postboks, `x.4` nettverk, `x.7` retningslinjer) avgrenser årsaken, og motpartens fritekst gir detaljene.
4. Mangler enhver melding selv om e-posten ikke kommer frem: Kontroller Message Tracking på eget system og tenk på karantene eller stille filtrering hos motparten.

Hvordan individuelle leveringsveier deretter kan gjenskapes målrettet, viser artiklene om [Message Tracking og SMTP-diagnostikk i kommandogeneratoren](https://rafaelpfister.ch/tools/command-builder) samt [Mail Header Analyzer](https://rafaelpfister.ch/tools/mail-header-analyzer) for analyse av transportveien til en mottatt e-post.

## Kilder

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): SMTP-utvidelse som avsendere kan bruke til å be om og styre DSN-er.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definisjon av de tredelte statuskodene (klasse.emne.detalj).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): DSN-ens struktur som multipart/report, felt som Action, Status og Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Grunnregler for svarkoder, ansvarsovergang ved aksept og tom envelope-avsender for feilmeldinger.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard for lesebekreftelser, for å skille dem fra DSN-er.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Regler for autosvar som fraværsmeldinger.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/non-delivery-reports-in-exchange-online/non-delivery-reports-in-exchange-online): NDR-struktur og kodeliste sett fra Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Blokkeringsliste for systemer som genererer backscatter; forklarer kriteriene for oppføring.
