---
title: "Downdetector under kritikk: Hva den amerikanske uken med driftsavbrudd viser om feilmeldinger"
navTitle: "Kritikk av Downdetector"
description: "Mellom 13. og 17. november 2026 registrerte Downdetector i USA flere driftsavbrudd enn på ett år. Avbruddene hos FAA, AWS, Chase, United og Apple var reelle, men hadde separate årsaker og ble som regel løst i løpet av timer. Tallene fra Downdetector forsterket inntrykket av en sammenhengende krise. Hva plattformen måler, hva kritikken går ut på, og hvordan administratorer bedre kan vurdere driftsavbrudd."
date: "2026-11-18"
kategorie: "Drift"
timeToRead: "7 min lesetid"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-under-kritikk-hva-den-amerikanske-uken-med-driftsavbrudd-viser-om-feilmeldinger"
translationId: article-0761d100437e6b25
translationOf: downdetector-kritik-ausfallwoche
url: https://rafaelpfister.ch/no/blog/downdetector-under-kritikk-hva-den-amerikanske-uken-med-driftsavbrudd-viser-om-feilmeldinger
translationSourceHash: 3f7c9f1263f57a9ba194011c81d634786e31250e328f8cc02bb45a26564c46d8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T08:46:22.540Z
translationReview: required
---

# Downdetector under kritikk: Hva den amerikanske uken med driftsavbrudd viser om feilmeldinger

Fra 13. til 17. november 2026 sviktet flere store systemer i USA: luftfartsmyndigheten FAA med et landsdekkende startforbud, deler av AWS i regionen us-east-1, banken JPMorgan Chase, bakketjenestene til United Airlines og tirsdag iCloud-påloggingen samt andre tjenester fra Apple. Jeg fulgte serien i en [liveoppdatering](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11). Etter at uken er over, er det klart: Avbruddene hadde separate årsaker, det finnes ingen tegn på et cyberangrep, og Europa ble bare marginalt berørt. Likevel oppstod det offentlig et bilde av et internett som sviktet samtidig overalt. En stor del av dette bildet stammer fra diagrammene til Downdetector, som ble sitert i nesten alle medieoppslag. Siden mandag har dette møtt bred kritikk.

## Hva Downdetector måler

Downdetector tilhører Ookla og dermed Accenture siden 17. juni 2026. Plattformen samler inn problemrapporter fra brukere som melder et problem på en tjenestes side, og supplerer ifølge sin egen beskrivelse med signaler fra sosiale medier og andre nettbaserte kilder. For hver tjeneste beregner Downdetector en grunnlinje: det gjennomsnittlige antallet rapporter på samme tidspunkt av dagen gjennom det siste året. Ligger det aktuelle tallet klart over dette, regnes det som et driftsavbrudd. Downdetector offentliggjør ikke hvordan kildene vektes, eller ved hvilken faktor en topp teller som et driftsavbrudd.

Dette gir tre egenskaper som ofte forsvinner i dekningen:

1.  **Rapporter er ikke det samme som berørte brukere.** Et tall som «20 000 rapporter» sier ingenting om hvor mange kunder som faktisk ikke hadde tilgang. Det teller personer som la merke til et problem og besøkte Downdetector.

2.  **Brukerne rapporterer tjenesten de ser.** Når et betalingsnettverk, en skyleverandør eller en identitetstjeneste svikter, dukker rapportene opp hos bankene, appene og nettbutikkene som er avhengige av dem. Årsaken er ikke synlig i diagrammet.

3.  **Grunnlinjen gjenspeiler ikke uker med ekstraordinære hendelser.** Når mediene fortløpende rapporterer om driftsavbrudd, sjekker flere mennesker ved enhver forsinkelse om en tjeneste har problemer. Hver ny overskrift øker antallet rapporter, også for tjenester som fungerer normalt.

## Tallene for uken

Ifølge Axios mottok Downdetector nesten 6 millioner problemrapporter fra USA i de sju dagene frem til 16. november, den høyeste ukesverdien på ett år. Den samme analysen viser imidlertid også at det over 30 dager var tre store avbrudd, mens fem er vanlig. Rekordtallet for rapporter står dermed opp mot et heller under gjennomsnittlig antall faktiske store driftsavbrudd.

Flere tilfeller fra uken viser hvor stor avstanden kan være mellom rapporter og den faktiske situasjonen:

- **Banker 16. november:** Medier skrev med henvisning til Downdetector at brukere rapporterte problemer hos «nesten alle store amerikanske banker», deriblant Chase, Wells Fargo og Capital One. Chase sa selv at transaksjoner tok lengre tid, mens kundene fortsatt hadde tilgang til kontoer og saldoer. Ifølge New York Times lå årsaken i en programvarefeil i JPMorgan Chase-maskinvare, som kan ha påvirket andre banker via felles transaksjoner. De andre bankene har ikke bekreftet egne driftsavbrudd.

- **Gmail 16. og 17. november:** Downdetector viste over 2 800 rapporter, først og fremst om e-poster som ikke ble levert. Google opplyste at det ikke fantes tegn på et omfattende driftsavbrudd, og Workspace Status Dashboard hadde ingen oppføring.

- **Charles Schwab 17. november:** Ifølge Wall Street Journal steg antallet rapporter rundt klokken 9.50 lokal tid og falt igjen en halv time senere. Det finnes ingen driftsmelding fra Schwab om dette.

- **Exchange Online 12. november:** Reuters oppga over 97 000 rapporter for samme driftsavbrudd, mens en lokal TV-stasjon oppga nesten 119 000, begge på forskjellige tidspunkter. Ifølge Downdetector begynte rapportene først å stige rundt 68 minutter etter starten som Microsoft oppgir i Admin Center.

Avbruddene hos FAA, AWS, Chase, United og Apple var reelle og bekreftet av operatørene. Det overdrevne var sammenkoblingen: Fem uavhengige driftsavbrudd og en generell økning i rapporter ble til fortellingen om et koordinert angrep. Det hvite hus' talsperson sa 17. november at det ikke var noen krise amerikanere trengte å bekymre seg for. Liz Fong-Jones (Honeycomb) sa til Axios at hendelsene hadde svært lite til felles.

## Hvor kritikken retter seg

Kritikken kommer fra flere hold. Operatører som Google motsier offentlig Downdetector når tjenesten viser et driftsavbrudd de ikke ser i egne data. Fagmedier påpeker at Downdetector teller rapporter i stedet for berørte brukere, viser driftsavbrudd forsinket og ikke fanger opp feil i komponenter som sluttkundene ikke ser. I diskusjoner blant utviklere, for eksempel på Hacker News, sirkulerer dessuten antakelsen om at bare det å åpne en side som «Har Chase problemer?» teller som en implisitt rapport, slik at topper kan overføres fra én tjeneste til lignende tjenester. Downdetector har ikke bekreftet denne antakelsen; det eneste som er dokumentert, er at plattformen i tillegg til aktive rapporter evaluerer andre nettbaserte kilder og ikke opplyser om vektingen av dem.

Forsvarerne av plattformen innvender at Downdetector ofte viser driftsavbrudd tidligere enn leverandørenes statussider. Under Apple-avbruddet 17. november viste Apples statusside fortsatt normal tilgjengelighet da rapportene allerede steg kraftig. Som tidlig varsling er tjenesten nyttig. Som mål på omfanget av et driftsavbrudd er den det ikke, og det var nettopp slik den ble brukt denne uken: Overskrifter som «Tusenvis rapporterer problemer» stod uten kontekst ved siden av bekreftede driftsavbrudd.

## Vurder driftsavbrudd bedre

For administratorer som ved et rapportert driftsavbrudd må avgjøre om de skal informere brukere eller sette i verk tiltak, følger en enkel rekkefølge:

1.  **Kontroller leverandørens statusside og administratorportal.** For Microsoft 365 er dette Service Health i Admin Center, ikke den offentlige statussiden. AWS, Google Cloud, Cloudflare og Apple har egne statussider med tidsstempler.

2.  **Bruk egne målinger.** En enkel syntetisk test (pålogging, sending av e-post, henting av en side) fra eget nettverk viser om organisasjonen selv er berørt. Min [M365-tilkoblingsmonitor](https://rafaelpfister.ch/liveticker) kontrollerer hvert femte minutt tilgjengeligheten til Microsoft 365-endepunktene fra Europa.

3.  **Les Downdetector kun som en indikasjon.** En topp er en grunn til å undersøke nærmere. Det absolutte tallet og topper hos flere lignende tjenester samtidig sier lite om årsak og omfang.

4.  **Ta hensyn til regional opprinnelse.** De fleste tallene i medieoppslag gjelder den amerikanske siden av Downdetector. For brukere i Sveits er de bare relevante dersom leverandøren også bekrefter driftsavbruddet for europeiske regioner.

## Liveoppdateringen

Jeg avsluttet liveoppdateringen om serien av driftsavbrudd med denne artikkelen. Den forblir på nett som en kronikk over uken, med en avsluttende vurdering av de enkelte årsakene og opplysningen om at Europa, med unntak av transatlantiske flyvninger og det verdensomspennende iCloud-avbruddet, ikke ble berørt.

## Kilder

1.  [Axios: Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking): nesten 6 millioner Downdetector-rapporter på sju dager, tre store avbrudd i stedet for de vanlige fem over 30 dager, vurdering fra Liz Fong-Jones.

2.  [PPC Land: Downdetector](https://ppc.land/downdetector/): funksjonsmåte, grunnlinje per tidspunkt på dagen, oppkjøp av Accenture, avvikende tall for Exchange Online-avbruddet 12. november.

3.  [Futurism: Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage): Downdetector-rapporter hos flere amerikanske banker, uttalelse fra Chase.

4.  [Hot Air: More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101): siterer New York Times og Wall Street Journal om Apple, Charles Schwab og årsaken hos JPMorgan Chase.

5.  [Pasquale Pillitteri: Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports): Downdetector-rapporter om Gmail og Googles avvisning.

6.  [Forbes: Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/): oversikt over operatørenes forklaringer.

7.  [Time: What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/): forløpet for Apple-avbruddet, statussiden lå etter rapportene.

8.  [Hacker News: Diskussion zur Methodik von Downdetector](https://news.ycombinator.com/item?id=50521819): antakelser om implisitte rapporter og overføring mellom lignende tjenester, ikke bekreftet av Downdetector.

9.  [WMTV: Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/): vurdering fra en sikkerhetsekspert, ingen tegn på en sammenheng.
