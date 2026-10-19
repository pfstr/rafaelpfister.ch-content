---
title: "Den angrepne tjenesteleverandøren for pensjonskasser har ingen DMARC-record"
navTitle: "DMARC pensjonskasser"
description: "Tjenesteleverandøren for pensjonskasser, PK Softech, som ble angrepet i slutten av september 2026, har fortsatt ingen DMARC-record. En passiv kontroll av 66 pensjonsordninger og 30 tjenesteleverandører viser at dette ikke er et enkelttilfelle: særlig leverandørene av administrasjonsprogramvare er ubeskyttet."
date: "2026-10-18"
kategorie: "SMTP og e-postflyt"
timeToRead: "9 min lesetid"
themen:
  - smtp-mailflow
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "den-angrepne-tjenesteleverandoren-for-pensjonskasser-har-ingen-dmarc-post-i-det-hele-tatt"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
translationOf: dmarc-pensionskassen-schweiz
translationSourceHash: 56251be16ac00d877c2536fc6384a564114736e38301637d9006f8e21da77d9a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T08:34:42.685Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/den-angrepne-tjenesteleverandoren-for-pensjonskasser-har-ingen-dmarc-post-i-det-hele-tatt
---

I slutten av september 2026 ble PK Softech AG angrepet, en programvareprodusent hvis pensjonsløsning brukes av flere sveitsiske pensjonskasser. Data lekket via denne veien. Ifølge medieomtalen er blant annet den føderale pensjonskassen Publica, Migros' pensjonskasse og pensjonskassen til byen Zürich, samt andre institusjoner, berørt. Avhengig av kassen dreier det seg om navn, AHV-nummer, sivilstatus, adresse og IBAN samt lønns- og pensjonsdata.

Denne lekkasjen er alvorlig i seg selv. Den berører identifikasjons- og finansdata for en stor del av den yrkesaktive befolkningen og, gjennom Publica, store deler av den føderale forvaltningen. Data i denne mengden og kvaliteten er ikke bare et problem for den enkelte. De muliggjør identitetsmisbruk, målrettet svindel og utpressing i stor skala og utgjør dermed en fare for offentlig orden og nasjonal sikkerhet. Selve pensjonsmidlene forble uberørt, men informasjonsplattformen Cybercrimepolice.ch advarte allerede mot svindelanrop med de stjålne dataene.

I tillegg kommer et funn som er vanskelig å forklare for en tjenesteleverandør i denne kategorien: **PK Softech AG har fortsatt ingen DMARC-record.** Domenet deres har en streng SPF-oppføring, men ingen DMARC-policy (kontrollert 18.10.2026). Dermed kan avsenderadressen til den angrepne tjenesteleverandøren forfalskes uten at en mottakende e-postplattform pålitelig avviser dette. Etter en datalekkasje er dette det verst tenkelige utgangspunktet. Den som kjenner navn, adresse og bankforbindelse til en forsikret person og i tillegg kan forfalske en e-post i tjenesteleverandørens eller kassens navn, kan skrive en melding som knapt lar seg skille fra en ekte, for eksempel en oppfordring om å oppdatere IBAN for neste pensjonsutbetaling. Det er nettopp mot slik forfalskning DMARC beskytter.

PK Softech er ikke et enkelttilfelle, men det tydeligste eksempelet på et mønster. Jeg har undersøkt hvor godt sveitsiske pensjonsordninger og deres typiske tjenesteleverandører er rustet når det gjelder e-postautentisering.

## Hvordan kontrollen ble gjennomført

Analysen omfatter 66 pensjonsordninger i andre søyle, fra de store offentligrettslige kassene via forsikringsselskapenes fellesstiftelser til mindre fellesordninger. Kun offentlige DNS-oppføringer ble kontrollert: SPF, DKIM, DMARC, MX-oppføringer samt DNSSEC og DANE (TLSA-oppføringer for e-postserverne, kontrollert 19.10.2026). Dette er de samme opplysningene som ethvert mottakende e-postsystem forespør ved hver innkommende melding.

Kontrollen var rent passiv. Det ble bare gjort oppslag i offentlig DNS, det ble ikke opprettet forbindelse til institusjonenes systemer, ikke forsøkt innlogging og ikke innhentet personopplysninger. Analysen gjengir bare det som uansett er synlig for alle fra offentlige kilder. Konsernfellesstiftelser er samlet under hoveddomenet til den ansvarlige enheten; enkelte tilknyttede stiftelser kan avvike.

## Resultatet

| Kjennetegn | Andel av de 66 institusjonene |
|---|---|
| SPF finnes | 88 % |
| Streng SPF (`-all`) | 59 % |
| DMARC finnes | 59 % |
| DMARC `p=reject` (full beskyttelse) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (kun overvåking) | 14 % |
| Ingen DMARC | 41 % |
| DKIM påviselig | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tre punkter skiller seg ut.

**Bare 20 prosent er effektivt beskyttet med DMARC `p=reject`.** Først denne policyen instruerer en mottakende e-postplattform om faktisk å avvise forfalskede meldinger. I alle andre tilfeller havner en forfalskning i tvilstilfeller i innboksen til den forsikrede personen.

**41 prosent har ingen DMARC.** Avsenderdomenet deres kan uten videre brukes til svindel-e-poster, og institusjonen får ikke engang vite om det, fordi rapporteringsfunksjonen også mangler uten DMARC. Det er påfallende at dette også omfatter kasser som bruker Microsoft 365 til e-post, der DMARC kan settes opp med liten innsats.

**DMARC `p=none` er ingen beskyttelse.** De 14 prosentene i denne gruppen har satt en DMARC-policy, men latt den stå på overvåkingsnivået. Det gir rapporter, men avviser ikke én eneste forfalsket melding. I praksis er dette et mellomsteg som ofte blir stående i måneder eller år.

Transportkrypteringen viser et lignende bilde: Bare 6 av de 66 institusjonene (9 prosent) sikrer e-postserverne sine med DANE. Da kontrollerer en avsendende e-postserver, ved hjelp av en oppføring signert med DNSSEC, om den er kryptert koblet til riktig mottaker. Forutsetningen er DNSSEC, som 38 prosent av institusjonene allerede har aktivert. For disse er DANE det naturlige neste steget.

## Alle datasett klikkbart

Følgende infografikk viser den kontrollerte statusen for alle 66 pensjonsordninger og 30 tjenesteleverandører. Med filtrene kan du avgrense etter DMARC-status, kategori og e-postleverandør, og hver oppføring kan utvides for å se de fullstendige DNS-funnene.

<div data-dmarc-infographic></div>

## Til sammenligning: de største sveitsiske selskapene

Hvordan pensjonsordningene ligger an sammenlignet med resten av næringslivet, viser [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index). Den kontrollerer daglig over 500 av de største sveitsiske selskapene med samme passive metode, med interaktivt kart etter kantoner, sammenligninger etter bransje og e-postleverandør og en detaljside for hvert selskap. Dette omfatter også bankene, forsikringsselskapene og IT-tjenesteleverandørene som pensjonsordningene samarbeider med.

Forskjellen er tydelig: Av de 518 selskapene i indeksen bruker 38 prosent DMARC med `p=reject` per 18.10.2026, mens andelen blant pensjonsordningene er 20 prosent. Helt uten DMARC er 5 prosent av selskapene, mot 41 prosent av pensjonsordningene.

## Leverandørene er det større temaet

Hendelsen i slutten av september rammet ikke en kasse, men en felles tjenesteleverandør. Derfor har jeg kontrollert 30 typiske underleverandører med samme passive metode: produsenter av administrasjonsprogramvare, selskaper for utkontraktert administrasjon, aktuarvirksomhet og rådgivning, depotbanker og IT-tjenesteleverandører.

I gjennomsnitt står tjenesteleverandørene noe bedre rustet enn kassene: 40 prosent bruker DMARC med `p=reject`, mot 20 prosent blant institusjonene. Det avgjørende er imidlertid fordelingen mellom kategoriene.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarvirksomhet og rådgivning | god |
| Depotbanker | god |
| IT-tjenesteleverandører | blandet |
| Utkontraktert administrasjon | svak |
| Administrasjonsprogramvare | svakest |

Produsentene av administrasjonsprogramvare er den svakeste gruppen. Ingen av de kontrollerte programvareleverandørene bruker DMARC med full effekt, og den angrepne leverandøren PK Softech har ingen DMARC-record i det hele tatt. Dette er nettopp kategorien som behandler pensjonsdata og er tett integrert med kassene gjennom grensesnitt, og det er kategorien i den aktuelle hendelsen.

Dette leder til det egentlige poenget når det gjelder aktsomhet: En kasse med `p=reject` er godt beskyttet så lenge angriperen vil forfalske domenet dens. Kan angriperen i stedet forfalske domenet til en dårlig beskyttet tjenesteleverandør som uansett kommuniserer med de forsikrede, er e-posten like troverdig. Sikring av eget domene er derfor bare første steg. Det andre er å forplikte de viktigste tjenesteleverandørene til samme nivå, helst kontraktsmessig.

## Hva dette betyr for stiftelsesrådet

Ansvaret for informasjonssikkerheten ligger hos det øverste organet i pensjonsordningen. En datalekkasje utløser dessuten konkrete plikter etter den reviderte personvernloven: Et brudd på datasikkerheten skal meldes til EDÖB dersom det sannsynligvis medfører høy risiko for de berørte personene, og de berørte skal informeres i den grad det er nødvendig for deres beskyttelse.

E-postautentisering er i denne situasjonen det mest effektive enkelttiltaket med minst innsats. SPF, DKIM og DMARC krever ingen lisens og ingen ny programvare. De består av DNS-oppføringer og en korrekt konfigurasjon av den eksisterende e-postplattformen. Beskyttelsen virker umiddelbart for hver forsikret person, uten at vedkommende selv må gjøre noe.

## Hva som må gjøres

For institusjoner uten effektiv beskyttelse finnes det en klar vei til målet:

1. **Kontroller SPF og sett det strengt.** SPF-oppføringen lister opp de autoriserte utsendingssystemene og avsluttes med `-all`, slik at alle øvrige avsendere anses som ikke autoriserte.
2. **Aktiver DKIM.** Hver utgående melding signeres kryptografisk. I Microsoft 365 og i de vanlige e-postgatewayene er dette en innstilling, ikke et prosjekt.
3. **Innfør DMARC og skjerp den trinnvis.** Start med `p=none` og aktiv rapportering for å registrere legitim e-posttrafikk. Deretter `p=quarantine`, til slutt `p=reject`. Hele prosessen tar, avhengig av miljøet, noen uker, ikke måneder.
4. **Evaluer rapporteringen.** DMARC-rapportene viser hvem som sender e-post på vegne av domenet, og avdekker både glemte legitime systemer og misbruk.
5. **Vurder i det minste DANE og se på e-postkrypteringen.** DANE sikrer transportkrypteringen mellom e-postservere via DNSSEC. Om egen e-postplattform og DNS-leverandøren støtter dette, kan avklares med liten innsats. For meldinger med lønns- og pensjonsdata bør det dessuten fastsettes hvordan de sendes ende-til-ende-kryptert, for eksempel via en e-postgateway med S/MIME eller sikker levering via en nettportal.
6. **Hold tjenesteleverandørene ansvarlige.** Programvareleverandører, utkontraktert administrasjon og IT-partnere bør dokumentere samme nivå for SPF, DKIM og DMARC. Dette hører hjemme i kontraktene og i den regelmessige leverandørkontrollen.

## Konklusjon

Angrepene i den senere tiden var rettet mot en tjenesteleverandør, ikke mot kassene selv. Følgeskadene oppstår imidlertid der et avsenderdomene er ubeskyttet, både hos kassen og dens underleverandører. Fire av fem kontrollerte pensjonsordninger er ikke fullt beskyttet mot forfalskede avsendere, to av fem har ingen DMARC i det hele tatt, og nettopp produsentene av administrasjonsprogramvare kommer dårligst ut. Gapet er kjent, rimelig å lukke og har umiddelbar effekt. Etter en datalekkasje som berører navn, adresse og bankforbindelse, er dette ikke en teknisk detalj, men en del av aktsomhetsplikten til det øverste organet, langs hele leverandørkjeden.

## Kilder

1.  [SRF: Den føderale pensjonskassen – hvordan kunne sensitive data havne hos et eksternt programvareselskap?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrunn om hendelsen, berørte datatyper og programvareleverandørens rolle.

2.  [cash.ch (AWP): Angrep på Publicas leverandør, lønnsdata kan ha vært berørt](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica oppgir overfor AWP at PK Softech AG er selskapet som ble angrepet.

3.  [Netzwoche: Den føderale pensjonskassen Publica melder om datalekkasje](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Melding om angrepet på programvareleverandøren og datalekkasjen.

4.  [Mimikama: Sveitsisk politi advarer etter datalekkasje mot mulige investeringsanrop](https://www.mimikama.org/?p=436628): Vurdering av advarselen mot svindelanrop med de lekkede dataene.

5.  [EDÖB: Melding av brudd på datasikkerheten](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Meldeplikt etter revDSG og fremgangsmåte ved brudd på datasikkerheten.
