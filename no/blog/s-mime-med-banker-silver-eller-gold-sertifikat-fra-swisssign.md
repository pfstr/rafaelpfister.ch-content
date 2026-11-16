---
title: "S/MIME med banker: Silver- eller Gold-sertifikat fra SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "For krypterte e-poster med en bank er et Silver-sertifikat fra SwissSign teknisk sett tilstrekkelig. Gold bekrefter i tillegg avsenderens navn og organisasjon. Hvilket nivå som kontrollerer hva, hva som står i sertifikatet og hvilke juridiske krav som gjelder i Sveits."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "8 min lesetid"
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
url: https://rafaelpfister.ch/no/blog/s-mime-med-banker-silver-eller-gold-sertifikat-fra-swisssign
translationSourceHash: cbcb890de323f2b9db40a5682192296524250e7bc8c23a67c6f643bcba605a14
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:05:41.639Z
translationReview: automatic
---

# S/MIME med banker: Silver- eller Gold-sertifikat fra SwissSign?

Kort konklusjon: Verken sveitsisk lov eller FINMA krever et bestemt sertifikatnivå for S/MIME. For kryptering er et Silver-sertifikat teknisk sett fullt tilstrekkelig. Gold bekrefter i tillegg avsenderens navn og organisasjon, og er det riktige valget dersom banken ikke stiller krav.

Spørsmålet om et Silver- eller Gold-sertifikat fra SwissSign er nødvendig for e-postkommunikasjon med en bank, dukker jevnlig opp i prosjektene mine. Som regel finnes det ikke noe informasjonsskriv fra banken, og kundekonsulenten kan ikke svare på spørsmålet. Svaret avhenger av hva sertifikatet brukes til: kryptering eller dokumentasjon på hvem som har sendt en e-post.

**Sveits-fokus:** Denne artikkelen omhandler rettstilstanden i Sveits. Der EU-reglene avviker, finnes det en merket merknad.

## Hva SwissSign kontrollerer for Silver og Gold

Siden 1. september 2023 har `S/MIME Baseline Requirements` fra CA/Browser Forum vært gjeldende for offentlig pålitelige S/MIME-sertifikater. De skiller mellom fire valideringstyper, avhengig av hva sertifiseringsinstansen kontrollerer før utstedelse. SwissSign tilordner produktene sine til disse typene i de publiserte sertifikatprofilene (CPR S/MIME):

| Produkt | Type etter CA/B Forum | Hva som kontrolleres | Hva som står i sertifikatet |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll over postkassen | kun e-postadressen |
| E-Mail ID Gold | `Sponsor-validated` | Organisasjon og person | Fornavn og etternavn (eller pseudonym), organisasjon, organisasjonsidentifikator, land, e-postadresse |
| Organization Validated | `Organization-validated` | Organisasjon | Organisasjonsnavn, ingen person |

For Silver sender SwissSign en bekreftelseskode til den søkte adressen. Den som kan motta e-posten, får sertifikatet. Personens identitet kontrolleres ikke, og adressen står også i sertifikatet som `Common Name`.

Gold er hos SwissSign utelukkende oppført som `Sponsor-validated`. Organisasjonen kontrolleres, og personen knyttes til den: Identiteten kontrolleres enten av SwissSign eller av organisasjonen selv som registreringsinstans, for eksempel innenfor en Managed PKI. En profil for privatpersoner uten organisasjon (`Individual-validated`) er ikke oppført i sertifikatprofilene.

For funksjonspostkasser som `support@` eller `buchhaltung@` er profilen `Organization-validated` beregnet. Gold-søknader for slike postkasser kan mislykkes fordi SwissSign avviser generiske betegnelser som «Support» eller «Helpdesk» i feltet `GivenName`.

Kryptografisk er det ingen forskjell mellom nivåene. Alle profilene bruker RSA-nøkler fra 2048 bit, nøkkelbruken `digitalSignature` og `keyEncipherment` samt den utvidede bruken `id-kp-emailProtection`. En e-post kryptert med Silver er like godt beskyttet som en med Gold.

## Hva forskjellen betyr i praksis

| | Silver | Gold |
|---|---|---|
| Kryptere og dekryptere e-poster | ja | ja |
| Visning av signaturen i Outlook | E-postadresse | Navn og organisasjon |
| Hva signaturen bekrefter | Avsenderen har tilgang til postkassen | Avsenderen er en kontrollert person i denne organisasjonen |
| Arbeid ved søknad | Bekreftelse per e-post | Dokumentasjon av organisasjon og identitetskontroll |

Forskjellen blir tydelig ved overtakelse av en konto. Den som kontrollerer en postkasse, kan søke om et Silver-sertifikat for denne adressen og dermed sende gyldig signerte e-poster. Et Gold-sertifikat forutsetter derimot at organisasjonen og personen er kontrollert. For en bank som for eksempel mottar betalingsoppdrag eller kontraktskorrespondanse per e-post, har en Gold-signatur derfor større utsagnskraft.

For kryptering spiller dette ingen rolle. Banken trenger bare mottakerens offentlige nøkkel, og den er likeverdig på begge nivåer.

## Juridiske krav i Sveits

Ingen sveitsisk lov krever S/MIME, og ingen skiller mellom sertifikatnivåer. Det finnes generelle plikter til å beskytte data, mens den tekniske gjennomføringen er åpen.

| Regelverk | Innhold | Tilknytning til S/MIME |
|---|---|---|
| DSG art. 8 og DSV art. 1 til 6 | Datasikkerhet som er tilpasset risikoen | Kryptering er et vanlig tiltak for sensitive personopplysninger, men en bestemt metode er ikke foreskrevet |
| BankG art. 47 | Bankkundekonfidensialitet | Forplikter banken, ikke kundene dens |
| FINMA-RS 2023/1 | Beskyttelse av kritiske data i banker | Teknologinøytral, ingen krav til sertifikater |
| ZertES og OR art. 14 nr. 2bis | Bare den kvalifiserte elektroniske signaturen er likestilt med håndskrevet underskrift | Silver og Gold er ikke kvalifiserte signaturer |
| VeÜ-ZSSV | Elektroniske innleveringer til domstoler og inkassomyndigheter | Kvalifisert signatur og anerkjent leveringsplattform kreves, S/MIME er ikke tilstrekkelig |

### Personvern: DSG og DSV

Art. 8 DSG forplikter behandlingsansvarlige og databehandlere til å sikre datasikkerhet som er tilpasset risikoen. Personvernforordningen konkretiserer dette med beskyttelsesmål som konfidensialitet og integritet, men nevner ingen metoder. Om man bruker S/MIME, en gateway med webmail-levering eller en utvekslingsplattform, er en vurdering i hvert enkelt tilfelle.

### Bankkundekonfidensialitet og FINMA

Bankkundekonfidensialiteten etter art. 47 BankG og FINMA-rundskriv 2023/1 «Operasjonelle risikoer og robusthet: Banker» retter seg mot banken. Rundskrivet krever at bankene beskytter sine kritiske data gjennom hele livssyklusen, også ved overføring. Hvordan dette gjennomføres, fastsetter bankene i egne retningslinjer. Om Silver aksepteres, er derfor et spørsmål om den enkelte bankens interne policy.

### Elektronisk signatur: ZertES og OR

ZertES skiller mellom elektronisk, avansert, regulert og kvalifisert elektronisk signatur. Etter art. 14 nr. 2bis OR er bare den kvalifiserte elektroniske signaturen med kvalifisert tidsstempel likestilt med håndskrevet underskrift. S/MIME-sertifikatene fra SwissSign utstedes etter reglene til CA/Browser Forum og ETSI-policyene LCP (Silver) og NCP (Gold), og er ikke kvalifiserte sertifikater etter ZertES.

En avtale som krever skriftlig form, kan derfor ikke inngås formgyldig med en S/MIME-signert e-post, heller ikke med Gold. Ved tvist kan en Gold-signatur likevel brukes som bevis; retten vurderer bevisverdien fritt.

### Innleveringer til domstoler og myndigheter

For elektroniske innleveringer i sivile saker, straffesaker samt gjeldsinndrivelses- og konkurssaker krever VeÜ-ZSSV en kvalifisert elektronisk signatur og overføring via en anerkjent leveringsplattform. En S/MIME-e-post oppfyller ikke disse kravene.

**EU-merknad:** I EU er det etter art. 25 nr. 2 eIDAS også bare den kvalifiserte elektroniske signaturen som er likestilt med håndskrevet underskrift.

## Når banken ikke stiller krav

Hvis det mangler et informasjonsskriv, kan situasjonen avklares med tre spørsmål til banken:

1.  Krever banken et sertifikat med kontrollert identitet for signerte e-poster, eller er ren e-postvalidering tilstrekkelig?

2.  Bruker banken en krypteringsgateway (f.eks. SEPPmail eller Kiteworks/Totemomail) og et domene-sertifikat, eller personlige sertifikater for sine ansatte?

3.  Hvordan gjør banken sine egne offentlige nøkler tilgjengelige?

En kort forespørsel til kundeservice, som kan videresendes til IT-avdelingen:

> For kryptert e-postkommunikasjon med dere anskaffer jeg et S/MIME-sertifikat. Vennligst opplys om dere krever et sertifikat med kontrollert identitet (f.eks. SwissSign E-Mail ID Gold med navn og organisasjon), eller om et sertifikat med bare e-postvalidering (Silver) er tilstrekkelig. Jeg ber også om det offentlige sertifikatet deres, eventuelt informasjon om dere bruker en gateway med domene-sertifikat.

Mange banker sender uansett konfidensielle e-poster via en gateway med webmail-levering. Mottakeren logger seg da inn i portalen og trenger ikke eget sertifikat. Hvis begge sider bruker en gateway, er domene-sertifikater eller tvungen TLS mellom e-postserverne ofte tilstrekkelig, og personlige sertifikater er ikke nødvendig.

Hvis spørsmålet forblir uavklart, er Gold det riktige valget for bedriftskorrespondanse. Gold dekker kryptering og signatur, og et senere bytte fra Silver til Gold innebærer en ny søknad og ny distribusjon av den offentlige nøkkelen til alle motparter. Silver er tilstrekkelig hvis banken bekrefter at den bare krypterer og ikke evaluerer signaturer. For funksjonspostkasser er et `Organization-validated`-sertifikat beregnet.

## Kilder

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Sertifikatprofiler fra SwissSign (status 18.09.2025), tilordning av Silver, Gold og Organization Validated til valideringstypene, Subject-felter og Policy-OID-er.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Bindende regler for offentlig pålitelige S/MIME-sertifikater, definisjon av de fire valideringstypene.

3.  [Forbundslov om personvern (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 om datasikkerhet.

4.  [Personvernforordning (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 til 6 med prinsipper og beskyttelsesmål for datasikkerhet.

5.  [Bankloven (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 om bankkundekonfidensialitet.

6.  [FINMA: Rundskriv](https://www.finma.ch/de/dokumentation/rundschreiben/): Oversikt over rundskrivet 2023/1 «Operasjonelle risikoer og robusthet: Banker», i kraft siden 1. januar 2024.

7.  [Forbundslov om elektronisk signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Typer elektroniske signaturer og krav til kvalifiserte sertifikater.

8.  [Obligasjonsretten (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 nr. 2bis om likestilling av kvalifisert elektronisk signatur med håndskrevet underskrift.
