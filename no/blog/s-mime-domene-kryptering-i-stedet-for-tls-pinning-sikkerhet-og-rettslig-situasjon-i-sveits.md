---
title: "S/MIME-domene-kryptering i stedet for TLS-pinning: Sikkerhet og rettslig situasjon i Sveits"
navTitle: "Domene i stedet for pinning"
description: "Hvorfor S/MIME-domene-kryptering beskytter e-posttrafikken mellom to organisasjoner bedre enn TLS-pinning, hva en angriper kan gjøre med en oppfanget S/MIME-melding, hvordan DSG, DSV og taushetsplikter påvirker valget, og hvorfor MTA-STS og DANE tar seg av transportbeskyttelsen."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "10 min. lesetid"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-domene-kryptering-i-stedet-for-tls-pinning-sikkerhet-og-rettslig-situasjon-i-sveits"
translationId: "article-92db7a4587660dec"
translationOf: domain-verschluesselung-statt-tls-pinning
url: https://rafaelpfister.ch/no/blog/s-mime-domene-kryptering-i-stedet-for-tls-pinning-sikkerhet-og-rettslig-situasjon-i-sveits
translationSourceHash: 8eeb8d5552f8680381710c1a052edd3e62b69b34e1af9fd032c4bd481e8d65a1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T16:01:12.102Z
translationReview: required
---

# S/MIME-domene-kryptering i stedet for TLS-pinning: Sikkerhet og rettslig situasjon i Sveits

Kort oppsummert: TLS-pinning beskytter forbindelsen mellom to e-postservere, mens S/MIME-domene-kryptering beskytter meldingens innhold hele veien mellom to organisasjoner. Dersom man må velge mellom dem, er domene-kryptering det beste valget teknisk og rettslig sett. MTA-STS og DANE overtar transportbeskyttelsen som pinning hittil har gitt.

I et aktuelt prosjekt skal en eksisterende e-postforbindelse mellom to organisasjoner, som hittil er sikret med påtvunget TLS og sertifikat-pinning, legges om til S/MIME-domene-kryptering via en sikker e-posttjeneste. Tjenesten kan ikke bruke begge deler samtidig for den samme forbindelsen: Enten gjelder pinning eller domene-kryptering. Spørsmålet om dette fører til tap av sikkerhet, er berettiget. Svaret er entydig når man ser på hva de to metodene beskytter.

**Sveits-fokus:** Den rettslige vurderingen følger sveitsisk rett, særlig DSG, DSV og StGB. Der EU-reglene avviker, gis en merket merknad. Artikkelen erstatter ikke juridisk rådgivning i enkelttilfeller.

## Hva TLS-pinning beskytter

Uten tilleggstiltak er TLS mellom e-postservere opportunistisk. Dette gir tre angrepsmuligheter:

- **Nedgradering:** En angriper i overføringsveien fjerner STARTTLS-tilbudet fra SMTP-økten. Den sendende serveren leverer da i klartekst.
- **Vilkårlig sertifikat:** Sendende e-postservere kontrollerer som standard ikke sertifikatet til motparten. En server med et selvutstedt sertifikat godtas.
- **Forfalsket MX:** Den som manipulerer DNS-svaret, videresender e-posten til sin egen server, som tilbyr korrekt TLS.

Pinning lukker alle tre hullene for en bestemt partner. Den sendende gatewayen håndhever TLS og godtar bare et forhåndsdefinert sertifikat eller en forhåndsdefinert nøkkel. Beskyttelsen opphører imidlertid med forbindelsen. På mottaksserveren, i køer, spam- og virusfiltre, journalførings- og arkivsystemer foreligger meldingen i klartekst. Dersom den går gjennom flere servere, gjelder pinning bare for strekningen den er konfigurert for.

I tillegg kommer driften. Pinnen er en manuell regel hos avsenderen. Dersom mottakeren fornyer sertifikatet sitt eller endrer rutingen, for eksempel med en ny filtertjeneste eller en backup-MX, stopper leveringen til regelen er oppdatert. Ved driftsforstyrrelser blir regelen derfor ofte midlertidig lempet på, og ikke alltid aktivert igjen.

## Hva S/MIME-domene-kryptering beskytter

Ved domene-kryptering krypterer avsenderens gateway meldingen med den offentlige nøkkelen fra mottakerdomenets domenesertifikat. Først mottakerens gateway dekrypterer den igjen. Alle systemer imellom ser bare det krypterte innholdet, uavhengig av hvor mange servere meldingen passerer og hvor den mellomlagres. I tillegg signerer avsenderens gateway meldingen med sitt eget domenesertifikat.

| Beskyttelse | TLS-pinning | S/MIME-domene-kryptering |
|---|---|---|
| E-posttekst og vedlegg | bare under overføring på den pinnede strekningen | hele veien, også på mellomservere og i køer |
| Emne, avsender, mottaker, dato | kryptert under overføring | lesbart, så lenge headerbeskyttelse ikke er aktiv |
| Konvolutt, størrelse, tidspunkt | kryptert under overføring | lesbart |
| Autentisitet for mottaksserveren | ja, feil server avvises | nei, e-posten overleveres til serveren som MX-rutingen angir |
| Forward Secrecy | ja, med ECDHE-nøkkelutveksling | nei |
| Integritet og opprinnelse etter levering | nei | ja, ved signatur |
| Avhengighet av ruting og infrastruktur | høy, pinnen gjelder én server eller nøkkel | lav, beskyttelsen er knyttet til domenesertifikatet |

Pinning har dermed tre egenskaper som domene-kryptering ikke tilbyr: Den beskytter metadataene, forhindrer at en feil server mottar e-posten, og tilbyr Forward Secrecy. For innholdet er domene-kryptering klart overlegent.

## Hva en angriper kan gjøre med en oppfanget S/MIME-melding

Anta at en angriper får sin egen e-postserver inn i overføringsveien, via forfalskede MX-oppføringer, BGP-hijacking eller et kompromittert relé.

Dette kan han ikke:

- **Lese innholdet:** Tekst og vedlegg er kryptert med mottakerdomenets offentlige nøkkel. Uten den private nøkkelen til mottakergatewayen forblir de uleselige.
- **Endre innholdet ubemerket:** Hvis meldingen er signert og mottakergatewayen krever en gyldig signatur, blir enhver endring oppdaget.
- **Opprette meldinger i avsenderens navn:** Til dette mangler han avsenderdomenets private signeringsnøkkel.

Dette kan han:

- **Analysere metadata:** Avsender, mottaker, tidspunkt, størrelse og som regel emnet er lesbare. Et emne som «Utskrivningsrapport Max Muster» er allerede en utlevering av personopplysninger.
- **Lagre meldinger og dekryptere dem senere:** S/MIME tilbyr ikke Forward Secrecy. Hvis den private domenenøkkelen senere kompromitteres, kan alle lagrede meldinger leses. Det samme gjelder på lang sikt dersom RSA blir angripbart med kvantedatamaskiner.
- **Undertrykke eller forsinke meldinger:** Avsenderen vil ikke nødvendigvis merke dette. Tilgjengeligheten påvirkes, noe som er relevant ved frister.
- **Endre ubeskyttede headere:** Han kan for eksempel omskrive emnet eller `Reply-To` for å omdirigere svar. Det krypterte innholdet forblir uendret.
- **Manipulere chiffertekst målrettet:** For meldinger som bare er kryptert og ikke signert, og som bruker AES-CBC, kan chifferteksten endres slik at deler av klarteksten kan lekke etter dekryptering. Dette er prinsippet bak EFAIL-angrepene fra 2018. Obligatorisk signaturkontroll eller autentisert kryptering med AES-GCM etter S/MIME 4.0 forhindrer dette.

Oppfanget klartekst ved ren transportkryptering er en fullstendig datalekkasje. En oppfanget S/MIME-melding gjelder i hovedsak metadata og tilgjengelighet.

## Rettslig vurdering

Ingen sveitsisk forskrift pålegger S/MIME, TLS-pinning eller en annen bestemt metode. Kravene er teknologinøytrale og fastsetter beskyttelsesmål. Valget av metode må imidlertid vurderes opp mot disse beskyttelsesmålene og dokumenteres.

| Regelverk | Innhold | Betydning for valg av metode |
|---|---|---|
| DSG art. 8, DSV art. 1 til 3 | tilstrekkelig datasikkerhet, beskyttelsesmål, transportkontroll | beskyttelse ved utlevering, målt etter risiko og teknisk nivå |
| DSG art. 61 bokstav c | bot ved brudd på minstekravene til datasikkerhet | personlig straffansvar for den ansvarlige personen |
| DSG art. 9 | behandling ved databehandler | hvert system med tilgang til klartekst må tas med i betraktningen |
| DSG art. 16 og 17 | utlevering til utlandet | behandling av klartekst på utenlandske systemer |
| DSG art. 24 | melding av brudd på datasikkerheten | risikoen for berørte avhenger av om lekkede data er lesbare |
| StGB art. 321, DSG art. 62, ATSG art. 33 | profesjonshemmelighet, yrkesmessig taushetsplikt, taushetsplikt i sosialforsikringene | innholdet må beskyttes mot enhver uautorisert kjennskap |
| ZPO art. 177 | elektroniske filer som dokumenter | signatur styrker beviset for opprinnelse og integritet |

### Datasikkerhet etter DSG og DSV

Art. 8 DSG krever datasikkerhet som står i forhold til risikoen. DSV konkretiserer beskyttelsesmålene konfidensialitet, tilgjengelighet, integritet og sporbarhet og nevner i art. 3 blant annet transportkontroll som tiltak: Personopplysninger må ikke kunne leses, kopieres, endres eller slettes uautorisert ved utlevering. For særlig beskyttelsesverdige personopplysninger etter art. 5 bokstav c DSG, som helseopplysninger eller opplysninger om tiltak innen sosialhjelp, gjelder tilsvarende høyere krav.

TLS-pinning oppfyller transportkontrollen for én del av overføringen. Meldingen lagres imidlertid på flere systemer før den når målet, og der kan den leses i klartekst. Med domene-kryptering er innholdet beskyttet på hele strekningen mellom de to organisasjonene. Signaturen dekker i tillegg beskyttelsesmålet integritet, som TLS ikke lenger kan dokumentere etter levering.

### Personlig straffansvar

Etter art. 61 bokstav c DSG straffes den som forsettlig ikke overholder minstekravene til datasikkerhet etter art. 8 tredje ledd DSG, etter begjæring, med bot på inntil CHF 250'000. Boten rettes mot den ansvarlige fysiske personen, ikke mot virksomheten. Innholdskryptering i tråd med gjeldende teknisk nivå, dokumentert i en risikovurdering, er et sterkere bevis for disse personene enn en transportregel for enkeltservere.

**EU-merknad:** GDPR krever også egnede tekniske og organisatoriske tiltak i art. 32 og nevner uttrykkelig kryptering. Bøter etter art. 83 GDPR rettes imidlertid mot virksomheten og utgjør ved brudd på art. 32 inntil 10 millioner euro eller 2 % av global årlig omsetning.

### Databehandlere og utlevering til utlandet

Hvert system på veien som behandler klartekst, må vurderes rettslig: filtertjenester, reléer, sky-gatewayer, arkiveringstjenester. Hvis tredjeparter drifter disse systemene, er det som regel databehandling på oppdrag etter art. 9 DSG, med tilhørende avtaler og kontrollplikter. Dersom systemene befinner seg i utlandet, kommer reglene om utlevering til utlandet etter art. 16 og 17 DSG i tillegg.

Med domene-kryptering behandler disse systemene bare chiffertekst. Kretsen av instanser som kan få tilgang til innholdet, begrenses til de to gatewayene. Metadataene forblir imidlertid personopplysninger, og de nevnte pliktene gjelder fortsatt for dem.

### Meldeplikt ved brudd på datasikkerheten

Et brudd på datasikkerheten som sannsynligvis fører til høy risiko for de berørte personene, må meldes til EDÖB etter art. 24 DSG. Dersom meldinger lekker via en kompromittert mellomserver, avhenger vurderingen av om innholdet er lesbart. Hvis det er sterkt kryptert og nøklene ikke er berørt, er risikoen for de berørte som regel lav, forutsatt at emnet ikke inneholder sensitive opplysninger. Ved ren transportkryptering foreligger innholdet i klartekst for angriperen.

### Profesjonshemmeligheter og taushetsplikter

Der opplysninger er underlagt profesjonshemmelighet etter art. 321 StGB, for eksempel hos leger, advokater eller deres hjelpere, eller en lovfestet taushetsplikt som art. 33 ATSG på sosialforsikringsområdet, må innholdet beskyttes mot enhver uautorisert kjennskap. Dette omfatter også driftsansvarlige for mellomsystemer. Art. 62 DSG gjør dessuten brudd på yrkesmessig taushetsplikt straffbart for personer som ikke faller inn under art. 321 StGB. Dette kravet oppfylles bare ved kryptering av innholdet.

### Signaturens bevisverdi

Etter art. 177 ZPO er elektroniske filer dokumenter, og retten vurderer deres beviskraft fritt. En gyldig S/MIME-signatur fra avsenderdomenet dokumenterer at meldingen stammer fra dette domenet og ikke er endret etter signeringen. Det styrker beviset ved frister, tilsagn eller instrukser. Den erstatter ikke en underskrift; formkravene i OR oppfylles bare av en kvalifisert elektronisk signatur etter ZertES. Med TLS-pinning kan verken opprinnelse eller integritet dokumenteres etter levering.

## MTA-STS og DANE som erstatning for TLS-pinning

Når pinning faller bort, mangler egenskapene som bare transportbeskyttelsen tilbyr: påtvunget TLS, en autentisert mottaksserver og dermed beskyttelse av metadata mot avlytting og omdirigering. Nettopp disse egenskapene leverer MTA-STS og DANE, som standarder mottakeren publiserer én gang for alle avsendere.

**MTA-STS (RFC 8461):** Mottakeren publiserer under `https://mta-sts.<domain>/.well-known/mta-sts.txt` en policy med tillatte MX-vertsnavn og modusen `enforce`. En avsender som støtter MTA-STS, henter policyen, lagrer den og leverer deretter bare via TLS med gyldig sertifikat til en av de oppførte MX-ene. En forfalsket MX eller en server uten gyldig sertifikat for MX-navnet avvises.

**DANE (RFC 7672):** Mottakeren publiserer en TLSA-oppføring med hashen av sertifikatet eller nøkkelen i sin DNSSEC-signerte sone. Avsenderen kontrollerer MX-sertifikatet mot denne oppføringen. Teknisk sett er DANE pinning som mottakeren selv publiserer i DNS og sikrer med DNSSEC.

| | TLS-pinning | MTA-STS | DANE |
|---|---|---|---|
| Vedlikeholdes av | avsender, manuelt per partner | mottaker, én gang for alle avsendere | mottaker, én gang for alle avsendere |
| Virker for | én konfigurert partner | alle avsendere med MTA-STS-støtte | alle avsendere med DANE-støtte |
| Sertifikatfornyelse | pinnen må oppdateres hos avsenderen | uten tilpasning, så lenge det nye sertifikatet er gyldig | uten tilpasning med samme nøkkel, ellers må ny TLSA-oppføring publiseres på forhånd |
| Tillitsgrunnlag | nøyaktig dette sertifikatet eller denne nøkkelen | offentlige sertifiseringsinstanser | DNSSEC |
| Svakhet | driftsarbeid, lemping ved forstyrrelser | første henting og utløpt cache (Trust on First Use), feilutstedelse av en CA | forutsetter DNSSEC |
| Feilrapporter | ingen | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

For dokumentasjonen etter art. 8 DSG har MTA-STS og DANE en ytterligere fordel: Via TLS-RPT mottar mottakeren daglige rapporter om leveringer ble kryptert eller mislyktes. Denne tilbakemeldingen finnes ikke ved pinning. Oppsettet av begge metodene beskrives i artikkelen [DNS-oppføringer for e-post](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) i avsnittet om MTA-STS, TLS-RPT og DANE.

## Hvem som må kontrollere MTA-STS og DANE

Det er alltid systemet som leverer meldingen direkte til mottakerens MX, som må kontrollere. Det er operatøren av MX-en som må publisere oppføringene.

For en sikker e-posttjeneste ser den utgående veien typisk slik ut:

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Den siste strekningen har hittil vært beskyttet av pinning. Her må tjenesten kontrollere partnerens MTA-STS og DANE; den egne e-postserveren har ingen innflytelse på denne strekningen. Strekningen fra den egne e-postserveren til tjenesten sikrer dere selv, med påtvunget TLS og sertifikatkontroll på connectoren.

På mottakersiden avhenger det av MX-en. Dersom MX-en peker på tjenesten, må tjenesten stille med passende sertifikater, og dere publiserer MTA-STS- og TLSA-oppføringer som passer til serverne deres. For DANE forutsetter dette at tjenesten varsler nøkkelbytter i god tid. Dersom MX-en peker på den egne gatewayen, for eksempel Exchange Online, setter dere opp MTA-STS og DANE der. Hvis begge organisasjonene bruker samme tjeneste, kan denne levere meldingen internt på sin egen plattform; da finnes det ingen offentlig SMTP-strekning mellom de to organisasjonene.

Hvis tjenesten ved sending ikke støtter verken MTA-STS eller DANE, er innholdet fortsatt beskyttet av S/MIME. Da gjenstår beskyttelsen av metadataene og autentiseringen av mottaksserveren på den siste strekningen. Denne gjenværende risikoen hører hjemme i risikovurderingen.

## Gjenværende risikoer og tiltak

| Gjenværende risiko | Tiltak |
|---|---|
| Personopplysninger i emnet er lesbare | Regel: Ingen personopplysninger i emnet, skriftlig avtalt med partneren |
| manglende Forward Secrecy | lagre privat domenenøkkel på en ikke-eksporterbar måte, ideelt i en HSM; loggfør tilgang |
| Manipulering av usignerte meldinger | signaturplikt på begge sider, AES-GCM dersom begge gatewayene støtter det |
| Levering i klartekst ved feil | hold tilbake meldinger ved mislykket kryptering i stedet for å levere i klartekst |
| Metadata på overføringsveien | MTA-STS i modusen `enforce` og, der DNSSEC finnes, DANE; vurder TLS-RPT |
| utløpte sertifikater | gjensidig varsling minst 30 dager før utløp, sammenlign fingeravtrykk via en annen kanal |

Beslutningen om domene-kryptering og de gjenværende risikoene hører hjemme i datasikkerhetskonseptet eller i dokumentasjonen av behandlingsaktivitetene.

## Spørsmål til gateway-leverandøren

1.  Kontrollerer tjenesten ved sending MTA-STS-policyer i modusen `enforce` og DANE-oppføringer for mottakerdomenet?

2.  Fra hvilke servere leveres meldinger, og kan TLS-RPT-rapporter eller leveringslogger for transportkrypteringen gjennomgås?

3.  Støtter tjenesten som MX MTA-STS og DANE for innkommende meldinger, og hvordan varsles nøkkelbytter?

4.  Kan connectoren fra den egne e-postserveren til tjenesten drives med påtvunget TLS og sertifikatkontroll?

5.  Leveres det via offentlig SMTP mellom to kunder av tjenesten, eller internt på plattformen?

6.  Hvilke algoritmer brukes til kryptering og signatur, og støttes AES-GCM?

## Kilder

1.  [Føderal lov om personvern (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 om datasikkerhet, art. 9 om databehandling på oppdrag, art. 16 og 17 om utlevering til utlandet, art. 24 om meldeplikt, art. 61 og 62 om straffebestemmelsene.

2.  [Personvernforordning (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 til 3 med prinsipper, beskyttelsesmål og tekniske og organisatoriske tiltak, blant annet transportkontroll.

3.  [Sveitsisk straffelov (StGB)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): Art. 321 om brudd på profesjonshemmeligheten.

4.  [Føderal lov om den alminnelige delen av sosialforsikringsretten (ATSG)](https://www.fedlex.admin.ch/eli/cc/2002/510/de): Art. 33 om taushetsplikt for gjennomføringsorganene.

5.  [Sivilprosessloven (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 177 om dokumentbegrepet, inkludert elektroniske filer.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Spesifikasjon av policyen, hentereglene og modusen `enforce`.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672): DANE for SMTP, TLSA-oppføringer og kontrollregler for sendende servere.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Format og levering av TLS-RPT-rapportene.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): Gjeldende S/MIME-spesifikasjon med AuthEnvelopedData og AES-GCM.

10.  [EFAIL](https://efail.de/): Beskrivelse av angrepene på usignerte S/MIME- og OpenPGP-meldinger i CBC-modus, publisert i 2018.

11.  [Personvernforordningen (GDPR)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679): Art. 32 om sikkerhet ved behandling, art. 83 om bøtene (EU-merknad).
