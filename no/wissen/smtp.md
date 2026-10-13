---
title: "SMTP: Transport, kø og leveringsansvar"
blatt: "smtp"
description: "Simple Mail Transfer Protocol sett fra en administrators perspektiv: rollene til MUA, MSA, MTA og MDA, konvolutt og meldingsinnhold, SMTP-tilstandsautomat, akseptgrense, MX-ruting, kø og gjenforsøk, svarkoder, ESMTP-utvidelser, innsending, TLS-policy og lagvis diagnostikk."
fakten:
  - label: Fullt navn
    wert: Simple Mail Transfer Protocol
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Grunnmodell
    wert: Command/Reply og Store-and-Forward
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Transportobjekt
    wert: Konvolutt pluss meldingsinnhold
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Transport
    wert: pålitelig, ordnet datastrøm; på Internett TCP
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-1.1
  - label: Server-til-server
    wert: SMTP over TCP 25
    href: https://datatracker.ietf.org/doc/html/rfc6409#section-1
  - label: Innsending
    wert: 587 med STARTTLS · 465 med implisitt TLS
    href: https://datatracker.ietf.org/doc/html/rfc8314#section-3
  - label: Kjerneroller
    wert: MUA · MSA · MTA · MDA eller LMTP
    href: https://datatracker.ietf.org/doc/html/rfc5598#section-4
  - label: Funksjoner
    wert: EHLO forhandler ESMTP-utvidelser per økt
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.2
  - label: Akseptgrense
    wert: 250 etter End-of-Data overfører ansvar
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4
  - label: Feilklasser
    wert: 4xx midlertidig · 5xx permanent · x.y.z som detalj
    href: https://datatracker.ietf.org/doc/html/rfc3463#section-2
  - label: Avhengigheter
    wert: DNS · TCP · TLS · SASL · MIME · persistent kø
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3
  - label: Standardhistorikk
    wert: RFC 821 → RFC 2821 → RFC 5321
    href: https://datatracker.ietf.org/doc/html/rfc5321#appendix-C
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: bd51241025647a47c1c52f82747a0f7eee7d7b8e42d57f52057e7123d6bc3825
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:19:29.186Z
translationReview: required
---

# SMTP: Transport, kø og leveringsansvar

**Simple Mail Transfer Protocol** (SMTP) bringer en melding fra én e-postprosess til den neste. Klienten kobler seg til en server, spør etter dens funksjoner, oppgir konvoluttavsender og mottaker og overfører først deretter meldingsinnholdet. Serveren svarer på hvert trinn med en numerisk kode. Fra denne tilsynelatende enkle sekvensen oppstår ruting, køsetting, gjenforsøk og det avgjørende spørsmålet om hvilken server som akkurat nå har ansvaret for meldingen ([RFC 5321, avsnitt 1 og 2](https://datatracker.ietf.org/doc/html/rfc5321#section-1)).

For meldingsadministratorer ligger den avgjørende egenskapen i **overføringen av ansvar**. En vellykket TCP-etablering betyr bare at en lytter er tilgjengelig. En akseptert `RCPT TO` reserverer en mottaker i den pågående transaksjonen. Først en positiv avslutning etter det fullstendige meldingsinnholdet overfører til det mottakende SMTP-systemet ansvaret for å levere meldingen eller rapportere en senere feil til konvoluttavsenderen ([RFC 5321, avsnitt 4.1.1.4 og 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)). Kø-ID, svarkode, ekstern MTA, mottakerstatus og `Received`-spor er derfor ikke tilfeldige loggdetaljer, men beviskjeden for en levering.

Forklaringen følger en melding fra den innsendende klienten via SMTP-økt, konvolutt og kø til neste e-postserver eller postboks. DNS, gjenforsøk, statuskoder, sikkerhet og diagnostikk utdypes der de påvirker veien.

## Roller og ansvarsområder

SMTP kjenner i utgangspunktet bare **klient** og **server**. I en faktisk leveringskjede byttes disse rollene ved hvert hopp: En Mail Transfer Agent tar som server imot en melding, lagrer den i køen sin og blir selv klient ved neste leveringsforsøk. Først gjennom dette rollebyttet kan logger og ansvar leses riktig ([RFC 5321, avsnitt 2.3.3](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.3), [RFC 5598, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc5598#section-4), [RFC 6409, avsnitt 2.1](https://datatracker.ietf.org/doc/html/rfc6409#section-2.1)).

| Rolle | Oppgave | Typisk tillits- og tilstandsgrense |
|---|---|---|
| **MUA** – Message User Agent | oppretter en melding og viser mottatte meldinger | brukeridentitet, lokale utkast, adressebok og visning; ingen offentlig relaytjeneste |
| **MSA** – Message Submission Agent | tar imot nye meldinger fra autoriserte brukere eller applikasjoner | autentisert identitet, avsenderautorisasjon, innsending-policy og umiddelbar feilretur |
| **MTA** – Message Transfer Agent | ruter og overfører aksepterte meldinger til andre MTA-er | persistent kø, gjenforsøksplanlegger, DNS-målvalg, TLS-policy og leveringsansvar |
| **Gateway eller relay** | konverterer mellom sikkerhetssoner, organisasjoner eller meldingssystemer | policy, omskriving, innholdskontroll, protokollkonvertering og egen kø |
| **MDA** – Message Delivery Agent | overleverer til postboks, fil, pipe eller proprietær lagring | endelig mottakeroppløsning, kvote og postbokskonsistens |
| **LMTP-server** | tar imot en melding fra MTA-en for lokal levering | rapporterer separat status for hver mottaker etter innholdet og trenger selv ingen langsiktig relaykø |

MSA-en er ikke bare en MTA på en annen port. Message Submission skiller kontrollert innsending av nye meldinger fra åpen overføring mellom e-postdomener. En MSA kan blant annet kontrollere avsenderrettigheter, legge til manglende obligatoriske felt og tilpasse meldinger til lokale regler; en etterfølgende MTA bør som hovedregel ikke vilkårlig endre ferdig meldingsinnhold ([RFC 6409, avsnitt 1, 6 og 8](https://datatracker.ietf.org/doc/html/rfc6409#section-1)). Denne grensen er særlig relevant for DKIM-signaturer: Enhver senere endring av signerte headere eller brødteksten kan gjøre signaturen ugyldig.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813" title="Interaktive Infografik: SMTP-Rollen, Sitzungsablauf, Envelope, Queue und Zustellverantwortung" loading="lazy">
  <a href="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813">Åpne infografikk om SMTP-leveringssti og akseptgrense</a>
</iframe>

Grafikken viser logiske grenser. MSA, utgående MTA, gateway og inngående MTA kan kjøre som separate tjenester, befinne seg på samme system eller være skjult av en skyleverandør. Standardene angir ingen prosessarkitektur. Administrativt er skillet likevel nødvendig, fordi hver aksepterte overlevering kan opprette en ny kø, en ny gjenforsøkstilstand og en ny ansvarlig.

## Konvolutt og meldingsinnhold

Før dialogen gjennomgås i detalj, må to dataområder skilles: **konvolutten** for transport og **meldingsinnholdet** for mottakere og e-postprogrammer. Konvolutten består av en Reverse Path fra `MAIL FROM` og én eller flere Forward Paths fra `RCPT TO`. Innholdet følger etter `DATA` og inneholder et headerområde i henhold til Internet Message Format samt brødteksten, ofte med MIME-struktur ([RFC 5321, avsnitt 2.3.1](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1), [RFC 5322, avsnitt 2.1](https://datatracker.ietf.org/doc/html/rfc5322#section-2.1)).

| Element | Protokollobjekt | Hva det faktisk brukes til |
|---|---|---|
| `MAIL FROM:<bounce@example.net>` | Envelope Reverse Path | mål for ikke-leveringsrapporter og inndata for SPF; kan være tom for DSN-er |
| `RCPT TO:<user@example.ch>` | Envelope Forward Path | faktisk ruting og mottakeravgjørelse; kan gjentas per mottaker |
| `From: Author <author@example.net>` | meldingsheader | synlig forfatteridentitet og DMARC-referansedomene; ingen SMTP-rutingskommando |
| `To:` og `Cc:` | meldingsheader | synlige målangivelser for mennesker og MUA-er; behøver ikke tilsvare alle konvoluttmottakere |
| `Bcc:` | opprettelsesinformasjon | Bcc-mottakere leveres via konvoluttadresser uten at adressen deres oppgis til de andre kopiene |
| `Message-ID:` | meldingsheader | innholdsmessig meldingsidentifikator; ikke det samme som kø-ID eller DSN-konvolutt-ID |
| `Return-Path:` | sporingsheader opprettet ved endelig levering | bevarer sist brukte Reverse Path; skal ikke overtas som en fritt satt avsenderlinje fra en inngående melding |

SMTP-servere må ikke avlede den endelige ruten fra `To:`, `Cc:` eller `From:`. Omvendt trenger ikke konvoluttadresser å forekomme i meldingsheaderen. Derfor fungerer Bcc, e-postlister, videresendinger, SRS og separate bounce-adresser. Skillet forklarer også hvorfor en melding i klienten visuelt kan synes å komme fra `ceo@example.ch`, mens konvoluttavsenderen bruker et annet domene. SPF vurderer SMTP-stien; DMARC oppretter i tillegg en referanse til synlig `From:`. Detaljene i disse kontrollene hører hjemme i [e-postautentisering](/kb/mail-auth) ([RFC 5321, vedlegg C](https://datatracker.ietf.org/doc/html/rfc5321#appendix-C), [RFC 5322, avsnitt 3.6](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6)).

En tom Reverse Path overføres som `MAIL FROM:<>`. Den forhindrer feilrapportsløyfer og er påkrevd for Delivery Status Notifications. En server som generelt avviser tomme avsendere, skader derfor tilbakerapporteringsveien i e-postsystemet ([RFC 5321, avsnitt 3.6.3](https://datatracker.ietf.org/doc/html/rfc5321#section-3.6.3)).

## SMTP-økt og transaksjon

På en [TCP-forbindelse](/kb/tcp) kjører en **SMTP-økt**; innenfor den kan flere meldinger følge som separate **e-posttransaksjoner**. Serveren starter med en `220`-hilsen; klienten identifiserer seg med `EHLO` og mottar en øktspesifikk liste over ESMTP-funksjoner. Deretter består en transaksjon av `MAIL`, minst én akseptert `RCPT` og `DATA` eller en forhandlet erstatning som `BDAT` ([RFC 5321, avsnitt 3.1 og 4.1](https://datatracker.ietf.org/doc/html/rfc5321#section-3.1)).

1. **Forbindelse og hilsen:** Klienten kobler seg til; serveren svarer med `220`. Hvis den forsinker eller avviser allerede her, finnes det ennå ingen e-posttransaksjon.
2. **Funksjonsforhandling:** `EHLO client.example` setter begge parter i initialtilstand og oppgir utvidelser som `SIZE`, `STARTTLS`, `PIPELINING` eller `SMTPUTF8`. En klient kan ikke forutsette at en annen klyngenode eller en senere økt tilbyr de samme funksjonene.
3. **Valgfri TLS-oppgradering:** Etter vellykket `STARTTLS` forkastes SMTP-tilstanden og tidligere lærte funksjoner. Klienten sender `EHLO` på nytt innenfor den beskyttede kanalen ([RFC 3207, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc3207#section-4)).
4. **Konvoluttavsender:** `MAIL FROM` åpner en transaksjon og tømmer tidligere konvolutt- og databufre. ESMTP-parametere som `SIZE`, `BODY` eller `SMTPUTF8` er knyttet til denne kommandoen.
5. **Mottaker:** Hver `RCPT TO` besvares enkeltvis. En server kan akseptere to mottakere og permanent avvise en tredje; klienten sender da innholdet for den aksepterte delmengden.
6. **Innhold:** Etter `DATA` følger ved beredskap `354`. Klienten overfører headere og brødtekst som CRLF-avsluttede linjer, masker punkter i starten av linjen og avslutter med en linje som kun består av `.`.
7. **Avslutning:** Svaret etter denne End-of-Data-markøren avgjør om serveren overtar meldingen som helhet for de tidligere aksepterte mottakerne. `RSET` forkaster den pågående transaksjonen uten å avslutte økten; `QUIT` avslutter den på en ordnet måte.

SMTP er lockstep-orientert, men ESMTP-`PIPELINING` kan tillate at tillatte kommandogrupper sendes uten å vente på hvert enkelt svar. Svarene forblir i mottaksrekkefølge og må fortsatt tilordnes riktig til hver kommando ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)). Et pakkefangst med flere `MAIL`-/`RCPT`-linjer rett etter hverandre er derfor ikke automatisk et protokollbrudd.

Dialogen er dermed fullført, men leveringsansvaret er ennå ikke avklart. Det skifter først med et helt bestemt svar fra den mottakende serveren.

## Akseptgrense: 250 etter End-of-Data

Den viktigste SMTP-grensen ligger etter meldingsinnholdet. En `250` på `MAIL FROM` aksepterer bare Reverse Path. En `250` på `RCPT TO` aksepterer en mottaker for den pågående transaksjonen. Først en `250` etter End-of-Data-markøren betyr at serveren har overtatt ansvaret for meldingen. Et korrekt system må deretter enten levere den eller opprette en ikke-leveringsrapport ved en senere oppdaget feil; den sendende klienten kan fjerne sin lokale kopi fra det aktive leveringsansvaret ([RFC 5321, avsnitt 4.1.1.4 og 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Denne semantikken gjør siste del av dialogen kritisk. Hvis forbindelsen brytes etter punktet, men før et fullstendig svar er mottatt, vet ikke klienten sikkert om serveren har lagret meldingen. Den må kunne forsøke leveringen på nytt; dermed er duplikater mulig. SMTP gir ingen global Exactly-once-garanti. Systemer trenger derfor idempotent sporing via kø-ID, `Message-ID`, mottaker, tid, størrelse og eventuelt en proprietær transaksjonsreferanse, uten å likestille disse identifikatorene.

Innholdsfiltre, DLP, skadevarekontroll eller DKIM-signering bør integreres i akseptpipen slik at serveren før det endelige positive svaret vet om den kan overta persistent ansvar. En foranliggende proxy som for tidlig rapporterer `250` og deretter mister meldingen internt, kan ikke lenger returnere feilen til den opprinnelige SMTP-klienten. Nettopp her må en arkitektur fastsette om aksept først bekreftes etter durable write, etter replikering, etter skanning eller allerede ved overlevering til en intern broker. Dette er en produktegenskap, ikke et krav i SMTP-standarden, og må dokumenteres gjennom produsentdokumentasjon samt feil- og gjenopprettingstester.

Etter aksept må MTA-en bestemme hvor meldingen skal sendes videre. For Internett-e-post begynner denne avgjørelsen i DNS.

## Ruting via DNS og MX

For direkte Internett-levering bestemmer den sendende MTA-en neste mål ut fra domenet til konvoluttmottakeren. Den spør [DNS](/kb/dns) etter MX-poster, sorterer kandidater etter stigende preferanseverdi og løser de valgte Exchange-navnene til A- eller AAAA-adresser. Flere mål med samme preferanse er likeverdige kandidater; en backup-MX med høyere verdi er ikke en kopi av primærkøen, men en selvstendig SMTP-mottaker med egen policy og leveringsansvar ([RFC 5321, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5)).

Hvis et MX-RRset mangler, behandler SMTP historisk domenet som en implisitt MX på selve domenenavnet og forsøker adressene dets. En eksplisitt **Null MX** med preferanse `0` og Exchange `.` erklærer derimot at et domene ikke mottar e-post ([RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)). `NXDOMAIN`, `NODATA`, `SERVFAIL`, DNSSEC-valideringsfeil og en tilgjengelig MX uten fungerende SMTP-tjeneste er ulike tilstander og må forbli separat synlige i køårsaken.

Interne e-postveier bruker ofte statiske neste hopp, smarthoster, connectorer eller rutingtabeller i stedet for offentlig MX-valg. Konvoluttmottakeren forblir likevel leveringsadressen; den lokale rutingpolicyen avgjør bare hvilket neste system den betros til. Ved hver diagnose må derfor tre navn noteres: mottakerdomenet, valgt neste-hop-navn og faktisk tilkoblet IP-adresse.

Selv når MTA-en kjenner målet, er levering ikke alltid mulig umiddelbart. Midlertidige feil fører derfor til køen i stedet for tap av meldingen.

## Kø, gjenforsøk og DSN

En fullverdig MTA er et Store-and-Forward-system. Hvis den ikke straks kan nå neste hopp eller dette svarer midlertidig, beholder MTA-en melding og konvolutt i en persistent kø, planlegger et nytt forsøk og administrerer status per mottaker. RFC 5321 krever en gjenforsøksstrategi, men foreskriver ingen enkelt algoritme for alle produkter. Den anbefaler forsinkede gjentakelser, varsling ved langvarig forsinkelse og oppbevaring i flere dager før endelig oppgivelse ([RFC 5321, avsnitt 4.5.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.5.4)). Produktverdier som backoff, maksimal kølevetid, parallellitet og Destination Concurrency må derfor leses fra den konkrete MTA-konfigurasjonen.

En fornuftig køoppføring inneholder minst:

- uforanderlig meldingsinnhold eller en referert blob;
- konvoluttavsender og status for hver konvoluttmottaker;
- tidspunkt for aksept, neste forsøk og utløpsgrense;
- valgt transport, neste hopp, siste adresse og TLS-/policyavgjørelse;
- siste SMTP- og Enhanced-statuskode sammen med ekstern tekst;
- lokal kø-ID samt korrelerbare ID-er fra foranliggende eller etterfølgende hopp;
- tilstander for aktiv, deferred, hold, delivered, bounced og quarantined;
- replikerings-, filsystem- eller databasetilstand når aksept skal være høytilgjengelig.

Midlertidige feil forsøkes normalt på nytt. Permanente feil før egen aksept returneres synkront til den sendende SMTP-klienten. Permanente feil etter egen aksept fører til en **Delivery Status Notification** (DSN) til Reverse Path. RFC 3464 definerer for dette en maskinlesbar `multipart/report`-melding med felt per melding og per mottaker, som `Reporting-MTA`, `Final-Recipient`, `Action`, `Status` og valgfritt `Remote-MTA` eller `Diagnostic-Code` ([RFC 3464](https://datatracker.ietf.org/doc/html/rfc3464)).

En bounce er ikke automatisk pålitelig. DSN-er kan forfalskes, avsløre det opprinnelige innholdet eller generere backscatter ved misbrukt konvoluttavsender. En gateway bør avvise ugyldige meldinger så langt som mulig under den opprinnelige SMTP-økten, i stedet for å akseptere dem og senere returnere dem til en mulig forfalsket adresse.

## Svarkoder og diagnostisk betydning

SMTP-svar har en tresifret basiskode. Det første sifferområdet beskriver den umiddelbare tilstanden; teksten er beregnet på mennesker og kan være produktspesifikk ([RFC 5321, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc5321#section-4.2)).

| Klasse | Betydning i dialogen | Konsekvens for den sendende prosessen |
|---|---|---|
| `2xx` | positiv fullføring | utfør neste protokolltrinn; etter End-of-Data bokfør ansvaret som overført |
| `3xx` | positiv mellomstatus | send videre data, for eksempel meldingsinnhold etter `354` |
| `4xx` | midlertidig negativ fullføring | behold meldingen eller den berørte mottakeren i køen og forsøk igjen i henhold til gjenforsøkspolicy |
| `5xx` | permanent negativ fullføring | ikke gjenta uendret; rapporter synkront til forrige klient eller opprett DSN etter tidligere aksept |

**Enhanced Status Codes** supplerer formatet `x.y.z`. `x` angir suksess, midlertidig eller permanent; `y` angir det grove feilområdet og `z` detaljen. Dermed tilhører `5.1.1` adresseklassen og angir en ikke-eksisterende målpostboks, mens `4.4.x` klassifiserer nettverks- og rutingproblemer. De registrerte betydningene finnes i [IANA-registeret for Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml); en produsenttekst må ikke vurderes i strid med den numeriske semantikken ([RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463), [RFC 5248](https://datatracker.ietf.org/doc/html/rfc5248)).

Den samme koden kan gjelde noe forskjellig avhengig av protokollfase. `550` på `MAIL FROM` avviser avsenderen eller dens autorisasjon, på `RCPT TO` en bestemt mottaker og etter innholdet hele transaksjonen. En sak uten ekstern vert, tid, fase, basiskoder, Enhanced Code og fullstendig svarttekst er derfor ufullstendig.

## ESMTP og utvidelser

Extended SMTP bruker `EHLO` for å forhandle funksjoner per forbindelse. IANA fører det bindende [registeret over SMTP Service Extensions](https://www.iana.org/assignments/smtp/smtp.xhtml). En klient kan bare bruke en utvidelse når den er tilbudt i den aktuelle økten; etter STARTTLS må den hente listen på nytt.

| EHLO-nøkkelord | Teknisk virkning | Administrativ relevans |
|---|---|---|
| `SIZE` | serveren kan angi en fast størrelsesgrense; klienten kan erklære forventet størrelse ved `MAIL FROM` | unngår unødvendig dataoverføring, men garanterer ennå ikke aksept ([RFC 1870](https://datatracker.ietf.org/doc/html/rfc1870)) |
| `PIPELINING` | tillater definerte kommandogrupper uten round trip per kommando | øker gjennomstrømmingen over forbindelser med høy latenstid; svar må korreleres nøyaktig i rekkefølge ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)) |
| `8BITMIME` | tillater 8-bit MIME-overføring av brødtekst med `BODY=8BITMIME` | er ingen tillatelse for vilkårlige binærdata og endrer ikke headerreglene ([RFC 6152](https://datatracker.ietf.org/doc/html/rfc6152)) |
| `CHUNKING` | introduserer `BDAT` med eksplisitte byteantall som alternativ til `DATA` | ingen punkttransparens; forutsetning for `BINARYMIME` ([RFC 3030](https://datatracker.ietf.org/doc/html/rfc3030)) |
| `SMTPUTF8` | tillater internasjonaliserte postbokser og UTF-8 i definerte headere | hvert påkrevd etterfølgende hopp må støtte funksjonen, ellers oppstår en interoperabilitetsfeil ([RFC 6531](https://datatracker.ietf.org/doc/html/rfc6531)) |
| `DSN` | overfører parametere for leveringsvarsler ved `MAIL` og `RCPT` | skiller mellom ønskede suksess-, forsinkelses- og feilrapporter ([RFC 3461](https://datatracker.ietf.org/doc/html/rfc3461)) |
| `STARTTLS` | skifter fra klartekstdialogen til en TLS-kanal | tilbakestiller SMTP-tilstanden; sikkerheten avhenger ved MTA-overføring i tillegg av policy ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)) |
| `AUTH` | integrerer SASL-autentisering i SMTP og viser mekanismer | beregnet for innsending; tilbudte mekanismer kan endres etter STARTTLS ([RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954)) |

Capability Drift mellom klyngenoder er et vanlig driftsproblem. Hvis bare enkelte lyttere tilbyr `SMTPUTF8`, samme `SIZE`-verdi eller identiske AUTH-mekanismer, oppstår tilsynelatende tilfeldige leveringsfeil. Overvåking bør derfor ikke bare kontrollere én VIP, men hver relevant backendsti med samme SNI, EHLO-navn og kildenettverk.

Frem til nå har det handlet om transport mellom servere. Innsending fra brukere og applikasjoner er en egen tjeneste med andre porter, identiteter og misbruksrisikoer.

## Innsending, relay og autentisering

Port **25** brukes til MTA-til-MTA-overføring. Port **587** angir Message Submission; port **465** er registrert for innsending over TLS som er aktivt fra starten av, som tjenesten `submissions`. Port 465 er ikke en alternativ MX-port: Offentlig MX-levering har ingen portangivelse og bruker port 25 ([RFC 6409, avsnitt 1 og 3](https://datatracker.ietf.org/doc/html/rfc6409#section-3.1), [RFC 8314, avsnitt 7.3](https://datatracker.ietf.org/doc/html/rfc8314#section-7.3), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

En innsendingstjeneste kombinerer autentisering med autorisasjon. `AUTH` bruker SASL og kan bære mekanismer som PLAIN over TLS eller OAuth-baserte Bearer-tokener. Den autentiserte identiteten, den forespurte autorisasjonsidentiteten og den tillatte konvoluttavsenderen er ulike størrelser. En vellykket innlogging gir ikke automatisk rett til å bruke ethvert avsenderdomene eller enhver konvoluttavsender ([RFC 4422, avsnitt 3.4](https://datatracker.ietf.org/doc/html/rfc4422#section-3.4), [RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954), [RFC 7628](https://datatracker.ietf.org/doc/html/rfc7628)).

Åpent relay oppstår når en MTA videresender meldinger fra uautoriserte kilder til ikke-lokale mål. En sikker konfigurasjon avgjør derfor separat for lokale mottakerdomener, pålitelige kildenettverk, autentisert innsending og eksplisitte connectorer. IP-tillatlister er nettverksautorisering, ikke en erstatning for brukeridentitet; ved NAT, Cloudflare-egress eller delte relayer kan den frigi en langt større avsenderkrets enn tiltenkt.

Først når rolle og port er avklart, kan riktig TLS-policy bestemmes: Innsending skal være beskyttet og autentisert, mens åpen MTA-transport trenger ytterligere domene-policyer.

## TLS-policy for SMTP

STARTTLS beskytter den enkelte forbindelsen når begge parter utfører oppgraderingen. Ved åpen MTA-overføring forblir rent STARTTLS likevel opportunistisk: En aktiv angriper kan fjerne tilbudet eller manipulere DNS-mål, og en avsender uten ytterligere policy kan falle tilbake til klartekst. [DANE for SMTP](https://datatracker.ietf.org/doc/html/rfc7672) knytter TLSA-data via DNSSEC til valget av MX-mål. [MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) publiserer en PKIX-basert MX- og TLS-policy via DNS og HTTPS. [TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460) rapporterer aggregerte feil ved slike policyer. Kryptografiske detaljer, sertifikatkjeder og testmetoder behandles i artikkelen [TLS](/kb/tls).

Innsending er annerledes: Her er klient og tjeneste administrativt tilordnet, og brukeren skal ikke stille falle tilbake til klartekst. RFC 8314 anbefaler implisitt TLS for Mail Submission and Access og krever at klartekst behandles som foreldet. SMTP AUTH med passord- eller Bearer-mekanismer krever en tilstrekkelig beskyttet kanal; en server bør ikke engang tilby svake mekanismer før STARTTLS ([RFC 8314, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc8314#section-3), [RFC 4954, avsnitt 14](https://datatracker.ietf.org/doc/html/rfc4954#section-14)).

## Lokal levering med LMTP

**Local Mail Transfer Protocol** (LMTP) bruker en SMTP-lignende dialog for overlevering til et system uten egen langsiktig e-postkø, for eksempel en postbokstjeneste. Den vesentlige forskjellen vises etter meldingsinnholdet: LMTP gir et eget endelig svar for hver tidligere aksepterte mottaker. Dermed kan e-postlageret levere vellykket til én mottaker og avvise en annen på grunn av kvote eller en lokal feil, uten at den foranliggende MTA-en må behandle hele transaksjonen enhetlig ([RFC 2033, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc2033#section-4.2)).

LMTP flytter ikke ansvaret ut av systemet. Hvis e-postlageret midlertidig avviser en mottaker, må den foranliggende MTA-en fortsatt beholde denne mottakeren i køen. Med Unix-sockets faller riktignok IP-nettverksstien bort, men ikke behovet for å overvåke socket-tillatelser, backpressure, tidsavbrudd, per-recipient-svar og persistensgrensen.

## Implementerings- og teknologistakk

SMTP definerer protokollen på ledningen, ikke den interne programvarearkitekturen. I praksis kombinerer en MTA flere tekniske lag:

| Lag | Typiske komponenter | Kritisk driftstilstand |
|---|---|---|
| Tjenestevalg | DNS-resolver, MX/A/AAAA, statiske rutingtabeller | cache, DNSSEC-status, neste-hop-valg og tilkoblet IP |
| Nettverkstransport | TCP-stakk, lytter, lastbalanserer, proxy | forbindelsesgrenser, tidsavbrudd, bevaring av kilde-IP og backpressure |
| Kanalbeskyttelse | TLS-bibliotek, sertifikat- og policy-motor | SNI, truststore, DANE/MTA-STS-avgjørelse og fallback |
| Protokollfrontend | SMTP-parser, tilstandsautomat, AUTH/SASL, rate limits | økttilstand, tilbudte funksjoner og misbruksgrenser |
| Policy og innhold | adresseoppløsning, antispam, skadevare, DLP, signering | deterministisk rekkefølge, tidsavbrudd og atferd ved skannerfeil |
| Persistens | kø-/spool-filer, database eller Blob Store | durable write før `250`, ledig plass, replikering og gjenoppretting |
| Planlegger og ruting | gjenforsøkstidtakere, Destination Concurrency, transporttabeller | rettferdighet, backoff, køalder og blokkerte domener |
| Leveringsadapter | SMTP, LMTP, lokal postboks, fil, pipe eller API | per-recipient-status, idempotens og overleveringsgrense |
| Observerbarhet | strukturerte logger, kø-ID, måledata, spor og DSN | korrelasjon på tvers av hopp uten tap av sensitivt innhold |

Implementeringene realiserer disse lagene forskjellig. [Postfix](https://www.postfix.org/OVERVIEW.html) deler mottak, cleanup, Queue Manager og leveringsagenter i samarbeidende prosesser og skiller `incoming`, `active` og `deferred` som køtilstander. [Exim](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) lagrer konvolutt/headere og brødtekst i spoolen og lar queue-runnere behandle forfalte leveringsforsøk. [Apache James](/kb/apache-james) kjører på JVM og kombinerer protokollserver, kø og en Mailet-pipeline med utskiftbare lagringsbackender. Alle tre snakker SMTP, men prosess-, kø-, utvidelses- og gjenopprettingsmodellene deres er ikke utskiftbare.

Programmeringsspråk og runtime påvirker pakking, patchprosess, minneprofil og utvidbarhet, ikke semantikken i SMTP-dialogen. Postfix bygges med en nativ C-toolchain for Unix-lignende systemer ([Postfix INSTALL](https://www.postfix.org/INSTALL.html)); [Exim-dokumentasjonen](https://www.exim.org/exim-html-2.00/doc/html/oview.html) angir ANSI C som implementeringsspråk. Apache James beskriver seg derimot som en Java-/Scala-plattform på JVM ([Apache James](https://james.apache.org/)). En skytjeneste kan avbilde de samme RFC-rollene gjennom proprietære mikrotjenester, distribuerte køer og API-er. For driftsgodkjenning må derfor ikke bare EHLO-funksjoner, men også persistensløfte, gjenforsøksmodell, køeksport, sikkerhetskopi, nøkkelhåndtering, skaleringsgrenser og feilområder dokumenteres.

Fra leveringsforløpet følger en fast feilsøking: Bestem mål, kontroller forbindelse og SMTP-dialog, vurder svaret og følg deretter køen samt neste hopp.

## Diagnostikk fra DNS til køen

Pålitelig SMTP-diagnostikk følger leveringsstien og holder observasjonsnivåene atskilt. Tester bør utføres med testdomener og autoriserte mottakere; fremmede servere må ikke misbrukes til relay-, adressevaliderings- eller lasttester.

### Bestem MX-mål og adresser

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser publiserte DNS-data. Utdataene må kontrolleres for preferanse, Null MX, CNAME-kjeder, IPv4/IPv6 og resolveren det faktisk ble spurt mot. Et DNS-svar beviser verken lytter- eller TLS-funksjon.

### Kontroller TCP-tilgjengeligheten til valgt hopp

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer her bare DNS-oppløsning og TCP-etablering. En suksess på 25 sier ikke om SMTP-hilsenen kommer i tide; en suksess på 587 eller 465 sier ingenting om TLS- og AUTH-policy.

### Kontroller STARTTLS, EHLO og sertifikat

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) utfører den nødvendige klartekstdialogen med `-starttls smtp`, angir SMTP-EHLO-navnet med `-name` og SNI med `-servername`. `-verify_hostname` og `-verify_return_error` gjør den rene handshaken om til en eksplisitt PKIX-navnetest. Resultatet beskriver OpenSSL med sin truststore, ikke automatisk policyen til den produktive MTA-en.

### Test konvolutt og svarfase kontrollert

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten SMTP-Dialog">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`swaks`](https://jetmore.org/john/code/swaks/) logger SMTP-kommandoer og svar og kan bevisst avslutte testen etter `RCPT` uten å sende meldingsdata. Verktøyet er ikke en systemkomponent i Windows; `swaks.exe` eller en Perl-installasjon må komme fra en kontrollert kilde. En RCPT-test må bare utføres mot egne eller uttrykkelig godkjente systemer og adresser, fordi den utløser policy, katalogoppslag og rate limits.

### Undersøk lyttere og pakker på serveren

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-Listener- und Paketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) og [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) tilordner lyttere og TCP-tilstander til lokale prosesser. [`pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) og [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) fanger pakker; produksjonsopptak kan inneholde konvoluttadresser, ukryptert innhold og autentiseringsdata og må beskyttes, begrenses og slettes tilsvarende. [`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) egner seg for analyse av TCP, SMTP og TLS-handshaker, men erstatter verken kø- eller policylogger.

### Tilordne feil til en grense

| Observasjon | Sannsynlig grense | Neste pålitelige kontroll |
|---|---|---|
| ingen MX eller Null MX | DNS-publisering for mottakerdomenet | autoritativt svar, DNSSEC-status, negativ cache-TTL |
| TCP-timeout til én av flere MX-er | nettverkssti eller enkelt MX | hver MX-adresse og IP-familie separat, SYN/SYN-ACK og retursti |
| forbindelse etablert, ingen `220` | lytter, proxy, tarpit eller overbelastning | hilsenlatens, connection limits, frontend- og backendlogger |
| `STARTTLS` mangler bare på enkelte forbindelser | Capability Drift, proxy eller policy etter kildenettverk | hver klyngenode, EHLO før/etter TLS, lastbalanserersti |
| `4xx` ved `MAIL FROM` | avsender-, rate- eller ressurspolicy | fullstendig kode, kilde-IP, konvoluttavsender, kø-/policylogg |
| `5xx` ved én `RCPT TO` | mottaker- eller relayavgjørelse | berørt konvoluttmottaker, Accepted Domains, Directory Edge |
| `4xx` etter End-of-Data | skanner, lagring, kø eller Content Policy | lokal akseptpipeline, spoolplass, skannerlatens, ingen køovertakelse |
| `250` etter End-of-Data, senere DSN | etterfølgende levering mislyktes etter egen aksept | Reporting-MTA, ekstern MTA, kø-ID, DSN-status og opprinnelig mottaker |
| køen vokser for mange domener | lokal planlegger, DNS, nettverk eller ressurser | aldersfordeling, leveringsrater, aktive/deferred-tilstander, globale avhengigheter |
| køen vokser bare for ett domene | Remote-MX, TLS-policy eller Destination Throttling | siste feil per MX/IP, DANE/MTA-STS, samtidighet og gjenforsøkstid |
| postboks mangler til tross for vellykket SMTP-hopp | MDA/LMTP eller internt rutinghopp | siste pålitelige `Received`-linje, LMTP per-recipient-svar, lagerlogger |
| sporadiske duplikater etter timeout | uklart svar ved akseptgrensen | avbruddstidspunkt relativt til endelig `250`, korrelasjon mellom kø- og Message-ID |

## Observerbarhet og driftsmodell

En SMTP-monitor bør ikke bare sende en test-e-post. Den må synliggjøre systemets enkelte tilstander:

- Greeting-, EHLO-, STARTTLS-, AUTH- og End-of-Data-latens separat;
- forbindelser, økter, aktive transaksjoner og avvisninger per fase;
- tilbudte ESMTP-funksjoner per lytter og backend;
- akseptrate, mottakerantall, meldingsstørrelse og durable-write-latens;
- køomfang etter alder, mål, transport, status og siste feil;
- gjenforsøksrate, eldste melding, utløpende leveringer og DSN-opprettelse;
- DNS-mål, tilkoblet IP, IPv4/IPv6, TLS-modus, cipher og policyresultat;
- svarkoder som strukturerte felt i stedet for utelukkende som fritekst;
- korrelasjon av Edge-ID, intern kø-ID, ekstern kø-ID, `Message-ID` og mottaker;
- LMTP-/postboksstatus per mottaker samt karantene- og frigivelsesstier;
- spoolkapasitet, I/O-latens, database- eller brokerstatus og replikeringsetterslep;
- beskyttelsesgrenser for logger og opptak, fordi konvolutt, headere og innhold inneholder personopplysninger eller konfidensielle data.

Køsikkerhetskopier alene er ikke nok. Gjenoppretting må ta hensyn til konfigurasjon, rutingtabeller, sertifikater og nøkler, kømetadata, meldingsblober, karantene, signaturtilstand og idempotens. Etter en restore må en allerede levert mottakerkopi ikke ukontrollert sendes på nytt; samtidig må fortsatt åpne mottakere av en delvis levert melding ikke gå tapt. RPO og RTO må derfor defineres per tilstandskomponent og testes med en faktisk kø med flere mottakere.

## Teknisk historie

SMTP oppstod fra tidligere ARPANET-e-postprosedyrer. Jon Postels [RFC 821](https://datatracker.ietf.org/doc/html/rfc821) standardiserte i 1982 modellen som fortsatt er gjenkjennelig, med Sender-SMTP, Receiver-SMTP, `MAIL`, `RCPT`, `DATA` og tresifrede svar. Meldingsformatet ble samme år beskrevet separat i RFC 822. Dette skillet mellom overføringsprotokoll og meldingsformat ble bevart.

Med MIME vokste SMTP utover ren 7-biters tekst. ESMTP-rammeverket introduserte `EHLO`, registrerte funksjoner og parametere på `MAIL` henholdsvis `RCPT`; versjonen fra 1995 stod i [RFC 1869](https://datatracker.ietf.org/doc/html/rfc1869). Utvidelser bygget på dette, som `SIZE`, `8BITMIME`, `PIPELINING`, `AUTH`, `STARTTLS`, `CHUNKING`, DSN og senere `SMTPUTF8`, utviklet protokollen gradvis uten å opprette en parallell «SMTP 2»-stakk.

[RFC 2821](https://datatracker.ietf.org/doc/html/rfc2821) konsoliderte SMTP i 2001 og erstattet RFC 821 samt flere supplerende dokumenter. [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) fulgte i 2008 som en ytterligere konsolidering; det tilhørende Internet Message Format finnes i [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322). Message Submission ble først spesifisert separat og konsolidert i RFC 6409. Den tekniske utviklingen skjedde dermed gjennom tydeligere roller, utvidelsesregistre, internasjonaliserte adresser, leveringsrapporter og transport-policy, mens den grunnleggende tilstandsbaserte dialogen forble kompatibel.

Også programvarearkitekturene utviklet seg i ulike retninger. Historisk Sendmail-kompatibilitet preget lokale kommandolinjegrensesnitt; Postfix satset på samarbeidende prosesser med separerte privilegier og eksplisitte køer, Exim på en sterkt konfigurerbar router-/transport- og spooltilnærming, og JVM-systemer som Apache James gjorde behandlingen utvidbar gjennom komponenter og Mailets. Skyplattformer fordeler de samme logiske rollene på frontender, brokere, policytjenester og regionale køer. For administratorer forblir derfor RFC-semantikken fellesnevneren, mens persistens, skalering og gjenoppretting alltid må dokumenteres produktspesifikt.

## Kilder

- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321) – SMTP-modell, ruting, kommandoer, svar, kø, gjenforsøk, sporing og leveringsansvar.
- [RFC 5598, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc5598)
- [RFC 6409 – Message Submission for Mail](https://datatracker.ietf.org/doc/html/rfc6409) – skille mellom innsending og relay, roller, port 587 og MSA-policy.
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322) – headere, brødtekst, adressefelt, Message-ID og sporingsfelt.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS og tilbakestilling av SMTP-tilstanden.
- [RFC 2920 – SMTP Command Pipelining](https://datatracker.ietf.org/doc/html/rfc2920) – ordnede kommandogrupper uten round trip per kommando.
- [RFC 7505 – A Null MX No Service Resource Record](https://datatracker.ietf.org/doc/html/rfc7505) – uttrykkelig markering av domener som ikke kan motta e-post.
- [RFC 3464 – Delivery Status Notification Format](https://datatracker.ietf.org/doc/html/rfc3464) – maskinlesbar DSN-struktur og mottakerstatus.
- [IANA – SMTP Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml) – registrerte betydninger av detaljkodene.
- [RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463) – struktur og klasser for utvidede statuskoder.
- [RFC 5248 – SMTP Enhanced Status Code Registry](https://datatracker.ietf.org/doc/html/rfc5248) – registreringsmodell for Enhanced Codes.
- [IANA – Simple Mail Transfer Protocol Parameters](https://www.iana.org/assignments/smtp/smtp.xhtml) – SMTP-kommandoer, utvidelser og EHLO-nøkkelord.
- [RFC 1870 – SMTP SIZE Extension](https://datatracker.ietf.org/doc/html/rfc1870) – størrelsesforhandling og avvisning før innholdet.
- [RFC 6152 – SMTP Service Extension for 8-bit MIME Transport](https://datatracker.ietf.org/doc/html/rfc6152) – 8BITMIME og BODY-parametere.
- [RFC 3030 – CHUNKING and BINARYMIME](https://datatracker.ietf.org/doc/html/rfc3030) – BDAT-overføring og binære MIME-meldinger.
- [RFC 6531 – SMTP Extension for Internationalized Email](https://datatracker.ietf.org/doc/html/rfc6531) – SMTPUTF8 og internasjonale postbokser.
- [RFC 3461 – SMTP Service Extension for Delivery Status Notifications](https://datatracker.ietf.org/doc/html/rfc3461) – konvoluttparametere for DSN-forespørsler.
- [RFC 4954 – SMTP Service Extension for Authentication](https://datatracker.ietf.org/doc/html/rfc4954) – AUTH-kommando og SASL-profil for SMTP.
- [RFC 8314, avsnitt 7.3](https://datatracker.ietf.org/doc/html/rfc8314)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – tilordning av SMTP, Submission og Submissions til porter.
- [RFC 4422, avsnitt 3.4](https://datatracker.ietf.org/doc/html/rfc4422)
- [RFC 7628 – SASL Mechanisms for OAuth](https://datatracker.ietf.org/doc/html/rfc7628) – OAuth-baserte SASL-mekanismer for SMTP og andre protokoller.
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – DNSSEC- og TLSA-bundet SMTP-transportpolicy.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – PKIX-basert MX- og TLS-policy.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – aggregerte rapporter om transport-TLS-feil.
- [RFC 2033, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc2033)
- [Postfix – Architecture Overview](https://www.postfix.org/OVERVIEW.html) – prosessroller, Queue Manager og leveringsagenter.
- [Exim – How Exim receives and delivers mail](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) – spool-, ruting-, transport- og queue-runner-modell.
- [Postfix – Installation From Source Code](https://www.postfix.org/INSTALL.html) – native build-, plattform- og toolchain-krav.
- [Exim – teknisk oversikt](https://www.exim.org/exim-html-2.00/doc/html/oview.html) – ANSI C som implementeringsspråk og tekniske produktegenskaper.
- [Apache James Project](https://james.apache.org/) – JVM-, Java-/Scala- og komponentarkitektur.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-oppslag i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS-oppslag i Unix-systemer.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-tilgjengelighet i Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – TCP-forbindelser med Netcat.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – STARTTLS-, SNI- og sertifikatdiagnostikk.
- [Swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/) – SMTP-transaksjons- og utvidelsestest.
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) – lokale TCP-lyttere og forbindelser.
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – socket- og TCP-tilstandsdiagnostikk.
- [Microsoft Learn – Packet Monitor](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – Windows-pakkefangst og PCAPNG-konvertering.
- [tcpdump(1) manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – pakkebasert diagnostikk på Unix-systemer.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – interaktiv analyse av nettverks- og protokollspor.
- [RFC 821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc821) – historisk SMTP-spesifikasjon fra 1982.
- [RFC 1869 – SMTP Service Extensions](https://datatracker.ietf.org/doc/html/rfc1869) – historisk ESMTP-rammeverk og EHLO-modell.
- [RFC 2821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc2821) – SMTP-konsolidering fra 2001.
- [RFC 3848 – ESMTP and LMTP Transmission Types](https://datatracker.ietf.org/doc/html/rfc3848) – `with`-verdier som ESMTPSA i Received-headere.
