---
title: "E-postkryptering: metoder, nøkler og driftsmodeller"
blatt: "verschluesselung"
description: "E-postkryptering for meldingsadministratorer: SMTP-TLS, MTA-STS, DANE og REQUIRETLS, S/MIME og OpenPGP, klient-, gateway- og portalmodeller, livssyklus for nøkler og sertifikater, metadata, overvåking, gjenoppretting og diagnostikk."
fakten:
  - label: Grunnspørsmål
    wert: Hvilke data er beskyttet mellom hvilke tillitsgrenser?
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Transport
    wert: SMTP STARTTLS beskytter nøyaktig ett hopp
    href: https://datatracker.ietf.org/doc/html/rfc3207
  - label: Transportpolicy
    wert: DANE · MTA-STS · REQUIRETLS
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf
  - label: S/MIME
    wert: MIME + CMS + X.509/PKIX
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: OpenPGP
    wert: Pakkeformat, nøkler, signaturer og kryptering
    href: https://datatracker.ietf.org/doc/html/rfc9580
  - label: Signatur
    wert: Autentisitet og integritet; ingen konfidensialitet
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Klientmodell
    wert: private nøkler ved endepunktet; e-postserveren lagrer chiffertekst
    href: https://www.nccoe.nist.gov/publication/1800-6/VolC/
  - label: Gatewaymodell
    wert: klartekst innenfor den kontrollerte sonen og ved gatewayen
    href: https://www.seppmail.com/products/
  - label: Portalmodell
    wert: varsling per e-post; innhold i nettlagring med egen identitet
    href: https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm
  - label: Nøkkeldrift
    wert: inventar · beskyttelse · distribusjon · rotasjon · tilbakekalling · gjenoppretting
    href: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
  - label: Synlige metadata
    wert: SMTP-konvolutten og ytre headere forblir synlige avhengig av modell
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Regel for gjenoppretting
    wert: sikkerhetskopier eventuelt krypteringsnøkler; ikke rekonstruer signaturnøkler
    href: https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - seppmail
translationSourceHash: 915d86b5c19b97fc5564faf33053144b692f76fd3d6c732fd955f466d25c4983
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T09:09:58.847Z
translationReview: required
---

# E-postkryptering: metoder, nøkler og driftsmodeller

Spørsmålet «Er denne e-posten kryptert?» er teknisk ufullstendig. En melding kan være lesbar på klienten, lagret som chiffertekst på e-postserveren, dekryptert på en gateway, beskyttet mellom to MTA-er med [TLS](/kb/tls) og lagret ukryptert igjen hos mottakeren. Et solid svar oppgir derfor **beskyttelsesobjekt, endepunkter, nøkkelinnehavere og klartekststeder**. NIST behandler transport-, domene- og innholdsmekanismer nettopp av denne grunn som samvirkende, men separate kontroller ([NIST SP 800-177 Rev. 1](https://www.nist.gov/publications/trustworthy-email-0)).

**Transportkryptering** beskytter en forbindelse. SMTP STARTTLS forhandles på nytt for hvert hopp og avsluttes ved den aktuelle MTA-en. **Innholdskryptering** beskytter en MIME-enhet med S/MIME eller OpenPGP og kan overleve flere transport- og lagringshopp. Om den faktisk fungerer ende-til-ende, avhenger ikke bare av formatet: Dersom en bedriftsgateway dekrypterer på vegne av brukeren, ligger de kryptografiske endepunktene på gatewayene og ikke hos brukerklientene ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

En **digital signatur** er enda en annen sikkerhetsfunksjon. Den knytter data til en privat signaturnøkkel og muliggjør integritets- og identitetskontroll; den gjør ikke innholdet hemmelig. Omvendt sier en vellykket dekryptert melding fortsatt ingenting om hvem som opprettet den. I S/MIME skilles det mellom CMS `SignedData` og `EnvelopedData` henholdsvis `AuthEnvelopedData`. OpenPGP skiller også mellom signatur-, øktnøkkel- og krypterte datapakker ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 8551, S/MIME Content Types](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580, Packet Syntax](https://datatracker.ietf.org/doc/html/rfc9580)).

Forklaringen følger en melding og spør ved hvert hopp hvor klartekst foreligger og hvem som kontrollerer nøklene. Transportkryptering, ende-til-ende-metoder, gatewayer, portaler, metadata, diagnostikk og gjenoppretting sammenlignes på dette grunnlaget.

E-postkryptering kan beskytte transporthoppet, meldingsinnholdet eller en portaltilgang. Disse modellene har ulike nøkkelinnehavere og ulike steder der klartekst foreligger. Artikkelen følger derfor først beskyttelsesveien og deretter de enkelte produktene og metodene.

## Arkitekturtilnærming: beskyttelsesmodeller og klartekstgrenser

E-postkryptering skiller seg først og fremst ved hvilke to punkter som krypterer og dekrypterer. Tabellen starter derfor med den normale meldingsveien og viser deretter hvor klartekst teknisk sett fortsatt er synlig.

| Modell | Kryptografiske endepunkter | Klartekst foreligger minst hos | Primær driftstilstand |
|---|---|---|---|
| opportunistisk SMTP-TLS | to tilstøtende MTA-er | hver deltakende MTA | sertifikater, TLS-funksjoner, kø og forsøk på nytt |
| påtvunget transport-TLS | to MTA-er pluss publisert eller lokal policy | hver deltakende MTA | MTA-STS/DANE/REQUIRETLS, DNS/HTTPS og rapporter |
| klient-til-klient S/MIME/OpenPGP | MUA eller kryptomodul hos brukerne | før kryptering og etter dekryptering ved endepunktet | private nøkler, sertifikater/nøkler, klientkompatibilitet |
| gateway-til-gateway | organisasjonenes e-postgatewayer | internt e-postsystem, gatewaybehandling, målsone | policy, katalog, gateway-nøkkellager, kø og tilordning |
| nettportal | avsender/gateway og portalens nettapplikasjon | gateway/portal, nettleser etter innlogging | meldingsobjekt, mottakerkonto, OTP, lenke, oppbevaring |

Ingen av disse modellene er generelt «sterkere». Klient-til-klient reduserer klartekststeder i infrastrukturen, men vanskeliggjør kontroll av skadelig programvare, journaling, eDiscovery og gjenoppretting. NCCoEs praktiske eksempel lar derfor S/MIME-kontrollen utføres på brukerklienten med dens Trust Anchors etter at transport- og domenebaserte kontroller er avsluttet ved MTA-en ([NCCoE: DNS-Based Email Security](https://www.nccoe.nist.gov/publication/1800-6/VolC/)). Gatewayer sentraliserer policy og nøkler, men blir dermed selv høyt privilegerte klartekst- og nøkkelsteder. Portaler når mottakere uten e-postkryptografi, men oppretter en ekstra metode for nett, identitet og lagring. Valget følger dataklassifisering, trusselmodell, oppbevaring, juridiske krav og kapasiteten hos begge kommunikasjonspartene – ikke produktnavnet.

<iframe class="kb-infographic kb-infographic--compact" style="aspect-ratio: 1400 / 1076; max-width: 720px; margin-inline: auto" src="/images/kb-interaktiv-verschluesselung.svg?v=20260813" title="Interaktive Infografik: E-Mail-Verschlüsselung mit Transport-, Client-, Gateway- und Portalmodell, Klartextgrenzen, Schlüsselzustand, Metadaten und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-verschluesselung.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## SMTP-TLS: beskyttelse per transporthopp

[SMTP](/kb/smtp) ble opprinnelig utformet uten obligatorisk konfidensialitet. STARTTLS utvider en eksisterende SMTP-økt: Etter vellykket TLS-handshake forhandles funksjoner på nytt med EHLO. I opportunistisk modus kan den sendende MTA-en falle tilbake til klartekst dersom STARTTLS mangler eller det oppstår en sertifikatfeil. Det forbedrer beskyttelsen mot passiv avlytting, men gir uten ekstra policy ingen robust garanti mot nedgradering eller en aktiv angriper ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

**MTA-STS** publiserer først en DNS-TXT-henvisning og henter den faktiske policyen via HTTPS. I `enforce`-modus må de angitte MX-navnene valideres med PKIX; feil fører til forsinket levering i stedet for klartekst. **DANE for SMTP** knytter sertifikat- eller nøkkelinformasjon til DNSSEC-validerte TLSA-poster. **REQUIRETLS** er en SMTP- og meldingsmerking som lar en avsender kreve TLS for videre overføring. Disse metodene har ulike avhengigheter og rekkevidder og må ikke samles i én avkrysningsboks «TLS required» ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

SMTP TLS Reporting, **TLS-RPT**, gir aggregerte rapporter om vellykkede og mislykkede TLS-policykontroller. Det beviser ingen enkeltlevering og inneholder ingen melding; det bidrar til å oppdage sertifikat-, DNS- og policyfeil før køer vokser lenge. NIST anbefaler at mottaksdomener signaliserer DANE, MTA-STS eller begge deler, og beskriver de ulike avhengighetene av DNSSEC, HTTPS og PKIX ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [NIST SP 800-177 Rev. 1, avsnitt 5.2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)).

SMTP-TLS avsluttes ved neste e-postserver. Dersom meldingsinnholdet skal forbli beskyttet på tvers av flere hopp, brukes meldingsrelaterte metoder som S/MIME eller OpenPGP.

## S/MIME: CMS, X.509 og MIME

S/MIME er ikke ett enkelt krypteringsbibliotek, men en kombinasjon av flere standarder. MIME strukturerer meldingsdeler; CMS definerer signerte og krypterte beholdere; X.509/PKIX knytter offentlige nøkler til identiteter og tillitsankere via sertifikater. For hver mottaker krypterer avsenderen den symmetriske innholdsnøkkelen med mottakerens offentlige nøkkel eller en avtalt nøkkelhåndteringsmetode. Den tilhørende private nøkkelen kreves for å lese ([RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

Administratoren må skille mellom to nøkkelroller. En **signaturnøkkel** skal bare være tilgjengelig for undertegneren; en sentral kopi ville teknisk gjøre det mulig for tredjeparter å opprette nye signaturer under vedkommendes identitet. En **dekrypteringsnøkkel** kan sikres eller legges i escrow i henhold til organisasjonens policy, slik at eldre forretningsmeldinger forblir lesbare etter tap av enhet, rolleendring eller sertifikatfornyelse. RFC 8551 påpeker uttrykkelig at en person kan ha flere aktive nøkkelpar til ulike formål; NIST skiller mellom beskyttelse og gjenoppretting for hver nøkkeltype ([RFC 8551, Multiple Key Pairs](https://datatracker.ietf.org/doc/html/rfc8551), [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Før sending trenger avsenderen et egnet og gyldig krypteringssertifikat for **hver** mottaker. Distribusjon kan skje via en katalog, signerte meldinger, en sertifikatplattform eller manuelt. Distribusjonsgrupper er særlig krevende fordi utvidelse, synlighet og sertifikattilgjengelighet for alle medlemmer må passe sammen. Microsoft dokumenterer for Exchange attributtene `userSMIMECertificate` og `userCertificate` samt sertifikatsamlingen for tillitskontroll ([Microsoft: S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime), [Microsoft: S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)).

## OpenPGP: format er ikke tillitsmodell

RFC 9580 definerer OpenPGP-pakker, nøkler, fingeravtrykk, signaturer, øktnøkler og krypterte data. En nøkkelring er ifølge standarden bare en samling av nøkler; hvordan en organisasjon finner nøkler, kontrollerer eiere og avgjør tillit, ligger utenfor dette dataformatet. Det historisk utbredte **Web of Trust** er derfor én mulig driftsform, men ikke en synonym definisjon av OpenPGP. Andre miljøer bruker manuelt kontrollerte fingeravtrykk, organisasjonssignerte nøkler, WKD, katalogtjenester eller sentralt administrerte gatewayer ([RFC 9580, Keyrings og Fingerprints](https://datatracker.ietf.org/doc/html/rfc9580)).

For e-post kapsler PGP/MIME inn signaturer og chiffertekst i MIME-strukturer. Inline-PGP beskytter derimot bare utvalgte tekstdeler og fører lett til interoperabilitetsproblemer med HTML, vedlegg, tegnsett og linjekanonisering. RFC 3156 standardiserer `multipart/encrypted` og `multipart/signed` for OpenPGP. Om en klient fullt ut støtter moderne RFC-9580-nøkler, AEAD, brukte kurver og PGP/MIME, må testes med de faktiske motpartene ([RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

En kort nøkkel-ID er ikke et tilstrekkelig identitetsbevis. RFC 9580 advarer om at nøkkel-ID-er ikke må antas å være entydige; fullstendig fingeravtrykk brukes ved kontroller. Tilbakekallingssertifikater, utløp, undernøkler, nye fingeravtrykk og sikker distribusjon av en rotasjon hører derfor til samme livssyklus som første nøkkelutstedelse.

S/MIME og OpenPGP kan brukes på endeenheten eller sentralt på gatewayen. Gatewayen forenkler policy og nøkkeldrift, men får til gjengjeld tilgang til klarteksten.

## Gatewaymodell: sentral policy og sentral makt over klartekst

En e-postgateway tar imot vanlige interne meldinger, klassifiserer avsender, mottaker og data, finner passende S/MIME- eller OpenPGP-nøkler og velger kryptering, signatur, TLS eller portallevering. Innkommende meldinger dekrypteres eventuelt før overlevering til det interne e-postsystemet. Produkter som [SEPPmail](/kb/seppmail), [Totemomail eller Kiteworks EPG](/kb/totemomail) og [HIN](/kb/hin) bruker ulike plattformer, men samme grunnleggende tillitsforskyvning: Organisasjonen delegerer kryptografi fra brukerens endepunkt til en infrastrukturkomponent ([SEPPmail: produkter](https://www.seppmail.com/products/), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Dette løser ikke nøkkeldistribusjonen av seg selv. Gatewayen trenger fortsatt en dokumentert kobling mellom mottakeridentitet og offentlig nøkkel, en prioritet mellom flere metoder, regler for utløpte eller tilbakekalte nøkler og en definert reserveprosedyre. Automatisk nøkkelutveksling mellom gatewayer kan forenkle disse trinnene, men er selv en tillitsprotokoll med motpartsidentitet, cache, utløp og feiltilstand.

Policyrekkefølgen avgjør hvilke kontroller som ser klartekst. Når en inngående gateway dekrypterer, kan anti-malware, DLP og arkivering arbeide etterpå. Dersom en utgående gateway krypterer for tidlig, er etterfølgende skannere blinde. Endrer en gateway en allerede signert MIME-enhet, kan innholds- eller [DKIM](/kb/mail-auth)-signaturen bli ugyldig. RFC 6376 gjør de kanoniserte headerne og meldingskroppen til signaturobjektet; derfor må signering, kryptering, ansvarsfraskrivelser, omskriving og DKIM ordnes i en dokumentert pipeline ([RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)).

Hvis ingen egnet nøkkel er tilgjengelig hos den eksterne mottakeren, kan en portal lagre meldingen og styre tilgang via nettinnlogging. Dermed går problemet fra e-postkryptografi til nettidentitet og plattformdrift.

## Portallevering: nettapplikasjon i stedet for mottakernøkkel

Ved portallevering mottar mottakeren en vanlig varslingse-post og åpner det beskyttede innholdet via en nettapplikasjon. Det kryptografiske endepunktet er portalen, ikke mottakerens e-postklient. Den tekniske oppgaven flyttes dermed fra distribusjon av offentlige nøkler til nettidentitet, øktadministrasjon, lagring av meldinger og vedlegg, varslingslenke, oppbevaring og sikker svarkanal.

HIN dokumenterer som et konkret eksempel registrering, mobilnummer og SMS-kode samt muligheten til å huske en enhet for senere tilgang. Dette eksemplet dokumenterer driftsmodellen, ikke en universelt gyldig portalimplementering. Andre produkter bruker passord, engangskode via separat kanal, føderert identitet eller en eksisterende kundeinnlogging ([HIN: Mail til ikke-medlemmer](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

For administratorer trenger en portal minst følgende bevis:

- Hvilke data finnes i det ubeskyttede varselet og i URL-tokenet?
- Hvordan sikres førstegangsregistrering, identitetsbinding, mislykkede forsøk, å huske enhet og kontogjenoppretting?
- Hvor lagres meldinger, vedlegg, nøkler og revisjonsdata, og når slettes de?
- Kan en melding gjenopprettes etter utløp, sletting av bruker eller bytte av tenant?
- Hvordan loggføres sikkert svar, videresending, nedlasting og lesebekreftelse?
- Hva skjer når portalen ikke er tilgjengelig, SMS-tjenesten er nede eller mottakerkontoen er kompromittert?

En HTTP-status `200` beviser bare at et webgrensesnitt svarer. En syntetisk portalgjennomgang må dekke varsling, registrering eller innlogging, annen faktor, åpning, vedlegg, svar og den forventede revisjonsposten.

## Metadata, headere og arkivering

Innholdskryptering beskytter ikke automatisk SMTP-konvolutten. Leverende MTA-er trenger minst mottakeradresser; systemer oppretter `Received`-linjer og behandler størrelses- og ruteinformasjon. Også med S/MIME forblir ytre headere normalt synlige. RFC 8551 tillater å kapsle inn en fullstendig melding som `message/rfc822` i den beskyttede MIME-enheten, men krever en skillelig fremstilling av indre og ytre headere. Om emne, avsendervisning og mottakerliste faktisk er beskyttet, er en klient- og gatewayfunksjon, ikke en sikker konsekvens av låssymbolet ([RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322), [RFC 8551, Header Protection](https://datatracker.ietf.org/doc/html/rfc8551)).

Arkivering må fastsette **hvilken representasjon** som bevares: opprinnelig klartekst, signert originalenhet, kryptert original, dekryptert arbeidskopi eller portalobjekt. Bare klartekst forenkler eDiscovery, men mister det kryptografiske originalbeviset. Bare chiffertekst kan bli ulesbar etter nøkkeltap. Signaturvalidering på et senere tidspunkt krever opprinnelige byte, sertifikatkjede, daværende tillits- og sperreinformasjon samt et tidsgrunnlag; et sertifikat som er utløpt i dag, gjør ikke automatisk en signatur som da var gyldig verdiløs.

## Livssyklus for nøkler og sertifikater

En produktiv kryptotjeneste trenger et inventar som går ut over utløpsdatoer. NIST SP 800-57 behandler generering, aktivering, beskyttelse, inventar, distribusjon, bruk, rotasjon, sperring, arkiv, gjenoppretting og destruksjon som sammenhengende administrasjonsoppgaver ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

| Fase | Administratorspørsmål | Bevis |
|---|---|---|
| Identitet | Hvem eller hvilken rolle knyttes til nøkkelen? | kontrollert registreringsprosess, Subject/SAN/User ID, godkjenning |
| Generering | Hvor opprettes den private nøkkelen? | klient, HSM, gateway eller CA; algoritme og eksporterbarhet |
| Distribusjon | Hvordan får avsenderen riktig offentlig nøkkel? | katalog, signert e-post, WKD, sertifikattjeneste, kontroll av fingeravtrykk |
| Bruk | Kan nøkkelen signere, dekryptere eller begge deler? | Key Usage/EKU, OpenPGP-flagg, policy og faktisk klient |
| Rotasjon | Hvordan overlapper gammel og ny nøkkel? | publisering før overgang, test mot alle målplattformer |
| Tilbakekalling | Hvordan får kontrollører og avsendere vite om kompromittering? | CRL/OCSP eller OpenPGP-tilbakekalling, cache- og feilpolicy |
| Gjenoppretting | Hvem kan dekryptere historisk innhold? | escrow-prosess, godkjenning etter fire øyne, revisjon, gjenopprettingstest |
| Destruksjon | Når opphører dekrypterbarheten med hensikt? | bevis for oppbevaring og sletting av nøkler og sikkerhetskopier |

S/MIME-banekontroll omfatter tillitsanker, mellomsertifikater, gyldighetstid, navn eller e-postidentitet, Key Usage/Extended Key Usage, algoritme og sperrestatus. OCSP og CRL-er er igjen avhengige av nettverk og cache. Ved en soft fail kan en kontroll fortsette selv om sperreinformasjon ikke er tilgjengelig; ved hard fail kan et infrastrukturproblem blokkere alle signerte eller krypterte meldinger. Denne beslutningen hører hjemme i policyen og overvåkingen ([RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

Diagnostikken starter med den valgte beskyttelsesmodellen: TLS-hopp, signatur og kryptering av innholdet eller portaltilgang. Først deretter kontrolleres sertifikat, nøkkel, policy og leveringsvei.

## Overvåking og feilanalyse

Overvåking må holdes atskilt etter beskyttelsesmodell. For transport-TLS teller tilbudte og forhandlede protokoller, policyfeil, sertifikatvalidering, TLS-RPT og køalder. For S/MIME/OpenPGP teller manglende mottakernøkler, ugyldige baner eller signaturer, reserveprosedyrer, rotasjon og dekrypteringsfeil. For gatewayer kommer policyavgjørelse, Message-ID, nøkkelfingeravtrykk, klartekstbehandling og neste hopp i tillegg. For portaler er registrering, MFA/OTP, lagring, lenkeutløp, innlogging, åpning av melding og svarvei egne signaler.

En fornuftig hendelsesprotokoll besvarer:

1. Hvilken melding og hvilken mottakervei er berørt?
2. Hvilken beskyttelsesmodell og hvilken policy ble valgt?
3. Hvilken offentlig nøkkel eller hvilket sertifikat ble brukt, med hvilket fingeravtrykk og hvilken status?
4. Hvor var meldingen sist lesbar, kryptert, lagret eller satt i kø?
5. Var resultatet levering, portalarkivering, karantene, reserveprosedyre, nytt forsøk eller permanent feil?
6. Kan resultatet gjenskapes med en kontrollert testidentitet uten å eksponere produksjonsdata?

## Sikkerhetskopi og gjenoppretting

Krypterte data og nøkler må planlegges sammen, men med separate beskyttelsestiltak. En sikkerhetskopi av krypterte meldinger uten de private dekrypteringsnøklene som trengs på lang sikt, kan være korrekt sikret og likevel ikke faglig gjenopprettbar. Omvendt gjør en utilstrekkelig beskyttet sikkerhetskopi av nøkkeleskrow mange enkelthemmeligheter til ett sentralt angrepspunkt. NIST behandler tillatt bruk av gjenopprettede private nøkler avhengig av nøkkeltype og kryptoperiode ([NIST SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)).

Minste omfang for gjenoppretting omfatter:

- krypterte originalmeldinger og eventuelt bevissikre signerte MIME-enheter;
- private **dekrypterings**nøkler med metadata, passfraser, roller og godkjenningsprosedyrer;
- offentlige nøkler, sertifikatkjeder, Trust Anchors, sperre- og historiske valideringsdata;
- gateway- og portalpolicy, ruting, nøkkeltilordning, brukerdata og meldingsobjekter;
- revisjons- og tidskilder, slik at tidligere avgjørelser forblir sporbare;
- en gjenopprettingstest med gamle og nye nøkler, flere mottakere, vedlegg og signaturkontroll.

Private signaturnøkler bør ikke kopieres som en generell gjenopprettingsmekanisme. Dersom en organisasjon må signere automatisk eller på vegne av noen, er en eksplisitt organisasjonsidentitet med egen policy bedre enn en umerkelig kopi av en personlig nøkkel. Gjenopprettbarhet og ikke-benektelse står ellers i konflikt med hverandre.

## Diagnoseverktøy

Feilsøkingen følger den valgte beskyttelsesmodellen. Ved transportkryptering kontrolleres DNS og TLS; ved ende-til-ende-metoder følger sertifikater, nøkler og klientbehandling; ved gatewayer kommer deres policy- og køtilstand i tillegg.

### MTA-STS, TLS-RPT og DANE i DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für E-Mail-Transportpolicy-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type TXT _mta-sts.example.ch
Resolve-DnsName -Type TXT _smtp._tls.example.ch
Resolve-DnsName -Type TLSA _25._tcp.mx.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short TXT _mta-sts.example.ch
dig +short TXT _smtp._tls.example.ch
dig +dnssec TLSA _25._tcp.mx.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser publiserte transportpolicyer. For DANE må DNSSEC-valideringsstatusen i tillegg dokumenteres; en bare synlig TLSA-tekst er ennå ikke en autentisert post.

### STARTTLS og sertifikatkjede

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer TCP-veien. [`curl`](https://curl.se/docs/manpage.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser STARTTLS, handshake og kjede. Om en MTA i produksjon faller tilbake eller setter meldingen i kø ved feil, fremgår først av policyen og loggene.

### S/MIME-sertifikat og bruksformål

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für S-MIME-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
certutil.exe -dump .\recipient-smime.cer
certutil.exe -verify -urlfetch .\recipient-smime.cer
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl x509 -in recipient-smime.pem -noout \
  -subject -issuer -serial -fingerprint -dates -purpose -ext keyUsage -ext extendedKeyUsage
openssl verify -show_chain -CAfile trust-chain.pem recipient-smime.pem
```

  </div>
</div>

[`certutil`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil) og [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) viser identitet, fingeravtrykk, gyldighet og bruksformål; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) kontrollerer en angitt Truststore. Testen må gjentas med de samme Trust Anchors og sperrereglene som produksjonsklienten eller gatewayen.

### OpenPGP-fingeravtrykk og nøkkelstruktur

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OpenPGP-Schlüsselprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
gpg.exe --show-keys --with-fingerprint --with-subkey-fingerprint .\recipient.asc
gpg.exe --list-packets .\message.pgp
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
gpg --show-keys --with-fingerprint --with-subkey-fingerprint recipient.asc
gpg --list-packets message.pgp
```

  </div>
</div>

[`gpg`](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html) viser fullstendige fingeravtrykk, undernøkler, funksjoner og pakkestruktur. `--list-packets` analyserer bare formatet; det bekrefter verken eier eller tillit. Fingeravtrykk må kontrolleres gjennom en autentisert sekundærkanal eller en definert organisasjonsprosedyre.

### Portalvei og pakkeopptak

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Portal- und Paketpfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
pktmon filter remove
pktmon filter add SecureMail -p 25
pktmon start --capture --pkt-size 0 --file-name securemail.etl
pktmon stop
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
tcpdump -ni any -s 0 -w securemail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) og [`curl`](https://curl.se/docs/manpage.html) kontrollerer bare HTTP/TLS-tilgjengelighet. [`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) og [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) fanger bare opp ved det valgte målepunktet. Pakkedata kan inneholde SMTP-klartekst, adresser og portalmetadata og må beskyttes tilsvarende.

## Teknisk historie

PEM, Privacy Enhanced Mail, standardiserte kryptografisk beskyttet Internett-e-post tidlig på 1990-tallet, men oppnådde ikke bred interoperabilitet. S/MIME koblet senere CMS-strukturen med opprinnelse i PKCS til MIME og X.509. S/MIME-standardene utviklet seg gjennom flere generasjoner frem til versjon 4.0, publisert i 2019; CMS er selv standardisert i RFC 5652 ([RFC 1421](https://datatracker.ietf.org/doc/html/rfc1421), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652)).

OpenPGP oppstod fra PGP-pakkeformatet og ble først standardisert i RFC 2440, senere i RFC 4880. RFC 9580 erstattet RFC 4880 i 2024 og moderniserte blant annet nøkkelversjoner, fingeravtrykk og metoder for autentisert kryptering. PGP/MIME etter RFC 3156 forblir e-postkapslingen; formatutviklingen fjerner imidlertid ikke den organisatoriske oppgaven med å regulere nøkkelidentitet og gjenoppretting ([RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580), [RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156)).

SMTP STARTTLS ble standardisert i RFC 2487 i 1999 og oppdatert av RFC 3207 i 2002. DANE for SMTP, MTA-STS, TLS-RPT og REQUIRETLS la senere til policy- og rapporteringslag mot nedgradering og feilkonfigurasjon. Parallelt etablerte virksomheter gatewayer og nettportaler fordi eksterne mottakere ikke pålitelig har kompatible S/MIME- eller OpenPGP-nøkler. Dermed ble e-postkryptering mindre en enkelt klientfunksjon og mer et distribuert system av SMTP, DNS, HTTPS, PKI, katalog, policy, nettidentitet og gjenoppretting ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

## Kilder

- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [NCCoE – DNS-Based Email Security Practice Guide](https://www.nccoe.nist.gov/publication/1800-6/VolC/)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8689 – SMTP REQUIRETLS](https://datatracker.ietf.org/doc/html/rfc8689)
- [IETF RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460)
- [NIST – SP 800-177 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime)
- [Microsoft Learn – S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)
- [IETF RFC 3156 – MIME Security with OpenPGP](https://datatracker.ietf.org/doc/html/rfc3156)
- [SEPPmail – Produkter](https://www.seppmail.com/products/)
- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [IETF RFC 6376 – DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [HIN Support – Mail til ikke-medlemmer](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [IETF RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – certutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/)
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/)
- [GnuPG – gpg manual](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [IETF RFC 1421 – Privacy Enhancement for Internet Electronic Mail](https://datatracker.ietf.org/doc/html/rfc1421)
