---
title: "S/MIME i banker: Rettslig situasjon i Sveits og valget mellom Silver og Gold"
navTitle: "S/MIME Silver/Gold"
description: "Hvilke rettslige krav gjelder når en sveitsisk bank signerer og krypterer e-poster til kunder: bankkundetaushetsplikt, FINMA-rundskriv 2023/1, DSG, ZertES, bevisrett og oppbevaring. I tillegg: hvilken SwissSign-sertifikatnivå som følger av dette."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "11 min lesetid"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-med-banker-silver-eller-gold-sertifikat-fra-swisssign"
translationId: "article-5c2fd8d42f088dcb"
translationOf: smime-silver-gold-schweiz
translationSourceHash: 60de430124ebe20a3001bd48fc55affcb921141039e7c85303da9c7216b01e89
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:23:52.515Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/s-mime-med-banker-silver-eller-gold-sertifikat-fra-swisssign
---

# S/MIME i banker: Rettslig situasjon i Sveits og valget mellom Silver og Gold

Kort oppsummert: Ingen sveitsisk forskrift pålegger en bank å bruke S/MIME eller et bestemt sertifikatnivå. Bankkundetaushetsplikten, FINMA-rundskriv 2023/1 og personvernloven krever imidlertid at kundedata beskyttes på en passende måte ved utsending, og banken må kunne begrunne og dokumentere løsningen sin. En S/MIME-signatur erstatter ikke en underskrift, heller ikke med et Gold-sertifikat. Som bevismiddel og dokumentasjon på opprinnelse er Gold klart bedre enn Silver-sertifikatet, fordi bare Gold inneholder bankens verifiserte navn.

Spørsmålet om hvilket SwissSign-sertifikat en bank trenger for e-postutsending, oppstår jevnlig i prosjektene mine med finansinstitusjoner. Bak det tekniske spørsmålet ligger det vanligvis et rettslig spørsmål: Hva må banken gjøre for å oppfylle pliktene sine ved sending av kundedata via e-post, og hvilken virkning har en signert e-post?

**Fokus på Sveits:** Denne artikkelen omhandler rettstilstanden i Sveits. Der EU-reglene avviker, er det angitt med en merknad. Den erstatter ikke juridisk rådgivning i enkelttilfeller.

## Oversikt over rettsgrunnlagene

| Regelverk | Innhold | Betydning for e-postutsending |
|---|---|---|
| BankG art. 47 | Bankkundetaushetsplikt, straffbar også ved uaktsomhet | Kundedata må ikke røpes uten tillatelse |
| FINMA-RS 2023/1, randnummer 73 til 82 | Beskyttelse av kritiske data gjennom hele livssyklusen | Beskyttelse også ved utsending, teknologinøytralt |
| DSG art. 8 og 24, DSV art. 1 til 6 | Datasikkerhet, melding av brudd | Passende tiltak, melding til EDÖB ved høy risiko |
| ZertES og OR art. 14 nr. 2bis | Typer elektroniske signaturer, formgyldighet | S/MIME-signaturer erstatter ikke en underskrift |
| ZPO art. 157 og 177 | Bevismidler og fri bevisvurdering | Signerte e-poster er dokumenter med bevisverdi som vurderes fritt |
| OR art. 958f, GeBüV | Oppbevaring av forretningsdokumenter | Krypterte e-poster må forbli lesbare |

Ingen av disse regelverkene nevner S/MIME eller et sertifikatnivå. De fastsetter beskyttelsesmål og rettsvirkninger; hvordan banken oppnår dem, avgjør den selv.

## Bankkundetaushetsplikt

Art. 47 BankG straffer den som, som organ, ansatt eller oppdragstaker for en bank, røper en hemmelighet som er blitt betrodd vedkommende i denne egenskapen. Uaktsom handling er også straffbar. Bestemmelsen retter seg mot fysiske personer, men bygger på avtaleforholdet mellom kunden som hemmelighetshaver og banken.

For e-postutsending betyr dette: En ukryptert e-post med kundedata som sendes til feil adresse eller leses underveis, kan være en uaktsom røping. Banken må derfor organisatorisk og teknisk sørge for at ansatte bare sender kundedata på en beskyttet måte.

Kunden kan som hemmelighetshaver samtykke til røpingen. Mange banker lar derfor kunden bekrefte i de generelle vilkårene eller i en separat avtale at kunden kjenner risikoen ved ukryptert kommunikasjon via e-post og ønsker den. Et slikt samtykke dekker bare kommunikasjon med denne kunden og bare i avtalt omfang. For dokumenter som gjelder tredjeparter, eller for kommunikasjon med forretningspartnere, erstatter det ikke kryptering.

## FINMA-rundskriv 2023/1: kritiske data

FINMA-rundskriv 2023/1 «Operasjonelle risikoer og robusthet: Banker» har vært i kraft siden 1. januar 2024 og erstattet rundskriv 2008/21. FINMA har utvidet det tidligere fokuset på konfidensialiteten til kundeidentifikasjonsdata (CID) til kritiske data generelt. Hva som er kritisk, fastsetter hver bank selv basert på risiko; kundedata hører som regel til her når det gjelder konfidensialitet.

For e-postutsending er særlig disse randnumrene relevante:

| Rn | Innhold | Tilknytning til e-postutsending |
|---|---|---|
| 73 | Identifisere kritiske data og kategorisere dem etter kritikalitet | Fastsette hvilke data som bare kan sendes beskyttet |
| 74, 75 | Sikre konfidensialitet, integritet og tilgjengelighet gjennom hele livssyklusen | Utsending er en del av livssyklusen |
| 76 | Beskytte kritiske data mot tilgang fra uvedkommende | Beskyttelse av e-posten under transport og hos mottakeren |
| 79 | Begrense økt risiko ved lagring i eller tilgang fra utlandet | Gjelder gatewayer og e-postplattformer med lokasjon i utlandet |
| 81 | Melde vesentlige hendelser til FINMA uten opphold | For eksempel masseutsending av kundedata til feil mottakere |
| 82 | Velge og overvåke tjenesteleverandører som behandler kritiske data, med omhu | Gjelder tilbydere av krypteringsgatewayer og sertifiseringsinstanser med administrert PKI |

Rundskrivet er teknologinøytralt og nevner verken S/MIME eller sertifikatnivåer. I forklaringene til høringen nevner FINMA datakryptering som et eksempel på særlige tiltak for å sikre kritiske data. Banken må fastsette løsningen sin i egne retningslinjer og der også angi hvilket sertifikatnivå den bruker og hvorfor.

## Personvern: DSG og DSV

Art. 8 DSG forplikter behandlingsansvarlige til å sikre datasikkerhet som er passende for risikoen. Personvernforordningen konkretiserer dette med beskyttelsesmål som konfidensialitet, integritet og sporbarhet, men foreskriver ingen bestemt metode. S/MIME, en gateway med levering via en nettportal eller en utvekslingsplattform oppfyller kravet på samme måte dersom de brukes i samsvar med risikoen.

Dersom et brudd på datasikkerheten sannsynligvis medfører høy risiko for personligheten eller de grunnleggende rettighetene til de berørte personene, må banken melde det til EDÖB så snart som mulig etter art. 24 DSG. Denne meldeplikten kommer i tillegg til meldeplikten overfor FINMA. Hvis e-posten var kryptert, er risikoen ved feilsending som regel lav, og melding faller bort.

## Hva en S/MIME-signatur rettslig innebærer

### Ingen underskrift i ORs forstand

ZertES skiller mellom elektronisk, avansert, regulert og kvalifisert elektronisk signatur, samt det regulerte elektroniske seglet for virksomheter. Etter art. 14 nr. 2bis OR er bare den kvalifiserte elektroniske signaturen med kvalifisert tidsstempel likestilt med håndskrevet underskrift.

S/MIME-sertifikatene fra SwissSign utstedes etter reglene fra CA/Browser Forum og ETSI-policyene LCP (Silver) og NCP (Gold). De er ikke kvalifiserte sertifikater etter ZertES. Banken kan derfor ikke avgi erklæringer som loven eller en avtale krever skriftlig form for, i formgyldig form via S/MIME-signert e-post, heller ikke med Gold.

### Personlig signatur og gateway-signatur

En avansert signatur forutsetter blant annet at signeringsnøkkelen er under den undertegnende personens eneste kontroll. Hvis en gateway sentralt signerer alle utgående e-poster med nøkler som ligger på gatewayen, er denne forutsetningen ikke oppfylt. Signaturen dokumenterer da at e-posten kommer fra bankens system og ikke er endret, men ikke at en bestemt person har godkjent den. Innholdsmessig tilsvarer den derfor heller et organisasjonssegl, uten å oppfylle kravene til et regulert segl etter ZertES.

### Bevisverdi

En e-post er et dokument etter art. 177 ZPO, og retten vurderer bevisverdien fritt etter art. 157 ZPO. En gyldig S/MIME-signatur viser at e-posten ikke er endret siden signeringen, og at den er signert med nøkkelen til et bestemt sertifikat. Hvor mye dette sier om avsenderen, avhenger av hva sertifiseringsinstansen har kontrollert før utstedelsen. Her ligger den rettslig relevante forskjellen mellom Silver og Gold.

## Silver eller Gold fra et rettslig perspektiv

Siden 1. september 2023 gjelder `S/MIME Baseline Requirements` fra CA/Browser Forum for offentlig tillitsverdige S/MIME-sertifikater. SwissSign klassifiserer produktene sine i sertifikatprofilene (CPR S/MIME) under disse valideringstypene:

| Produkt | Type etter CA/B Forum | Hva SwissSign kontrollerer | Hva som står i sertifikatet | Hva signaturen sier |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll over postkassen | Bare e-postadressen | Noen med tilgang til denne postkassen |
| E-Mail ID Gold | `Sponsor-validated` | Organisasjon og person | Navn, organisasjon, organisasjonsidentifikator, land, e-postadresse | En kontrollert person i denne banken |
| Organization Validated | `Organization-validated` | Organisasjon | Organisasjonens navn, ingen person | En postkasse hos denne banken |

Kryptografisk er nivåene likeverdige. For kryptering og dermed for beskyttelsespliktene etter bankkundetaushetsplikten, FINMA-rundskrivet og DSG spiller nivået derfor ingen rolle.

For signaturen er forskjellen betydelig. Et Silver-sertifikat får enhver som kan motta e-post på en adresse. Den som registrerer et domene med et lignende navn, for eksempel `musterbank-ch.com` i stedet for `musterbank.ch`, kan dermed sende e-poster som er like gyldig signert som bankens ekte e-poster. I en tvist om en e-post stammer fra banken, dokumenterer en Silver-signatur derfor lite. Et Gold- eller Organization Validated-sertifikat inneholder derimot bankens navn, som er kontrollert av SwissSign. Med Gold kommer i tillegg tilknytningen til en person, hvis identitet banken bekrefter som registrerende instans innenfor rammen av administrert PKI.

## Oppbevaring: OR og GeBüV

Etter art. 958f OR skal forretningsbøker og bokføringsbilag oppbevares i ti år, og forordningen om oppbevaring av forretningsbøker (GeBüV) krever at elektronisk oppbevarte dokumenter kan gjøres lesbare i denne perioden. For banker kommer tilsynsrettslige og avtalefestede oppbevaringsplikter i tillegg. Dette gir to krav til S/MIME-driften:

- Krypterte e-poster må kunne dekrypteres gjennom hele oppbevaringsperioden. Banken må derfor oppbevare de private nøklene til utløpte sertifikater sikkert eller dekryptere e-postene før arkivering.
- Dersom en signatur fortsatt skal tjene som bevis etter mange år, må det fortsatt være mulig å fastslå at sertifikatet var gyldig på signeringstidspunktet. Derfor må kontrollresultatene eller sperreinformasjonen på signeringstidspunktet også arkiveres.

**EU-merknad:** I EU er etter art. 25 nr. 2 eIDAS også bare den kvalifiserte elektroniske signaturen likestilt med håndskrevet underskrift. For finansforetak i EU gjelder DORA-forordningen med egne krav til IKT-risikostyring i tillegg siden 17. januar 2025.

## Anbefaling

- **Fastsett retningslinjer:** Banken fastsetter i en intern retningslinje hvilke data som bare kan sendes kryptert, hvilke metoder som er tillatt, og hvilke sertifikatnivåer den bruker. Dette er grunnlaget for å dokumentere kravene fra FINMA-rundskrivet og DSG.
- **Gold for personlige postkasser, Organization Validated for funksjonspostkasser:** Bare disse nivåene identifiserer banken som avsender i sertifikatet og har dermed en bevisverdi som går utover tilgang til en postkasse. Silver egner seg ikke for kommunikasjon med kunder.
- **Avklar samtykker:** Den som kommuniserer ukryptert med kunder via e-post, trenger deres dokumenterte samtykke. Det erstatter ikke kryptering overfor tredjeparter og forretningspartnere.
- **Ikke send formbundne erklæringer via e-post:** For erklæringer som krever skriftlig form, er det nødvendig med en kvalifisert elektronisk signatur eller en håndskrevet underskrift.
- **Kontroller tjenesteleverandører:** Tilbydere av gatewayer og sertifiseringsinstansens administrerte PKI omfattes av randnummer 82 i FINMA-rundskrivet, og ved lokasjoner i utlandet også av randnummer 79.
- **Arkiver nøkler og kontrolldata:** Private nøkler for dekryptering og dataene for signaturkontroll må være tilgjengelige i hele oppbevaringsperioden.
- **Knytt meldeprosesser sammen:** En feilsending av kundedata kan samtidig utløse melding til FINMA (randnummer 81) og EDÖB (art. 24 DSG). Begge pliktene kan håndteres i én felles hendelsesprosess.

## Kilder

1.  [FINMA-rundskriv 2023/1 «Operasjonelle risikoer og robusthet: Banker»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): Randnummer 73 til 82 om kritiske data, i kraft siden 1. januar 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 med forklaringer](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): revidert tekst med FINMAs uttalelser fra høringen, blant annet om datakryptering som beskyttelsestiltak.

3.  [Bankloven (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 om bankkundetaushetsplikt.

4.  [Swissbanking: Bankhemmelighet og sky, betenkning 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): juridisk betenkning om tolkningen av art. 47 BankG, blant annet om kundens samtykke.

5.  [Forbundsloven om personvern (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 om datasikkerhet, art. 24 om melding av brudd på datasikkerheten.

6.  [Personvernforordningen (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 til 6 med prinsipper og beskyttelsesmål for datasikkerhet.

7.  [Forbundsloven om elektronisk signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): typer elektroniske signaturer og segl, krav til kvalifiserte sertifikater.

8.  [Obligasjonsretten (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 nr. 2bis om likestilling av kvalifisert elektronisk signatur med håndskrevet underskrift, art. 958f om oppbevaring.

9.  [Sivilprosessloven (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 om fri bevisvurdering, art. 177 om dokumentbegrepet.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): sertifikatprofiler fra SwissSign (per 18.09.2025), klassifisering av Silver, Gold og Organization Validated under valideringstypene og Subject-feltene.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): bindende regler for offentlig tillitsverdige S/MIME-sertifikater, definisjon av valideringstypene.
