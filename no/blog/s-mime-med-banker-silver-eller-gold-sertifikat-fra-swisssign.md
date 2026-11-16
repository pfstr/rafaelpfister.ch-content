---
title: "S/MIME for banker: Silver- eller Gold-sertifikat fra SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "En bank som signerer og krypterer kundemeldinger med S/MIME, trenger Gold-sertifikater for personlige postkasser og Organization-Validated-sertifikater for funksjonelle postkasser. Silver krypterer like sterkt, men identifiserer ikke banken som avsender. Hva SwissSign kontrollerer, hva som står i sertifikatet og hvilke krav som gjelder i Sveits."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "9 min lesetid"
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
translationSourceHash: f95529dd7c72d4d783714affbcf18d96e37adf66a52379b94212034dda610b74
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:50:03.515Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/s-mime-med-banker-silver-eller-gold-sertifikat-fra-swisssign
---

# S/MIME for banker: Silver- eller Gold-sertifikat fra SwissSign?

Kort konklusjon: En bank som signerer og krypterer e-poster til kunder og forretningspartnere med S/MIME, bør utstyre personlige postkasser med Gold-sertifikater og funksjonelle postkasser med Organization-Validated-sertifikater fra SwissSign. Silver krypterer like sterkt, men sertifikatet inneholder bare e-postadressen og ikke bankens navn. Verken loven eller FINMA stiller krav til sertifikatnivå.

Spørsmålet om hvilket sertifikatnivå som er nødvendig for e-postsending, dukker regelmessig opp i mine prosjekter med finansinstitusjoner. Det kommer som regel opp i starten av et gateway-prosjekt eller når bedriftskunder krever at konfidensielle dokumenter kun sendes kryptert. For å svare må to funksjoner ved S/MIME skilles fra hverandre: krypteringen, som beskytter innholdet, og signaturen, som dokumenterer overfor mottakeren hvem som har sendt e-posten.

**Fokus på Sveits:** Denne artikkelen omhandler rettstilstanden i Sveits. Der EU-reglene avviker, er dette angitt i en merknad.

## Hva SwissSign kontrollerer for Silver og Gold

Siden 1. september 2023 har `S/MIME Baseline Requirements` fra CA/Browser Forum vært gjeldende for offentlig betrodde S/MIME-sertifikater. De skiller mellom fire valideringstyper, avhengig av hva sertifiseringsinstansen kontrollerer før utstedelse. SwissSign knytter produktene sine til disse typene i de publiserte sertifikatprofilene (CPR S/MIME):

| Produkt | Type etter CA/B Forum | Hva som kontrolleres | Hva som står i sertifikatet |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll over postkassen | bare e-postadressen |
| E-Mail ID Gold | `Sponsor-validated` | Organisasjon og person | Fornavn og etternavn (eller pseudonym), organisasjon, organisasjonsidentifikator, land, e-postadresse |
| Organization Validated | `Organization-validated` | Organisasjon | Organisasjonsnavn, ingen person |

For Silver sender SwissSign en bekreftelseskode til den søkte adressen. Den som kan motta e-posten, får sertifikatet. Personens identitet og organisasjonen blir ikke kontrollert, og adressen står også i sertifikatet som `Common Name`.

Gold hos SwissSign er utelukkende oppført som `Sponsor-validated`. SwissSign kontrollerer organisasjonen, og personen er knyttet til den. Identiteten kontrolleres enten av SwissSign eller av organisasjonen selv som registreringsinstans, vanligvis innenfor rammen av en Managed PKI. For funksjonelle postkasser som `kundendienst@` eller `zahlungsverkehr@` er profilen `Organization-validated` beregnet; der står banken som organisasjon i sertifikatet, men ingen person.

Kryptografisk skiller nivåene seg ikke. Alle profilene bruker RSA-nøkler på minst 2048 bit, nøkkelbruken `digitalSignature` og `keyEncipherment` samt den utvidede bruken `id-kp-emailProtection`.

## Hva forskjellen betyr for en bank som avsender

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Kryptere og dekryptere e-poster | ja | ja | ja |
| Signaturen viser i e-postprogrammet | E-postadresse | Navn og bank | Bank |
| Hva signaturen bekrefter | Avsenderen har tilgang til postkassen | Kontrollert person i denne banken | Postkasse tilhørende denne banken |
| Bruk i en bank | uegnet for kundekommunikasjon | personlige postkasser | funksjonelle postkasser |

Forskjellen gjelder signaturen. En bank signerer utgående e-poster slik at mottakerne kan kontrollere at meldingen faktisk kommer fra banken og ikke har blitt endret underveis. Med Silver er dette bare mulig i begrenset grad:

- Den som registrerer et domene med et lignende navn, for eksempel `musterbank-ch.com` i stedet for `musterbank.ch`, får et Silver-sertifikat for adresser i dette domenet uten ytterligere kontroll. Phishing-e-postene vedkommende sender, er da like gyldig signert som bankens ekte e-poster, og sertifikatet inneholder i begge tilfeller bare en e-postadresse.
- Den som overtar en postkasse hos banken, kan også søke om et Silver-sertifikat for denne adressen.

Et Gold- eller Organization-Validated-sertifikat inneholder bankens kontrollerte navn i feltet `Organization`. Denne oppføringen får bare den som består organisasjonskontrollen hos SwissSign. For mottakere som ser på signaturen, er dette det relevante beviset.

For kryptering spiller nivået ingen rolle. En e-post kryptert med Silver er like godt beskyttet som en som er kryptert med Gold.

## Kryptering forutsetter sertifikater hos mottakerne

For å kryptere en e-post med S/MIME trenger banken mottakerens offentlige nøkkel. Bankens eget sertifikat brukes ikke til dette; det brukes til signering og mottak av krypterte svar.

De fleste privatkunder har ikke et S/MIME-sertifikat. Banker leverer derfor vanligvis konfidensielle e-poster til dem via en krypteringsgateway, for eksempel SEPPmail (GINA) eller Kiteworks/Totemomail, som gjør meldingen tilgjengelig i en webportal eller som et kryptert vedlegg. S/MIME-kryptering brukes først og fremst for bedriftskunder og partnere som selv drifter sertifikater eller en gateway. Hvis begge sider bruker en gateway, er domenesertifikater eller tvungen TLS mellom e-postserverne ofte tilstrekkelig.

## Sertifikater i drift: Managed PKI og gateway

Ved mer enn en håndfull postkasser innhenter en bank sertifikatene via SwissSigns Managed PKI. Banken opptrer da som registreringsinstans og bekrefter selv identiteten til medarbeiderne; dette tilsvarer typen `Sponsor-validated`. Gatewayer som SEPPmail og Kiteworks/Totemomail kan via et grensesnitt mot Managed PKI automatisk søke om sertifikater, fornye dem og knytte dem til postkassene.

Tre forhold fra prosjekter er kjent for å skape problemer i driften:

- Gold-søknader for funksjonelle postkasser mislykkes fordi gatewayen henter fornavn og etternavn fra katalogen, og SwissSign avviser generiske betegnelser som «Support» eller «Helpdesk» i feltet `GivenName`. For slike postkasser kan profilen `Organization-validated` lagres i gatewayen.
- Automatiske fornyelser kan mislykkes i ukevis uten at det oppdages, helt til de første sertifikatene utløper og kundene ikke lenger kan sende krypterte e-poster. Utløpsdatoene for utstedte sertifikater og feilmeldingene fra fornyelseskjøringene bør derfor overvåkes.
- Gatewayen signerer søknadene til Managed PKI med et eget sertifikat for registreringsinstansen. Når dette utløper, stopper hele automatiseringen.

## Rettslige krav i Sveits

Ingen sveitsisk lov krever S/MIME, og ingen skiller mellom sertifikatnivåer. Bankens plikter gjelder beskyttelsen av kundedata; hvordan den gjennomfører dette, avgjør den selv.

| Regelverk | Innhold | Betydning for bankens e-postsending |
|---|---|---|
| BankG art. 47 | Bankkundekonfidensialitet | Kundedata må ikke avsløres uberettiget; ukryptert sending over internett er en risiko banken må vurdere |
| FINMA-RS 2023/1 | Beskyttelse av kritiske data | Beskyttelse også ved overføring, teknologinøytral, ingen krav til sertifikater |
| DSG art. 8 og DSV art. 1 til 6 | Datasikkerhet tilpasset risikoen | Kryptering er et vanlig virkemiddel, men en bestemt metode er ikke foreskrevet |
| ZertES og OR art. 14 avsn. 2bis | Bare den kvalifiserte elektroniske signaturen likestilles med egenhendig underskrift | S/MIME-signaturer erstatter ikke en underskrift |

### Bankkundekonfidensialitet og FINMA

Bankkundekonfidensialiteten etter art. 47 BankG forplikter banken, dens organer og medarbeidere til ikke å avsløre kundedata uberettiget. FINMA-rundskriv 2023/1 «Operasjonelle risikoer og resiliens: Banker» krever at banker beskytter sine kritiske data gjennom hele livssyklusen, også ved overføring. Hvordan dette skjer, fastsetter banken i egne retningslinjer. Valget av sertifikatnivå er dermed en del av den interne policyen og bør dokumenteres der.

### Personvern: DSG og DSV

Art. 8 DSG forplikter behandlingsansvarlige til å sikre datasikkerhet som er tilpasset risikoen. Personvernforordningen konkretiserer dette med beskyttelsesmål som konfidensialitet og integritet, men angir ingen metoder. S/MIME, en gateway med levering via webmail eller en utvekslingsplattform oppfyller dette kravet på samme måte, forutsatt at de brukes på en måte som er tilpasset risikoen.

### Elektronisk signatur: ZertES og OR

ZertES skiller mellom elektronisk, avansert, regulert og kvalifisert elektronisk signatur. Etter art. 14 avsn. 2bis OR er bare den kvalifiserte elektroniske signaturen med kvalifisert tidsstempel likestilt med egenhendig underskrift. S/MIME-sertifikatene fra SwissSign utstedes etter reglene til CA/Browser Forum og ETSI-policyene LCP (Silver) og NCP (Gold), og er ikke kvalifiserte sertifikater etter ZertES. Avtaler eller erklæringer som krever skriftlig form, kan ikke avgis formgyldig med en S/MIME-signert e-post, heller ikke med Gold.

**EU-merknad:** I EU er etter art. 25 avsn. 2 eIDAS også bare den kvalifiserte elektroniske signaturen likestilt med håndskrevet underskrift.

## Anbefaling

- **Personlige postkasser** for kunderådgivning og andre medarbeidere med kundekontakt: Gold (`Sponsor-validated`) via Managed PKI.
- **Funksjonelle postkasser:** `Organization-validated`, slik at banken står som avsender i sertifikatet.
- **Silver** bør ikke brukes til kundekommunikasjon, fordi signaturen ikke identifiserer banken som avsender.
- **Kryptering til privatkunder** via gatewayens push- eller portallevering; S/MIME for mottakere med eget sertifikat eller gateway.
- **Signatur og avsenderkontroll** utfyller hverandre: S/MIME-signaturen erstatter ikke SPF, DKIM og DMARC, men viser mottakeren den kontrollerte organisasjonen i e-postprogrammet.
- **Informasjonsark for kunder:** Beskriv hvilke sertifikater banken bruker, hvordan kunder kontrollerer en signatur og hvilke sertifikatnivåer banken aksepterer fra kunder som selv vil sende kryptert. Hvis et slikt informasjonsark mangler, går spørsmålet til kunderådgivningen, som som regel ikke kan besvare det.

## Kilder

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Sertifikatprofiler fra SwissSign (per 18.09.2025), tilordning av Silver, Gold og Organization Validated til valideringstypene, Subject-felt og Policy-OID-er.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): bindende regler for offentlig betrodde S/MIME-sertifikater, definisjon av de fire valideringstypene.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): Tilkobling av SEPPmail-gatewayen til SwissSigns Managed PKI.

4.  [Bankengesetz (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 om bankkundekonfidensialitet.

5.  [FINMA: Rundschreiben](https://www.finma.ch/de/dokumentation/rundschreiben/): Oversikt over rundskrivet 2023/1 «Operasjonelle risikoer og resiliens: Banker», i kraft siden 1. januar 2024.

6.  [Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 om datasikkerhet.

7.  [Datenschutzverordnung (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 til 6 med prinsipper og beskyttelsesmål for datasikkerhet.

8.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Typer elektroniske signaturer og krav til kvalifiserte sertifikater.

9.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 avsn. 2bis om likestilling av kvalifisert elektronisk signatur med egenhendig underskrift.
