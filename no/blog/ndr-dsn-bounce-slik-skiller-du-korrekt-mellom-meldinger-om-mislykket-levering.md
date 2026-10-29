---
title: "NDR, DSN, Bounce: Slik skiller du feilmeldinger ved manglende levering riktig"
navTitle: "NDR og bounces"
description: "NDR, DSN, bounce, reject, backscatter: Begrepene rundt mislykket levering brukes ofte om hverandre, men betegner ulike ting. Hva RFC-ene definerer, hvem som oppretter hvilken melding, hvordan en DSN er bygget opp, og hvorfor forskjellen mellom reject og bounce avgjør backscatter."
date: "2026-08-28"
kategorie: "SMTP og e-postflyt"
timeToRead: "10 min lesetid"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-slik-skiller-du-korrekt-mellom-meldinger-om-mislykket-levering"
translationId: "article-5c5164049a129fa4"
translationOf: ndr-dsn-bounce-unterschiede
translationSourceHash: 1500e27283dc4c53456810f133d12d81e1f60d7e874b6f3450b0a50fb47d4cf2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:21:59.603Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/ndr-dsn-bounce-slik-skiller-du-korrekt-mellom-meldinger-om-mislykket-levering
---

# NDR, DSN, Bounce: Slik skiller du feilmeldinger ved manglende levering riktig

En e-post kommer ikke frem, og i saken står det enten «bounce», «NDR», «Mailer-Daemon» eller «feilmelding fra serveren». I administratorhverdagen brukes disse begrepene om hverandre, selv om de betegner ulike ting: En reject i SMTP-økten er ikke en retur-e-post, en forsinkelsesmelding er ikke en leveringsfeil, og en lesebekreftelse har overhodet ingenting med manglende levering å gjøre. De som skiller begrepene tydelig, finner årsaken raskere, for hver meldingstype sier noe forskjellig om hvor i transportveien problemet ligger og hvem som kan løse det.

## DSN: det overordnede begrepet fra RFC-ene

Det formelle overordnede begrepet er Delivery Status Notification (DSN), definert i RFC 3461 til 3464. En DSN er en maskinelt generert e-post som informerer avsenderen om leveringsstatusen til meldingen. Viktig: En DSN melder ikke bare om feil. Feltet `Action` i den maskinlesbare delen har fem verdier:

| Action | Betydning |
|---|---|
| `failed` | Levering har mislyktes endelig; e-posten forsøkes ikke på nytt |
| `delayed` | Levering er forsinket; serveren forsøker videre |
| `delivered` | Levert (leveringsbekreftelse, kun ved uttrykkelig forespørsel) |
| `relayed` | Videresendt til en server som selv ikke oppretter DSN-er |
| `expanded` | Overlevert til en distribusjonsliste og utvidet |

Meldingen om manglende levering er altså bare et spesialtilfelle: en DSN med `Action: failed`. Microsoft kaller nettopp dette spesialtilfellet Non-Delivery Report (NDR). Begrepet NDR kommer fra Exchange-verdenen, men er nå i bruk på tvers av leverandører. Presist sagt: Alle NDR-er er DSN-er, men ikke alle DSN-er er NDR-er.

Forsinkelsesmeldingen (`Action: delayed`) fortjener særlig oppmerksomhet fordi den jevnlig misforstås som en leveringsfeil i support. Et typisk emne er «Delivery delayed» eller «Levering forsinket». E-posten ligger da fortsatt i køen på den sendende serveren, som fortsetter å prøve, vanligvis i én til to dager. Først når køens levetid utløper, følger den endelige NDR-en. En bruker som sender e-posten på nytt etter en forsinkelsesmelding, oppretter duplikater så snart målsystemet er tilgjengelig igjen.

## Reject eller bounce: det viktigste skillet

Før de øvrige begrepene følger, bør det sentrale tekniske veivalget forklares, for det avgjør hvilken server som oppretter en melding.

**Reject (avvisning i økten):** Den mottakende serveren avviser e-posten allerede under SMTP-økten, med en 5xx-svarkode på `RCPT TO` eller etter `DATA`. Den tar aldri imot e-posten og oppretter heller ingen egen returmelding. Plikten til å informere avsenderen ligger hos den innleverende serveren: Den sendende MTA-en ser 5xx-svaret og oppretter deretter NDR-en for sin lokale bruker. NDR-en brukeren leser, kommer i dette tilfellet fra egen server, men siterer feilmeldingen fra motparten.

**Bounce (aksept med senere retur):** Den mottakende serveren godtar e-posten med `250 OK` og oppdager først etterpå at den ikke kan levere den, for eksempel fordi postboksen ikke finnes, kvoten er full eller en etterfølgende server avviser. Nå har den ansvaret for meldingen og må selv sende en DSN til avsenderen. Denne etterfølgende retur-e-posten er bounce i snever betydning.

For feilsøking kan forskjellen brukes direkte: Hvis egen server står oppført som systemet som opprettet NDR-en, ble e-posten avvist i økten eller kom aldri ut. Hvis en fremmed server er avsender av meldingen, godtok motparten først e-posten, og problemet ligger bak deres akseptpunkt, usynlig for avsenderen.

Fra markedsføringsmiljøet kommer to ytterligere bounce-begreper som ikke står i noen RFC: Hard bounce for endelige feil (5xx, `Action: failed`) og soft bounce for midlertidige feil (4xx, `Action: delayed`). For e-postplattformer er skillet sentralt, fordi hard bounces bør føre til umiddelbar opprydding i listen. Teknisk sett er det de samme mekanismene som ovenfor.

## Begrepene i oversikt

| Begrep | Hva det er | Hvem oppretter meldingen | Standard |
|---|---|---|---|
| DSN | Overordnet begrep: statusmelding om levering (failed, delayed, delivered, relayed, expanded) | MTA-en som har ansvaret for e-posten | RFC 3461 til 3464 |
| NDR | DSN med `Action: failed`; Microsoft-begrep for melding om manglende levering | Sendende MTA (etter reject) eller mottakende MTA (etter aksept) | RFC 3464, Microsoft-dokumentasjon |
| Reject | 5xx-avvisning i den pågående SMTP-økten; ingen egen e-post | Ingen; den sendende MTA-en gjør dette om til en NDR | RFC 5321 |
| Bounce | Retur-e-post etter at e-posten allerede er akseptert | Mottakende MTA | RFC 5321, RFC 3464 |
| Hard/soft bounce | Markedsføringsinndeling: endelig (5xx) kontra midlertidig (4xx) | som bounce | ingen RFC |
| Forsinkelsesmelding | DSN med `Action: delayed`; e-posten ligger fortsatt i køen | Sendende eller videresendende MTA | RFC 3464 |
| Backscatter | NDR-er til forfalskede avsenderadresser, som oftest utløst av spam | Feilkonfigurerte mottakende MTA-er | ingen RFC, anti-misbruksbegrep |
| MDN / lesebekreftelse | Melding om visning eller sletting hos mottakeren | Mottakerens e-postklient | RFC 8098 |
| Fraværsmelding | Automatisk svar fra en nådd postboks | Postboks- eller groupware-server | RFC 3834 |

## Slik er en DSN bygget opp

Standardkonforme DSN-er bruker MIME-typen `multipart/report; report-type=delivery-status` med tre deler: en lesbar forklaring, en maskinlesbar del av typen `message/delivery-status` og valgfritt den opprinnelige meldingen eller dens hoder. Den maskinlesbare delen er den mest verdifulle for diagnose, fordi feltene er standardiserte:

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
| `Reporting-MTA` | Serveren som har opprettet denne DSN-en; første indikasjon på ansvar |
| `Final-Recipient` | Mottakeradressen statusen gjelder (én blokk per mottaker) |
| `Action` | Én av de fem statusverdiene (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code i henhold til RFC 3463, f.eks. `5.1.1` |
| `Remote-MTA` | Motparten Reporting-MTA-en kommuniserte med |
| `Diagnostic-Code` | Det ordrette SMTP-svaret fra motparten; ofte den mest informative linjen |

En DSN sendes alltid med tom envelope-avsender (`MAIL FROM:<>`). Dette er ikke skjødesløshet, men et krav i RFC 5321: Den tomme avsenderen hindrer at en DSN for en DSN som ikke kan leveres, opprettes, slik at to servere sender feilmeldinger til hverandre i det uendelige. Dette gir en konfigurasjonsregel: Et e-postsystem må ikke avvise e-poster med tom envelope-avsender generelt, ellers når legitime meldinger om manglende levering aldri egne brukere.

Exchange og Exchange Online følger standarden for formatet, men pakker innholdet i sin egen presentasjon: Brukeren ser en tilrettelagt side med forklaring i klartekst, under den står «Generating server» (tilsvarer `Reporting-MTA`) og råopplysningene. For diagnose lønner det seg alltid å se på denne nedre, tekniske delen.

## Slik leser du Enhanced Status Codes

I feltet `Status` og som regel også i `Diagnostic-Code` står en tredelt kode i henhold til RFC 3463: klasse.emne.detalj. Klassen angir endeligheten, emne og detalj angir årsaken:

| Kodeområde | Betydning |
|---|---|
| `2.x.x` | Suksess (kun i leveringsbekreftelser) |
| `4.x.x` | Midlertidig feil; serveren prøver på nytt |
| `5.x.x` | Endelig feil; ingen flere forsøk |
| `x.1.x` | Adresseringsproblem, f.eks. `5.1.1` ukjent mottaker, `5.1.10` domene uten MX |
| `x.2.x` | Postboksproblem, f.eks. `5.2.2` postboks full, `5.2.3` melding for stor for postboksen |
| `x.3.x` | Problem i målsystemet, f.eks. `4.3.2` systemet aksepterer ingenting akkurat nå |
| `x.4.x` | Nettverk og ruting, f.eks. `4.4.1` intet svar, `4.4.7` køens levetid utløpt |
| `x.5.x` | Protokollfeil i SMTP-dialogen |
| `x.7.x` | Retningslinje og sikkerhet, f.eks. `5.7.1` videresending nektet eller avvisning på grunn av retningslinje, `5.7.26` manglende autentisering (SPF/DKIM/DMARC) |

Den klassiske tresifrede SMTP-svarkoden (for eksempel `550`) og Enhanced Status Code står ofte sammen på én linje: `550 5.7.1 ...`. Den tresifrede koden styrer protokollatferden til den sendende serveren, mens den utvidede koden gir diagnoseinformasjonen. Ved motsetninger mellom kode og fritekst er friteksten fra motparten ofte den mer presise kilden, fordi mange systemer setter generiske koder og skriver den egentlige årsaken i kommentaren, inkludert referanse-ID-er for support hos motparten.

Merk: `5.7.x`-avvisninger fra omdømme- og innholdsfiltre sier ofte bevisst lite. Den som bare ser på koden her, leter på feil sted; blokklisten eller filterprodusenten i friteksten fører raskere til målet.

## Backscatter: den skadelige typen bounce

Backscatter oppstår når en server først godtar spam med forfalsket avsender og deretter sender en NDR til den forfalskede adressen. NDR-en treffer dermed en utenforstående hvis adresse er misbrukt av spammeren. Ved store spam-bølger mottar de berørte tusenvis av NDR-er for e-poster de aldri har sendt, og servere som genererer slike NDR-er i stort omfang, havner selv på blokklister (for eksempel Backscatterer-listen fra UCEPROTECT).

Løsningen følger direkte av skillet mellom reject og bounce: Alt som kan avvises, bør avvises i SMTP-økten, ikke sendes tilbake etter aksept. Konkret betyr dette mottakervalidiering ved det ytterste akseptpunktet (edge-gatewayen kjenner de gyldige adressene, via katalogoppslag eller Recipient Callout, i stedet for å godta alt og la det feile internt), spam- og malware-avvisning under økten i stedet for NDR-er ved karantene, og å unnlate å sende NDR-er for meldinger klassifisert som spam. En reject skaper ikke backscatter, for ved forfalsket avsender havner 5xx-svaret hos spammerens server, som ikke lager noen NDR til offeret av dette.

## Hva som ikke er en melding om manglende levering

Tre meldingstyper havner regelmessig i samme kategori i saker, men hører ikke hjemme der:

**MDN (Message Disposition Notification, RFC 8098):** Lesebekreftelsen. Den opprettes ikke av transportsystemet, men av mottakerens e-postklient, og melder om visning eller sletting av meldingen, ikke om leveringen. MIME-typen heter følgelig `multipart/report; report-type=disposition-notification`. En uteblitt lesebekreftelse sier ingenting om leveringen; de fleste klienter spør brukeren eller undertrykker MDN-er helt.

**Fraværsmeldinger og autosvar (RFC 3834):** En fraværsmelding beviser det motsatte av en leveringsfeil, fordi den forutsetter at e-posten har nådd postboksen. I saksbeskrivelser («jeg får et automatisk svar, kommer e-posten min frem?») lønner det seg å spørre hvilken melding som faktisk foreligger.

**Karantenevarsler:** Meldinger som karanteneoversikten fra Microsoft 365 eller en gateway informerer mottakeren om tilbakeholdte e-poster. De går til mottakeren, ikke til avsenderen, og følger ingen DSN-standard. Avsenderen mottar ofte ingenting i dette scenariet, noe som forklarer tilfellene der en e-post «forsvinner uten feilmelding».

## Sjekkliste for diagnose

Når en melding foreligger, avklar dette i følgende rekkefølge:

1. Hvilken type er det: NDR (`Action: failed`), forsinkelse (`Action: delayed`), MDN, autosvar eller karantenevarsel? Ved en forsinkelsesmelding: vent, ikke send på nytt.
2. Hvem opprettet meldingen (`Reporting-MTA` eller «Generating server»)? Egen server betyr reject eller intern feil, en fremmed server betyr aksept med senere feil hos motparten.
3. Hva sier status- og diagnostic-koden? Klasse 4 kontra klasse 5 skiller midlertidig fra endelig, emnet (`x.1` adresse, `x.2` postboks, `x.4` nettverk, `x.7` retningslinje) avgrenser årsaken, mens friteksten fra motparten gir detaljene.
4. Mangler enhver melding selv om e-posten ikke kommer frem: Kontroller Message Tracking i eget system, og tenk på karantene eller stille filtrering hos motparten.

Hvordan enkelte leveringsveier kan etterprøves målrettet, viser artiklene om [Message Tracking og SMTP-diagnose i kommando-generatoren](https://rafaelpfister.ch/tools/command-builder) samt [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer) for analyse av transportveien til en e-post som har kommet frem.

## Endringslogg

- **25. oktober 2026:** Rettet: Lenken til Mail Header Analyzer ble satt til riktig adresse. Kildelenken ble oppdatert til den nye adressen hos Microsoft Learn.

## Kilder

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): SMTP-utvidelse som lar avsendere be om og styre DSN-er.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definisjon av de tredelte statuskodene (klasse.emne.detalj).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): Oppbygning av DSN som multipart/report, felt som Action, Status og Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Grunnregler for svarkoder, ansvarsovergang ved aksept og tom envelope-avsender for feilmeldinger.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard for lesebekreftelser, for å skille dem fra DSN-er.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Regler for autosvar som fraværsmeldinger.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/non-delivery-reports-in-exchange-online): NDR-oppbygning og kodeliste sett fra Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Blokkliste for systemer som oppretter backscatter; forklarer kriteriene for oppføring.
