---
title: "Den angrepne tjenesteleverandøren for pensjonskasser har ingen DMARC-post i det hele tatt"
navTitle: "DMARC pensjonskasser"
description: "PK Softech, tjenesteleverandøren for pensjonskasser som ble angrepet i slutten av september 2026, har fortsatt ingen DMARC-post. En passiv undersøkelse av 66 pensjonsinnretninger og 30 tjenesteleverandører viser at dette ikke er et enkelttilfelle: særlig leverandørene av administrasjonsprogramvare er ubeskyttet."
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
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
translationOf: dmarc-pensionskassen-schweiz
url: https://rafaelpfister.ch/no/blog/den-angrepne-tjenesteleverandoren-for-pensjonskasser-har-ingen-dmarc-post-i-det-hele-tatt
translationSourceHash: 64ec035a358c39b09487102da66cf7aca02df2c567e29e343895906a4490d205
translationModel: gpt-5.6-terra
translatedAt: 2026-10-18T14:55:08.904Z
translationReview: automatic
---

I slutten av september 2026 ble PK Softech AG angrepet, en programvareprodusent hvis pensjonsløsning brukes av flere sveitsiske pensjonskasser. Data ble lekket via denne veien. Ifølge medieomtalen er blant annet den føderale pensjonskassen Publica, Migros' pensjonskasse og pensjonskassen i byen Zürich, samt andre institusjoner, berørt. Avhengig av kassen gjelder det navn, AHV-nummer, sivilstatus, adresse og IBAN, samt lønns- og pensjonsdata.

Denne lekkasjen er alvorlig i seg selv. Den omfatter identifikasjons- og finansdata for en stor del av den yrkesaktive befolkningen og, via Publica, store deler av den føderale forvaltningen. Data i denne mengden og kvaliteten er ikke bare et problem for den enkelte personen. De muliggjør identitetsmisbruk, målrettet svindel og utpressing i stor skala og utgjør dermed en fare for offentlig orden og nasjonal sikkerhet. Selve pensjonsmidlene forble uberørt, men Cybercrimepolice advarte allerede mot svindelanrop med de stjålne dataene.

I tillegg kommer et funn som er vanskelig å forklare for en tjenesteleverandør i denne kategorien: **PK Softech AG har fortsatt ingen DMARC-post.** Domenet har en streng SPF-post, men ingen DMARC-policy (kontrollert 18.10.2026). Avsenderadressen til den angrepne tjenesteleverandøren kan dermed forfalskes uten at en mottakende e-postplattform pålitelig avviser den. Etter en datalekkasje er dette verst tenkelige utgangspunkt. Den som kjenner navn, adresse og bankforbindelse til en forsikret person og i tillegg kan forfalske en e-post i tjenesteleverandørens eller kassens navn, kan skrive en melding som knapt lar seg skille fra en ekte, for eksempel en oppfordring om å oppdatere IBAN for neste pensjonsutbetaling. Det er nettopp mot denne forfalskningen DMARC beskytter.

PK Softech er ikke et enkelttilfelle, men det tydeligste eksempelet på et mønster. Jeg har undersøkt hvor godt sveitsiske pensjonsinnretninger og deres typiske tjenesteleverandører er stilt når det gjelder e-postautentisering.

## Hvordan undersøkelsen ble gjennomført

Analysen omfatter 66 pensjonsinnretninger i andre søyle, fra store offentligrettslige kasser via forsikringsselskapenes fellesstiftelser til mindre fellesinnretninger. Det ble utelukkende undersøkt offentlige DNS-poster: SPF, DKIM, DMARC, MTA-STS, TLS-RPT og MX-postene. Dette er de samme opplysningene som ethvert mottakende e-postsystem spør etter for hver innkommende melding.

Undersøkelsen var rent passiv. Det ble bare foretatt oppslag i offentlig DNS, ingen forbindelse til institusjonenes systemer ble opprettet, ingen innlogging forsøkt og ingen personopplysninger innhentet. Analysen gjengir bare det som uansett er offentlig tilgjengelig for alle. Konserntilknyttede fellesstiftelser er samlet under hoveddomenet til foretaket; enkelte tilknyttede stiftelser kan avvike.

## Resultatet

| Kjennetegn | Andel av de 66 institusjonene |
|---|---|
| SPF finnes | 88 % |
| Streng SPF (`-all`) | 59 % |
| DMARC finnes | 59 % |
| DMARC `p=reject` (full beskyttelse) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (kun overvåking) | 14 % |
| ingen DMARC | 41 % |
| DKIM påvisbar | 55 % |
| MTA-STS | 2 % |
| TLS-RPT | 5 % |

Tre punkter skiller seg ut.

**Bare 20 prosent er effektivt beskyttet med DMARC `p=reject`.** Først denne policyen instruerer en mottakende e-postplattform om faktisk å avvise forfalskede meldinger. I alle andre tilfeller havner en forfalskning i tvilstilfeller i innboksen til den forsikrede personen.

**41 prosent har ingen DMARC.** Avsenderdomenet deres kan uten videre brukes til svindel-e-post, og institusjonen får ikke engang vite om det, fordi rapporteringsfunksjonen også mangler uten DMARC. Det er påfallende at dette også omfatter kasser som bruker Microsoft 365 til e-post, der DMARC kan settes opp med liten innsats.

**DMARC `p=none` er ingen beskyttelse.** De 14 prosentene i denne gruppen har satt en DMARC-policy, men latt den stå på overvåkingsnivå. Det gir rapporter, men avviser ikke én eneste forfalsket melding. I praksis er dette et mellomtrinn som ofte blir stående i måneder eller år.

I tillegg kommer et gjennomgående funn: MTA-STS er med 2 prosent praktisk talt fraværende. Uten MTA-STS håndheves ikke transportkryptering for innkommende e-post, noe som gagner en angriper med tilgang til transportveien.

## Alle datasettene du kan klikke deg gjennom

Følgende infografikk viser den undersøkte statusen for alle 66 pensjonsinnretningene og 30 tjenesteleverandørene. Med filtrene kan du avgrense etter DMARC-status, kategori og e-postleverandør, og hver oppføring kan utvides for å vise fullstendige DNS-funn.

<div data-dmarc-infographic></div>

## Til sammenligning: de største sveitsiske selskapene

Hvordan pensjonsinnretningene står sammenlignet med resten av næringslivet, viser [E-postsikkerhetsindeksen Sveits](/open-data/mail-security-index). Den undersøker daglig over 500 av de største sveitsiske selskapene med samme passive metode, med et interaktivt kart over kantonene, sammenligninger etter bransje og e-postleverandør samt en detaljside for hvert selskap. Blant dem er også bankene, forsikringsselskapene og IT-tjenesteleverandørene som pensjonsinnretningene samarbeider med.

Forskjellen er tydelig: Av de 518 selskapene i indeksen bruker 38 prosent DMARC med `p=reject` per 18.10.2026, mot 20 prosent blant pensjonsinnretningene. Helt uten DMARC er 5 prosent av selskapene, mot 41 prosent blant pensjonsinnretningene.

## Leverandørene er det større temaet

Hendelsen i slutten av september rammet ikke en kasse, men en felles tjenesteleverandør. Derfor har jeg undersøkt 30 typiske leverandører med samme passive metode: produsenter av administrasjonsprogramvare, selskaper for outsourcet forretningsadministrasjon, aktuarvirksomhet og rådgivning, depotbanker og IT-tjenesteleverandører.

I gjennomsnitt står tjenesteleverandørene noe bedre enn kassene: 40 prosent bruker DMARC med `p=reject`, mot 20 prosent blant institusjonene. Det avgjørende er imidlertid fordelingen mellom kategoriene.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarvirksomhet og rådgivning | god |
| Depotbanker | god |
| IT-tjenesteleverandører | blandet |
| Outsourcet forretningsadministrasjon | svak |
| Administrasjonsprogramvare | svakest |

Produsentene av administrasjonsprogramvare er den svakeste gruppen. Ingen av de undersøkte programvareleverandørene bruker DMARC med full effekt, og den angrepne leverandøren PK Softech har ingen DMARC-post i det hele tatt. Dette er nettopp kategorien som behandler pensjonsdata og er tett koblet til kassene gjennom grensesnitt, og det er kategorien for den aktuelle hendelsen.

Dette leder til det egentlige punktet for aktsomhet: En kasse med `p=reject` er godt beskyttet så lenge angriperen vil forfalske dens domene. Kan angriperen i stedet forfalske domenet til en dårlig beskyttet tjenesteleverandør som uansett kommuniserer med de forsikrede, er e-posten like troverdig. Sikring av eget domene er derfor bare første steg. Det andre er å forplikte de viktigste tjenesteleverandørene til samme nivå, helst kontraktsmessig.

## Hva dette betyr for stiftelsesrådet

Ansvaret for informasjonssikkerhet ligger hos det øverste organet i pensjonsinnretningen. En datalekkasje utløser dessuten konkrete plikter etter den reviderte personvernloven: Et brudd på datasikkerheten skal meldes til EDÖB dersom det sannsynligvis medfører høy risiko for de berørte personene, og de berørte skal informeres i den grad det er nødvendig for deres beskyttelse.

E-postautentisering er i denne situasjonen det mest effektive enkelttiltaket med minst innsats. SPF, DKIM og DMARC krever ingen lisens og ingen ny programvare. De består av DNS-poster og en korrekt konfigurasjon av den eksisterende e-postplattformen. Beskyttelsen virker umiddelbart for hver forsikrede person, uten at vedkommende selv må gjøre noe.

## Hva som må gjøres

For institusjoner uten effektiv beskyttelse fører en klar vei til målet:

1. **Kontroller SPF og sett den strengt.** SPF-posten lister opp de autoriserte utsendelsessystemene og avsluttes med `-all`, slik at alle øvrige avsendere anses som uautoriserte.
2. **Aktiver DKIM.** Hver utgående melding signeres kryptografisk. I Microsoft 365 og vanlige e-postgatewayer er dette en innstilling, ikke et prosjekt.
3. **Innfør DMARC og stram den gradvis inn.** Start med `p=none` og aktiv rapportering for å registrere legitim e-posttrafikk. Deretter `p=quarantine`, til slutt `p=reject`. Hele prosessen tar, avhengig av miljøet, noen uker, ikke måneder.
4. **Analyser rapporteringen.** DMARC-rapportene viser hvem som sender e-post på vegne av domenet, og avdekker både glemte legitime systemer og misbruk.
5. **Suppler med MTA-STS og TLS-RPT.** Begge håndhever og overvåker transportkrypteringen for innkommende e-post.
6. **Hold tjenesteleverandørene ansvarlige.** Programvareleverandører, outsourcet forretningsadministrasjon og IT-partnere bør dokumentere samme nivå for SPF, DKIM og DMARC. Dette hører hjemme i kontraktene og i den regelmessige leverandørkontrollen.

## Konklusjon

Angrepene i den senere tiden var rettet mot en tjenesteleverandør, ikke mot kassene selv. Følgeskadene oppstår imidlertid der et avsenderdomene er ubeskyttet, både hos kassen og hos dens leverandører. Fire av fem undersøkte pensjonsinnretninger er ikke fullt beskyttet mot forfalskede avsendere, to av fem har ingen DMARC i det hele tatt, og nettopp produsentene av administrasjonsprogramvare kommer dårligst ut. Gapet er kjent, rimelig å lukke og virker umiddelbart. Etter en datalekkasje som omfatter navn, adresse og bankforbindelse, er det ikke en teknisk detalj, men en del av aktsomhetsplikten til det øverste organet, langs hele leverandørkjeden.

## Kilder

1.  [SRF: Den føderale pensjonskassen – hvordan kunne sensitive data havne hos et eksternt programvareselskap](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrunn om hendelsen, berørte datatyper og programvareleverandørens rolle.

2.  [cash.ch (AWP): Angrep på Publica-leverandør, lønnsdata kan ha vært berørt](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica oppgir PK Softech AG som det angrepne selskapet overfor AWP.

3.  [Netzwoche: Den føderale pensjonskassen Publica melder om datalekkasje](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Melding om angrepet på programvareleverandøren og datalekkasjen.

4.  [Mimikama: Sveitsisk politi advarer etter datalekkasje mot mulige investeringsanrop](https://www.mimikama.org/?p=436628): Vurdering av advarselen mot svindelanrop med de lekkede dataene.

5.  [EDÖB: Melding av brudd på datasikkerheten](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Meldeplikt etter revDSG og fremgangsmåte ved brudd på datasikkerheten.
