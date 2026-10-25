---
title: "Den angrepne tjenesteleverandøren for pensjonskasser har ingen DMARC-record"
navTitle: "DMARC pensjonskasser"
description: "Tjenesteleverandøren for pensjonskasser, PK Softech, som ble angrepet i slutten av september 2026, har fortsatt ingen DMARC-record. En passiv kontroll av 66 pensjonsordninger og 30 tjenesteleverandører viser at dette ikke er et enkelttilfelle: særlig produsentene av administrasjonsprogramvare er ubeskyttet."
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
translationSourceHash: 357621c8e85052ed11d792c9e6f911df15e6e32a0ad4cf0c5d92ba1fdbd9f906
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:35:21.979Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/den-angrepne-tjenesteleverandoren-for-pensjonskasser-har-ingen-dmarc-post-i-det-hele-tatt
---

I slutten av september 2026 ble PK Softech AG angrepet, en programvareprodusent hvis pensjonsløsning brukes av flere sveitsiske pensjonskasser. Data lekket via denne veien. Ifølge medieomtalen er blant andre den føderale pensjonskassen Publica, pensjonskassen til Migros og pensjonskassen til byen Zürich, samt andre institusjoner, berørt. Avhengig av kassen gjelder det navn, AHV-nummer, sivilstatus, adresse og IBAN, samt lønns- og pensjonsdata.

Denne lekkasjen er alvorlig i seg selv. Den berører identifikasjons- og finansdata for en stor del av den yrkesaktive befolkningen og, gjennom Publica, store deler av den føderale administrasjonen. Data i denne mengden og kvaliteten er ikke bare et problem for den enkelte personen. De muliggjør identitetsmisbruk, målrettet bedrag og utpressing i stor skala og utgjør dermed en fare for offentlig orden og nasjonal sikkerhet. Selve pensjonsmidlene forble uberørt, men informasjonsplattformen Cybercrimepolice.ch advarte allerede mot svindelanrop med de stjålne dataene.

I tillegg kommer et funn som er vanskelig å forklare for en tjenesteleverandør i denne kategorien: **PK Softech AG har fortsatt ingen DMARC-record.** Domenet har en streng SPF-oppføring, men ingen DMARC-policy (kontrollert 18.10.2026). Dermed kan avsenderadressen til den angrepne tjenesteleverandøren forfalskes uten at en mottakende e-postplattform pålitelig avviser den. Etter en datalekkasje er dette den verst tenkelige situasjonen. Den som kjenner navn, adresse og bankforbindelse til en forsikret person og i tillegg kan forfalske en e-post i tjenesteleverandørens eller kassens navn, kan skrive en melding som knapt kan skilles fra en ekte, for eksempel en oppfordring om å oppdatere IBAN for neste pensjonsutbetaling. Det er nettopp mot denne forfalskningen DMARC beskytter.

PK Softech er ikke et enkelttilfelle, men det tydeligste eksempelet på et mønster. Jeg har undersøkt hvor godt sveitsiske pensjonsordninger og deres typiske tjenesteleverandører er rustet når det gjelder e-postautentisering.

## Hvordan kontrollen ble gjennomført

Analysen omfatter 66 pensjonsordninger i den andre søylen, fra de store offentligrettslige kassene via forsikringsselskapenes fellesstiftelser til mindre fellesordninger. Kun offentlige DNS-oppføringer ble kontrollert: SPF, DKIM, DMARC, MX-oppføringer samt DNSSEC og DANE (TLSA-oppføringer for e-postserverne, kontrollert 19.10.2026). Dette er de samme opplysningene som ethvert mottakende e-postsystem slår opp for hver innkommende melding.

Kontrollen var utelukkende passiv. Bare offentlig DNS ble forespurt, det ble ikke opprettet forbindelse til institusjonenes systemer, ingen pålogging ble forsøkt og ingen personopplysninger ble innhentet. Analysen gjengir bare det som uansett kan leses av fra offentlige kilder. Konsernenes fellesstiftelser er samlet under hoveddomenet til enheten som står bak; enkelte tilknyttede stiftelser kan avvike.

## Resultatet

| Kjennetegn | Andel av de 66 institusjonene |
|---|---|
| SPF finnes | 88 % |
| SPF streng (`-all`) | 59 % |
| DMARC finnes | 59 % |
| DMARC `p=reject` (full beskyttelse) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (kun overvåking) | 14 % |
| ingen DMARC | 41 % |
| DKIM synlig | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tre punkter skiller seg ut.

**Bare 20 prosent er effektivt beskyttet med DMARC `p=reject`.** Først denne policyen instruerer en mottakende e-postplattform om faktisk å avvise forfalskede meldinger. I alle andre tilfeller havner en forfalskning i tvilstilfeller i innboksen til den forsikrede personen.

**41 prosent har ingen DMARC.** Avsenderdomenet deres kan uten videre brukes til svindel-e-poster, og institusjonen får ikke engang vite om det, fordi rapporteringsfunksjonen også mangler uten DMARC. Det er påfallende at dette også omfatter kasser som kjører e-posten sin hos Microsoft 365, hvor DMARC kan settes opp med liten innsats.

**DMARC `p=none` er ingen beskyttelse.** De 14 prosentene i denne gruppen har satt en DMARC-policy, men latt den stå på overvåkingsnivå. Det gir rapporter, men avviser ikke én eneste forfalsket melding. I praksis er dette et mellomsteg som ofte blir stående i måneder eller år.

For transportkryptering viser det seg et tilsvarende bilde: Bare 6 av de 66 institusjonene (9 prosent) sikrer e-postserverne sine med DANE. Dermed kontrollerer en sendende e-postserver, basert på en DNSSEC-signert oppføring, om den er kryptert koblet til riktig mottaker. Forutsetningen er DNSSEC, som 38 prosent av institusjonene allerede har aktivert. For disse er DANE det naturlige neste steget.

## Alle datasett å klikke gjennom

Følgende infografikk viser den kontrollerte statusen for alle 66 pensjonsordningene og 30 tjenesteleverandørene. Med filtrene kan du avgrense etter DMARC-status, kategori og e-postleverandør, og hver oppføring kan utvides for å vise de fullstendige DNS-funnene.

<div data-dmarc-infographic></div>

## Til sammenligning: de største sveitsiske selskapene

Hvordan pensjonsordningene står seg sammenlignet med resten av næringslivet, viser [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index). Den kontrollerer daglig, med samme passive metode, over 1000 av de største sveitsiske selskapene samt forbund, kantoner og kommuner, med interaktivt kart etter kantoner, sammenligninger etter bransje og e-postleverandør og en detaljside for hver organisasjon. Alle pensjonsordningene og tjenesteleverandørene som er kontrollert her, er inkludert der med egen profil, samlet i bransjen [pensjonsordninger](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) og oppdateres daglig. I infografikken over leder lenken i den utvidede oppføringen direkte til den aktuelle profilen.

Forskjellen er tydelig: Av de 518 selskapene i indeksen brukte 38 prosent DMARC med `p=reject` 18.10.2026, mens andelen for pensjonsordningene er 20 prosent. Helt uten DMARC er 5 prosent av selskapene, mot 41 prosent av pensjonsordningene.

## Leverandørene er det større temaet

Hendelsen i slutten av september rammet ikke en kasse, men en felles tjenesteleverandør. Derfor har jeg kontrollert 30 typiske leverandører med samme passive metode: produsenter av administrasjonsprogramvare, selskaper for utkontraktert administrasjon, aktuarvirksomhet og konsulenttjenester, depotbanker og IT-tjenesteleverandører.

I gjennomsnitt står tjenesteleverandørene noe bedre enn kassene: 40 prosent bruker DMARC med `p=reject`, mot 20 prosent blant institusjonene. Det avgjørende er imidlertid fordelingen mellom kategoriene.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarvirksomhet og konsulenttjenester | god |
| Depotbanker | god |
| IT-tjenesteleverandører | blandet |
| Utkontraktert administrasjon | svak |
| Administrasjonsprogramvare | svakest |

Produsentene av administrasjonsprogramvare er den svakeste gruppen. Ingen av de kontrollerte programvareleverandørene bruker DMARC med full effekt, og den angrepne leverandøren PK Softech har ingen DMARC-record i det hele tatt. Dette er nettopp kategorien som behandler pensjonsdata og er tett integrert med kassene gjennom grensesnitt, og det er kategorien i den aktuelle hendelsen.

Dette leder til det sentrale punktet for aktsomhet: En kasse med `p=reject` er godt beskyttet så lenge angriperen vil forfalske domenet dens. Hvis angriperen i stedet kan forfalske domenet til en dårlig beskyttet tjenesteleverandør, som uansett kommuniserer med de forsikrede, er e-posten like troverdig. Sikring av eget domene er derfor bare første steg. Det andre er å forplikte de viktigste tjenesteleverandørene til samme nivå, helst kontraktsfestet.

## Hva dette betyr for stiftelsesstyret

Ansvaret for informasjonssikkerheten ligger hos det øverste organet i pensjonsordningen. En datalekkasje utløser dessuten konkrete plikter etter den reviderte personvernloven: Et brudd på datasikkerheten skal meldes til EDÖB dersom det sannsynligvis medfører høy risiko for de berørte personene, og de berørte skal informeres når det er nødvendig for deres beskyttelse.

E-postautentisering er i denne situasjonen det mest effektive enkelttiltaket med minst innsats. SPF, DKIM og DMARC krever ingen lisens og ingen ny programvare. De består av DNS-oppføringer og en korrekt konfigurasjon av den eksisterende e-postplattformen. Beskyttelsen virker umiddelbart for hver forsikrede person, uten at personen selv må gjøre noe.

## Hva som må gjøres

For institusjoner uten effektiv beskyttelse finnes det en klar vei til målet:

1. **Kontroller SPF og sett den strengt.** SPF-oppføringen lister de autoriserte utsendelsessystemene og avsluttes med `-all`, slik at alle øvrige avsendere regnes som ikke autoriserte.
2. **Aktiver DKIM.** Hver utgående melding signeres kryptografisk. I Microsoft 365 og de vanlige e-postgatewayene er dette en innstilling, ikke et prosjekt.
3. **Innfør DMARC og skjerp den trinnvis.** Start med `p=none` og aktiv rapportering for å registrere legitim e-posttrafikk. Deretter `p=quarantine`, til slutt `p=reject`. Hele prosessen tar, avhengig av miljøet, noen uker, ikke måneder.
4. **Evaluer rapporteringen.** DMARC-rapportene viser hvem som sender e-post på vegne av domenet, og avdekker både glemte legitime systemer og misbruk.
5. **Vurder i det minste DANE og se på e-postkrypteringen.** DANE sikrer transportkrypteringen mellom e-postservere via DNSSEC. Om egen e-postplattform og DNS-leverandøren støtter dette, kan avklares med liten innsats. For meldinger med lønns- og pensjonsdata må det dessuten fastsettes hvordan de sendes ende-til-ende-kryptert, for eksempel via en e-postgateway med S/MIME eller sikker levering gjennom en nettportal.
6. **Hold tjenesteleverandørene ansvarlige.** Programvareleverandører, utkontraktert administrasjon og IT-partnere bør dokumentere samme nivå for SPF, DKIM og DMARC. Dette hører hjemme i kontraktene og i den regelmessige leverandørkontrollen.

## Konklusjon

Angrepene i den siste tiden var rettet mot en tjenesteleverandør, ikke mot kassene selv. Følgeskadene oppstår imidlertid der et avsenderdomene er ubeskyttet, både hos kassen og hos leverandørene. Fire av fem kontrollerte pensjonsordninger er ikke fullt beskyttet mot forfalskede avsendere, to av fem har ingen DMARC i det hele tatt, og nettopp produsentene av administrasjonsprogramvare gjør det dårligst. Gapet er kjent, rimelig å lukke og virker umiddelbart. Etter en datalekkasje som berører navn, adresse og bankforbindelse, er det ikke en teknisk bagatell, men en del av aktsomhetsplikten til det øverste organet, gjennom hele leverandørkjeden.

## Kilder

1.  [SRF: Forbundets pensjonskasse – hvordan kunne sensitive data havne hos et eksternt programvareselskap](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrunn om hendelsen, berørte datatyper og programvareleverandørens rolle.

2.  [cash.ch (AWP): Angrep på Publica-leverandør, lønnsdata kan ha vært berørt](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica oppgir PK Softech AG som det angrepne selskapet overfor AWP.

3.  [Netzwoche: Forbundets pensjonskasse Publica melder om datalekkasje](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Melding om angrepet på programvareleverandøren og datalekkasjen.

4.  [Mimikama: Sveitsisk politi advarer mot mulige investeringsanrop etter datalekkasje](https://www.mimikama.org/?p=436628): Vurdering av advarselen mot svindelanrop med de lekkede dataene.

5.  [EDÖB: DataBreach](https://www.edoeb.admin.ch/de/databreach): Meldeplikt etter art. 24 DSG, meldeportal og veiledning for håndtering av brudd på datasikkerheten.
