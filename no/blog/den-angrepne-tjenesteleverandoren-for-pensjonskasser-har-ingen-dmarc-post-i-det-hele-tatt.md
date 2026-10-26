---
title: "Den angrepne tjenesteleverandøren for pensjonskasser har ingen DMARC-post"
navTitle: "DMARC pensjonskasser"
description: "PK Softech, tjenesteleverandøren for pensjonskasser som ble angrepet i slutten av september 2026, har fortsatt ingen DMARC-post. En passiv kontroll av 66 pensjonsordninger og 30 tjenesteleverandører viser at dette ikke er et enkelttilfelle: særlig leverandørene av administrasjonsprogramvare er ubeskyttet."
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
translationSourceHash: a673909df1426a76c58ced5aaa46d839e480ae867b2ffad468c903d65919e0c3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:17:03.796Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/den-angrepne-tjenesteleverandoren-for-pensjonskasser-har-ingen-dmarc-post-i-det-hele-tatt
---

I slutten av september 2026 ble PK Softech AG angrepet, en programvareprodusent hvis pensjonsløsning brukes av flere sveitsiske pensjonskasser. Data lekket denne veien. Ifølge medieomtalen er blant annet den føderale pensjonskassen Publica, Migros' pensjonskasse og pensjonskassen til byen Zürich, samt andre institusjoner, berørt. Avhengig av kassen gjelder det navn, AHV-nummer, sivilstatus, adresse og IBAN, samt lønns- og pensjonsdata.

Denne lekkasjen er alvorlig i seg selv. Den berører identifikasjons- og finansdata for en stor del av den yrkesaktive befolkningen og, gjennom Publica, store deler av den føderale administrasjonen. Data i denne mengden og kvaliteten er ikke bare et problem for den enkelte. De muliggjør identitetsmisbruk, målrettet svindel og utpressing i stor skala, og utgjør dermed en fare for offentlig orden og nasjonal sikkerhet. Selve pensjonsmidlene forble uberørt, men informasjonsplattformen Cybercrimepolice.ch advarte allerede mot svindelanrop med de stjålne dataene.

I tillegg kommer et funn som er vanskelig å forklare for en tjenesteleverandør i denne kategorien: **PK Softech AG har fortsatt ingen DMARC-post.** Domenet deres har en streng SPF-oppføring, men ingen DMARC-policy (kontrollert 18.10.2026). Dermed kan avsenderadressen til den angrepne tjenesteleverandøren forfalskes uten at en mottakende e-postplattform pålitelig avviser dette. Etter en datalekkasje er dette verst tenkelige utgangspunkt. Den som kjenner navn, adresse og bankforbindelse til en forsikret person, og i tillegg kan forfalske en e-post i tjenesteleverandørens eller kassens navn, kan skrive en melding som knapt lar seg skille fra en ekte, for eksempel en oppfordring om å oppdatere IBAN for neste pensjonsutbetaling. Det er nettopp mot denne forfalskningen DMARC beskytter.

PK Softech er ikke et enkelttilfelle, men det tydeligste eksemplet på et mønster. Jeg har undersøkt hvor godt sveitsiske pensjonsordninger og deres typiske tjenesteleverandører er rustet når det gjelder e-postautentisering.

## Hvordan kontrollen ble gjennomført

Analysen omfatter 66 pensjonsordninger i andre søyle, fra de store offentligrettslige kassene via forsikringsselskapenes samlede stiftelser til mindre fellesordninger. Kun offentlige DNS-oppføringer ble kontrollert: SPF, DKIM, DMARC, MX-oppføringer samt DNSSEC og DANE (TLSA-oppføringer for e-postservere, kontrollert 19.10.2026). Dette er de samme opplysningene som ethvert mottakende e-postsystem slår opp for hver innkommende melding.

Kontrollen var utelukkende passiv. Bare offentlig DNS ble forespurt, det ble ikke opprettet forbindelse til institusjonenes systemer, ingen pålogging ble forsøkt og ingen personopplysninger ble innhentet. Analysen gjengir kun det som uansett er synlig for alle fra offentlige kilder. Konsernenes samlede stiftelser er samlet under hoveddomenet til foretaket; enkelte tilknyttede stiftelser kan avvike.

## Resultatet

| Kjennetegn | Andel av de 66 institusjonene |
|---|---|
| SPF finnes | 88 % |
| SPF strengt (`-all`) | 59 % |
| DMARC finnes | 59 % |
| DMARC `p=reject` (full beskyttelse) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (kun overvåking) | 14 % |
| ingen DMARC | 41 % |
| DKIM påvisbar | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tre punkter skiller seg ut.

**Bare 20 prosent er effektivt beskyttet med DMARC `p=reject`.** Først denne policyen instruerer en mottakende e-postplattform om faktisk å avvise forfalskede meldinger. For alle andre vil en forfalskning i tvilstilfeller havne i innboksen til den forsikrede personen.

**41 prosent har ingen DMARC.** Avsenderdomenet deres kan uten videre brukes til svindel-e-poster, og institusjonen får ikke engang vite det, fordi rapporteringsfunksjonen også mangler uten DMARC. Det er påfallende at dette også omfatter kasser som bruker Microsoft 365 for e-post, der DMARC kan konfigureres med liten innsats.

**DMARC `p=none` er ingen beskyttelse.** De 14 prosentene i denne gruppen har satt en DMARC-policy, men latt den stå på overvåkingsnivå. Det gir rapporter, men avviser ikke én eneste forfalsket melding. I praksis er dette et mellomtrinn som ofte blir stående i måneder eller år.

Et lignende bilde viser seg for transportkryptering: Bare 6 av de 66 institusjonene (9 prosent) sikrer e-postserverne sine med DANE. Dermed kontrollerer en sendende e-postserver ved hjelp av en DNSSEC-signert oppføring at den er kryptert koblet til riktig mottaker. Forutsetningen er DNSSEC, som 38 prosent av institusjonene allerede har aktivert. For disse er DANE det nærliggende neste steget.

## Alle datasett for gjennomgang

Følgende infografikk viser kontrollert status for alle 66 pensjonsordningene og 30 tjenesteleverandørene. Med filtrene kan du avgrense etter DMARC-status, kategori og e-postleverandør, og hver oppføring kan utvides for å se de fullstendige DNS-funnene.

<div data-dmarc-infographic></div>

## Til sammenligning: de største sveitsiske selskapene

Hvordan pensjonsordningene står sammenlignet med resten av næringslivet, viser [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index). Den kontrollerer daglig, med samme passive metode, over 1000 av de største sveitsiske selskapene samt forbund, kantoner og kommuner, med et interaktivt kart etter kantoner, sammenligninger etter bransje og e-postleverandør og en detaljside for hver organisasjon. Alle pensjonsordningene og tjenesteleverandørene som er kontrollert her, er inkludert der med egen profil, samlet i bransjen [Vorsorgeeinrichtungen](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) og oppdatert daglig. I infografikken ovenfor fører lenken i den utvidede oppføringen direkte til den aktuelle profilen.

Forskjellen er tydelig: Av de 518 selskapene i indeksen brukte 38 prosent DMARC med `p=reject` 18.10.2026, mot 20 prosent blant pensjonsordningene. Helt uten DMARC er 5 prosent av selskapene, mot 41 prosent av pensjonsordningene.

## Leverandørene er det større temaet

Hendelsen i slutten av september rammet ikke en kasse, men en felles tjenesteleverandør. Derfor har jeg med samme passive metode kontrollert 30 typiske leverandører: produsenter av administrasjonsprogramvare, selskaper for outsourced administrasjon, aktuarvirksomhet og rådgivning, depotbanker og IT-tjenesteleverandører.

I gjennomsnitt står tjenesteleverandørene noe bedre enn kassene: 40 prosent bruker DMARC med `p=reject`, mot 20 prosent blant institusjonene. Det avgjørende er imidlertid fordelingen på kategoriene.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarvirksomhet og rådgivning | god |
| Depotbanker | god |
| IT-tjenesteleverandører | blandet |
| Outsourcet administrasjon | svak |
| Administrasjonsprogramvare | svakest |

Produsentene av administrasjonsprogramvare er den svakeste gruppen. Ingen av de kontrollerte programvareleverandørene bruker DMARC med full effekt, og den angrepne leverandøren PK Softech har ingen DMARC-post i det hele tatt. Dette er nettopp kategorien som behandler pensjonsdata og er tett integrert med kassene gjennom grensesnitt, og det er kategorien for den aktuelle hendelsen.

Dette leder til det egentlige poenget med aktsomheten: En kasse med `p=reject` er godt beskyttet så lenge angriperen vil forfalske domenet dens. Kan angriperen i stedet forfalske domenet til en dårlig beskyttet tjenesteleverandør, som uansett kommuniserer med de forsikrede, er e-posten like troverdig. Sikring av eget domene er derfor bare første steg. Det andre er å forplikte de viktigste tjenesteleverandørene til samme nivå, helst kontraktsmessig.

## Hva dette betyr for stiftelsesrådet

Ansvaret for informasjonssikkerhet ligger hos det øverste organet i pensjonsordningen. En datalekkasje utløser dessuten konkrete plikter etter den reviderte personvernloven: Et brudd på datasikkerheten skal meldes til EDÖB dersom det sannsynligvis medfører høy risiko for de berørte personene, og de berørte skal informeres så langt det er nødvendig for deres beskyttelse.

E-postautentisering er i denne situasjonen det mest effektive enkelttiltaket med minst innsats. SPF, DKIM og DMARC krever ingen lisens og ingen ny programvare. De består av DNS-oppføringer og en korrekt konfigurasjon av den eksisterende e-postplattformen. Beskyttelsen virker umiddelbart for hver forsikrede person, uten at vedkommende selv må gjøre noe.

## Hva som må gjøres

For institusjoner uten effektiv beskyttelse finnes det en tydelig vei til målet:

1. **Kontroller SPF og sett det strengt.** SPF-oppføringen lister opp autoriserte utsendingssystemer og avsluttes med `-all`, slik at alle andre avsendere anses som uautoriserte.
2. **Aktiver DKIM.** Hver utgående melding signeres kryptografisk. I Microsoft 365 og vanlige e-postgatewayer er dette en innstilling, ikke et prosjekt.
3. **Innfør DMARC og skjerp det trinnvis.** Start med `p=none` og aktiv rapportering for å registrere legitim e-posttrafikk. Deretter `p=quarantine`, til slutt `p=reject`. Hele prosessen tar, avhengig av miljøet, noen uker, ikke måneder.
4. **Evaluer rapporteringen.** DMARC-rapportene viser hvem som sender e-post på vegne av domenet, og avdekker både glemte legitime systemer og misbruk.
5. **Vurder i det minste DANE og se på e-postkrypteringen.** DANE sikrer transportkrypteringen mellom e-postservere via DNSSEC. Det kan avklares med liten innsats om egen e-postplattform og DNS-leverandør støtter dette. For meldinger med lønns- og pensjonsdata bør det dessuten være avklart hvordan de sendes ende-til-ende-kryptert, for eksempel via en e-postgateway med S/MIME eller sikker levering gjennom en nettportal.
6. **Hold tjenesteleverandørene ansvarlige.** Programvareleverandører, outsourced administrasjon og IT-partnere bør dokumentere samme nivå for SPF, DKIM og DMARC. Dette hører hjemme i kontraktene og i den regelmessige leverandørkontrollen.

## Konklusjon

Angrepene den siste tiden var rettet mot en tjenesteleverandør, ikke mot kassene selv. Følgeskadene oppstår imidlertid der et avsenderdomene er ubeskyttet, både hos kassen og dens leverandører. Fire av fem kontrollerte pensjonsordninger er ikke fullt beskyttet mot forfalskede avsendere, to av fem har ingen DMARC i det hele tatt, og nettopp produsentene av administrasjonsprogramvare kommer dårligst ut. Gapet er kjent, rimelig å lukke og virker umiddelbart. Etter en datalekkasje som omfatter navn, adresse og bankforbindelse, er dette ikke en teknisk bagatell, men en del av aktsomhetsplikten til det øverste organet, langs hele leverandørkjeden.

## Endringslogg

- **25. oktober 2026:** Kildelenke oppdatert til EDÖBs nye adresse.

## Kilder

1.  [SRF: Den føderale pensjonskassen Publica – hvordan kunne sensitive data havne hos et eksternt programvareselskap](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrunn om hendelsen, berørte datatyper og programvareleverandørens rolle.

2.  [cash.ch (AWP): Angrep på Publica-leverandør, lønnsdata kan ha vært berørt](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica oppgir overfor AWP at PK Softech AG var det angrepne selskapet.

3.  [Netzwoche: Den føderale pensjonskassen Publica melder om datalekkasje](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Melding om angrepet på programvareleverandøren og datalekkasjen.

4.  [Mimikama: Sveitsisk politi advarer etter datalekkasje mot mulige investeringsanrop](https://www.mimikama.org/?p=436628): Vurdering av advarselen om svindelanrop med de lekkede dataene.

5.  [EDÖB: DataBreach](https://www.edoeb.admin.ch/de/databreach): Meldeplikt etter art. 24 DSG, meldingsportal og veiledning for håndtering av brudd på datasikkerheten.
