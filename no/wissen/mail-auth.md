---
title: "E-postautentisering: SPF, DKIM og DMARC"
blatt: "mail-auth"
description: "SPF, DKIM og DMARC i teknisk sammenheng: identiteter, DNS-evaluering, signaturer, alignment, policyer, rapportering, videresendinger, tillitsgrenser og drift for meldingsadministratorer."
fakten:
  - label: SPF-formål
    wert: IP autoriserer RFC5321.MailFrom eller HELO
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-2
  - label: SPF-publisering
    wert: én TXT-record med v=spf1
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-3
  - label: SPF-DNS-budsjett
    wert: høyst 10 termer som utløser oppslag
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4
  - label: DKIM-formål
    wert: domenesignatur over utvalgte hoder og brødtekst
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3
  - label: DKIM-nøkkel
    wert: selector._domainkey.signing-domain
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3.6.2.1
  - label: DKIM-kryptografi
    wert: RSA-SHA256 eller Ed25519-SHA256
    href: https://datatracker.ietf.org/doc/html/rfc8463#section-3
  - label: DMARC-standard
    wert: RFC 9989; rapportering i RFC 9990 og 9991
    href: https://datatracker.ietf.org/doc/html/rfc9989
  - label: DMARC-identitet
    wert: ett Author Domain fra nøyaktig ett From-felt
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.2
  - label: DMARC-pass
    wert: SPF eller DKIM består og er aligned
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Alignment
    wert: relaxed som standard; strict valgfritt
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Policyer
    wert: none · quarantine · reject
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.7
  - label: Driftsbevis
    wert: Authentication-Results pluss Aggregate Reports
    href: https://datatracker.ietf.org/doc/html/rfc8601
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 1247c1771c81b476bf23da2eeee6feb35a3d16c7146e2c61d6c0fa5625c55bbd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:33:37.053Z
translationReview: automatic
---

# E-postautentisering: SPF, DKIM og DMARC

SPF, DKIM og DMARC besvarer tre ulike spørsmål om avsenderdomenet som brukes. SPF kontrollerer den sendende IP-adressen, DKIM en kryptografisk signatur og DMARC forholdet mellom begge resultatene og det synlige From-domenet. Ingen av disse mekanismene autentiserer en person eller beviser at en melding er ufarlig. Et DMARC-pass sier kun at bruken av Author Domain er autorisert i henhold til de publiserte reglene ([RFC 7208, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc7208#section-1), [RFC 6376, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc6376#section-1), [RFC 9989, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc9989#section-1)).

For meldingsadministratorer er systemet først og fremst en kjede av ansvarsområder. Den utgående tjenesten må opprette passende envelope-domener og DKIM-signaturer. Den autoritative [DNS](/kb/dns) må levere SPF-policy, DKIM-nøkler og DMARC-policy korrekt og til rett tid. Den mottakende [SMTP](/kb/smtp)-grenseserveren trenger den opprinnelige klient-IP-en, en resolver, kryptografisk verifikasjon og en definert tillitsgrense for resultatene. En report collector må behandle ikke-pålitelig XML sikkert, oppdage duplikater og gjøre data analyserbare over tid. Et `dmarc=pass` er bare sluttresultatet av denne distribuerte pipeline-en.

Forklaringen følger identitetene til en e-post: envelope-avsender, synlig From-domene og DKIM-signatur. SPF og DKIM forklares først hver for seg, deretter kobler DMARC resultatene deres gjennom alignment, policy og rapportering.

## Identiteter og tillitsgrenser

En melding inneholder flere avsenderbegreper som ikke kan byttes om. **RFC5321.MailFrom** er SMTP-envelopeens Reverse Path og det primære SPF-objektet. Ved tom Reverse Path, slik den er beregnet for leveringsrapporter, utleder SPF identiteten fra HELO/EHLO. **RFC5322.From** står i meldingshodet, vises av e-postklienten og gir DMARC Author Domain. En DKIM-signatur angir med `d=` sitt Signing Domain og med `s=` selektoren for den offentlige nøkkelen. SMTP AUTH autentiserer på sin side en klient mot en submission-tjeneste, men er verken SPF, DKIM eller DMARC ([RFC 5321, avsnitt 3.3 og 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321#section-3.3), [RFC 5322, avsnitt 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2), [RFC 7208, avsnitt 2.3 og 2.4](https://datatracker.ietf.org/doc/html/rfc7208#section-2.3), [RFC 6376, avsnitt 3.5](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

| Identitet | Kilde | Kontrollør | Primær utsagn |
|---|---|---|---|
| tilkoblet IP-adresse | TCP-forbindelse ved mottakende MTA | SPF | denne verten er eller er ikke autorisert for det kontrollerte SMTP-domenet |
| HELO/EHLO-domene | SMTP-dialog | SPF | domeneidentitet til SMTP-klienten |
| RFC5321.MailFrom-domene | SMTP-envelope | SPF og DMARC | bounce- eller Return-Path-domene |
| DKIM `d=` og `s=` | `DKIM-Signature` | DKIM og DMARC | Signing Domain og nøkkelselektor |
| RFC5322.From-domene | synlig meldingshode | DMARC | Author Domain som alignment må etableres mot |
| `authserv-id` | `Authentication-Results` | interne konsumenter | hvilken pålitelig kontrolltjeneste som opprettet resultatet |

Dette skillet er en sikkerhetsgrense. En angriper kan bestå SPF og DKIM fullt ut med sitt eget domene og likevel oppgi et fremmed varemerke i visningsnavnet. DMARC begrenser uautorisert bruk av domenet i det synlige `From:`, ikke look-alike-domener, svindel med visningsnavn, kompromitterte legitime avsendere eller skadelig innhold ([RFC 7208, avsnitt 11.2](https://datatracker.ietf.org/doc/html/rfc7208#section-11.2), [RFC 9989, avsnitt 2.2 og 11.4](https://datatracker.ietf.org/doc/html/rfc9989#section-11.4)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-mail-auth.svg?v=20260813" title="Interaktive Infografik: Identitäten, Alignment, DMARC-Entscheidung, Reporting und indirekte Nachrichtenflüsse" loading="lazy">
  <a href="/images/kb-interaktiv-mail-auth.svg?v=20260813">Åpne infografikk om SPF, DKIM og DMARC</a>
</iframe>

## SPF: Autorisering av den tilkoblede IP-adressen

Sender Policy Framework er en DNS-basert autorisering for domenet i `MAIL FROM` eller HELO. Mottakeren evaluerer klient-IP-en, det kontrollerte domenet, avsenderidentiteten og det lokale vertsnavnet med funksjonen `check_host()` definert i RFC 7208. Posten ligger som en TXT-ressurs direkte på det aktuelle domenet og begynner med `v=spf1`; den utgåtte DNS-RR-typen SPF brukes ikke ([RFC 7208, avsnitt 3.1 og 4.1](https://datatracker.ietf.org/doc/html/rfc7208#section-3.1)).

En post som `v=spf1 ip4:192.0.2.0/24 include:_spf.sender.example -all` evalueres fra venstre mot høyre. En mekanisme som samsvarer, avslutter behandlingen med sin qualifier. `+` betyr `pass` og er standarden, `-` betyr `fail`, `~` betyr `softfail`, `?` betyr `neutral`. `all`, `ip4` og `ip6` krever ingen ytterligere DNS-oppslag under normal evaluering; `include`, `a`, `mx`, `ptr`, `exists` og `redirect` gjør det ([RFC 7208, avsnitt 4.6.1 til 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6)).

| Resultat | Protokollbetydning | Admin-spørsmål |
|---|---|---|
| `pass` | IP-en er autorisert for denne SPF-identiteten | Er akkurat dette domenet også aligned med RFC5322.From? |
| `fail` | den publiserte policyen autoriserer ikke IP-en | feil kilde, stjålet domenebruk eller utdatert policy? |
| `softfail` | svak negativ uttalelse fra domenet | tjener den fortsatt en kontrollert overgangsfase, eller skjuler den drift? |
| `neutral` | ingen autoriseringsuttalelse | mangler en avsluttende mekanisme, eller er nøytralitet tilsiktet? |
| `none` | ingen anvendbar SPF-policy | ble riktig MailFrom-/HELO-domene kontrollert? |
| `temperror` | midlertidig feil, vanligvis DNS | kontroller resolver, timeout og autoritativ tilgjengelighet |
| `permerror` | posten eller evalueringen er varig ugyldig | kontroller syntaks, flere SPF-poster, rekursjon og oppslagsbudsjett |

### Oppslagsbudsjett og avhengigheter

SPF begrenser summen av `include`, `a`, `mx`, `ptr`, `exists` og `redirect` til ti termer over hele den rekursive evalueringen. En overskridelse må gi `permerror`. For `mx` og `ptr` gjelder ytterligere adressegrenser; mer enn to tomme svar eller NXDOMAIN-resultater, såkalte Void Lookups, bør også føre til `permerror` ([RFC 7208, avsnitt 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4)).

Budsjettet er en kjøretidsgrense og ikke bare en tegnkontroll. Ett enkelt `include` kan introdusere flere includes, MX-oppløsninger og feilområder. `include` delegerer bare spørsmålet om den aktuelle verten oppnår `pass` der; `redirect` overtar hele policyen til et annet domene etter mislykket mekanismekontroll. RFC 7208 anbefaler `include` for å krysse administrative grenser og `redirect` heller for å sentralisere domener som forvaltes likt ([RFC 7208, avsnitt 5.2 og 6.1](https://datatracker.ietf.org/doc/html/rfc7208#section-5.2)). Derfor skal eier, formål, endringsvei og målt worst-case-budsjett for hvert eksterne include inngå i inventaret.

### Videresending og SPF-domene

En klassisk videresender kobler seg til neste mottaker med sin egen IP, men beholder den opprinnelige `MAIL FROM`. Dermed kontrolleres en IP mot SPF-policyen til et fremmed domene, og SPF kan feile selv om den opprinnelige innleveringen var legitim. RFC 7208 beskriver omskriving av Reverse Path til et domene hos formidleren som et mottiltak; e-postlister gjør ofte dette uansett ([RFC 7208, vedlegg D.2](https://datatracker.ietf.org/doc/html/rfc7208#appendix-D.2)). Dette løser SPF-pass for det nye envelope-domenet, men gir bare DMARC-pass dersom dette nye domenet er aligned med det synlige `From:`. For indirekte veier er en bevart, aligned DKIM-signatur derfor særlig viktig.

## DKIM: Signatur fra et domene

DomainKeys Identified Mail legger til et `DKIM-Signature`-headerfelt. Signatøren kanoniserer utvalgte hoder og brødteksten, danner brødtekst-hashen `bh=`, signerer de fastsatte dataene og publiserer nøkkelen under `s=._domainkey.d=`. Verifikatoren rekonstruerer de samme dataene, slår opp DNS-TXT-nøkkelen og kontrollerer signaturen og brødtekst-hashen. Et pass beviser at de signerte delene ikke er endret på en gjenkjennbar måte siden signeringen, og at signatøren kontrollerte den private nøkkelen til Signing Domain. DKIM beviser verken en fysisk person eller sannheten i innholdet ([RFC 6376, avsnitt 3.5 til 3.8](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

Viktige tagger er `a=` for algoritme, `c=` for header-/brødtekst-kanonisering, `d=` for Signing Domain, `s=` for selektor, `h=` for de signerte headernavnene, `bh=` for brødtekst-hash og `b=` for signaturen. `t=` og `x=` kan overføre signeringstidspunkt og utløpstidspunkt, men er ikke pålitelig replay-beskyttelse. Den valgfrie `l=` begrenser det signerte brødtekstområdet og kan dermed gjøre det mulig å legge til usignert innhold; RFC 6376 beskriver uttrykkelig denne misbruksflaten ([RFC 6376, avsnitt 3.5 og 8.2](https://datatracker.ietf.org/doc/html/rfc6376#section-8.2)).

Kanoniseringen `simple` tolererer nesten ingen endring. `relaxed` normaliserer blant annet bestemte skrivemåter og whitespace, slik at vanlige transportendringer ikke unødig bryter en signatur. Hoder og brødtekst kan bruke ulike metoder; uten `c=` gjelder `simple/simple`. Kanonisering endrer ikke den overførte meldingen, men bare inndataformen for signering eller verifikasjon ([RFC 6376, avsnitt 3.4](https://datatracker.ietf.org/doc/html/rfc6376#section-3.4)).

### Nøkler, algoritmer og rotasjon

RFC 8301 krever minst 1024 bit for RSA, anbefaler minst 2048 bit for signatører og erklærer RSA-SHA1 som historisk. RFC 8463 legger til Ed25519-SHA256 og tillater parallelle signaturer med ulike selektorer for overgangskompatibilitet ([RFC 8301, avsnitt 3.1 og 3.2](https://datatracker.ietf.org/doc/html/rfc8301#section-3), [RFC 8463, avsnitt 5 og 6](https://datatracker.ietf.org/doc/html/rfc8463#section-5)). Hvilken algoritme som brukes, er fortsatt en interoperabilitetsbeslutning: En standardisert plikt på verifikatorsiden beviser ikke automatisk at hver reelle mottaksplattform implementerer den feilfritt.

Selektorer skiller nøkkelbytte fra domenet. Ved rotasjon publiseres først den nye offentlige nøkkelen, deretter signeres det med den nye private nøkkelen, og den gamle DNS-nøkkelen fjernes først når gamle meldinger ikke lenger må kunne kontrolleres regelmessig. RFC 6376 fraråder å gjenbruke en selektor med en ny nøkkel, fordi gamle signaturfeil da ikke kan skilles fra forfalskninger. En tom `p=` i nøkkelposten tilbakekaller nøkkelen ([RFC 6376, avsnitt 3.1 og 6.1.2](https://datatracker.ietf.org/doc/html/rfc6376#section-3.1)).

Den private nøkkelen hører ikke hjemme i DNS eller i generelle konfigurasjonsrepositorier. Driftsdesignet må fastsette nøkkeleier, generering, beskyttet lagring, signatørens tilgang, rotasjon, nødtilbakekalling, backupbeslutning og revisjonsspor. RFC 6376 krever omhu ved beskyttelse av private nøkler og nevner kryptert lagring og kryptografisk maskinvare som mulige beskyttelsestiltak. Flere utsendelsesplattformer bør ha separate selektorer, slik at en kompromittert plattform kan isoleres uten globalt nøkkelbytte. Denne arkitekturen følger den administrative oppdelingen av selektornavnerommet som DKIM legger opp til ([RFC 6376, avsnitt 3.1 og 8.3 samt vedlegg C](https://datatracker.ietf.org/doc/html/rfc6376#section-8.3)).

SPF kan feile etter videresending selv om meldingen forble uendret. DKIM kan derimot overleve en videresending, men bryte på en endret footer. DMARC kobler derfor begge metodene gjennom såkalt alignment.

## DMARC: Alignment, policy og evaluering

DMARC forutsetter ett enkelt, korrekt formatert RFC5322.From-felt og trekker ut nøyaktig ett **Author Domain** fra det. Det vurderer det SPF-autentiserte domenet og alle vellykket verifiserte DKIM Signing Domains. DMARC-pass foreligger når minst ett SPF- eller DKIM-resultat er `pass` og domenet er aligned med Author Domain. Begge mekanismene trenger altså ikke å bestå samtidig; i drift er begge ønskelige fordi de kan feile på ulike indirekte flyter ([RFC 9989, avsnitt 4.2 til 4.4 og 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-4.2)).

Ved **strict alignment** må domenene være identiske. Ved **relaxed alignment** må de ha samme Organizational Domain. `adkim=s` eller `aspf=s` krever strict; uten disse taggene gjelder relaxed. Fastsettelsen av Organizational Domain skjer etter RFC 9989 gjennom en begrenset DNS Tree Walk og ikke lenger bare gjennom en Public Suffix List. Walk-en spør maksimalt åtte navnenivåer og tar hensyn til `psd=y` eller `psd=n` ([RFC 9989, avsnitt 4.4 og 4.10](https://datatracker.ietf.org/doc/html/rfc9989#section-4.10)). Denne endringen er relevant for blandede gamle og nye verifikatorer; RFC 9989 peker selv på mulige ulike alignment-resultater.

### Policy-record og tagger

DMARC-posten ligger under `_dmarc.<domain>` og bruker tag/verdi-syntaks. `v=DMARC1` angir formatet. `p=` beskriver ønsket behandling av mislykkede meldinger fra policy-domenet: `none`, `quarantine` eller `reject`. `sp=` kan behandle eksisterende underdomener annerledes, `np=` ikke-eksisterende underdomener. `rua=` angir mål for Aggregate Reports, `ruf=` valgfrie mål for Failure Reports. `t=y` markerer testmodusen definert i RFC 9989. Ukjente tagger ignoreres ([RFC 9989, avsnitt 4.5 til 4.8](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)).

Den tidligere prosentvise utrullingen med `pct=` hører ikke lenger til protokollen etter RFC 9989. Driftserfaring viste inkonsekvent anvendelse; `t=` erstatter bare den tidligere spesialvirkningen av `pct=0`, ikke vilkårlige prosenttrinn. Trinnvis innføring må derfor planlegges gjennom separate Author Domains, underdomene-policyer, målrettede e-poststrømmer og solid rapportevaluering, ikke gjennom en angivelig standardisert prosentregulator ([RFC 9989, vedlegg A.6](https://datatracker.ietf.org/doc/html/rfc9989#appendix-A.6)).

En policy er en publisert preferanse fra Domain Owner. Mottakeren kan avvike på grunn av lokal policy, omdømme, indirekte flyter eller annen innsikt, og dokumenterer eventuelt overstyringer i rapporter. `p=reject` gjør ikke automatisk en melding uleverbar i SMTP-forstand hos enhver mottaker; det gir en entydig, automatiserbar uttalelse for håndtering av DMARC-fail ([RFC 9989, avsnitt 5.3 og 5.4](https://datatracker.ietf.org/doc/html/rfc9989#section-5.3)).

En DMARC-policy bør først strammes inn når alle legitime utsendelsesveier er kjent. Aggregate Reports leverer dataene for denne kartleggingen og viser hvilke systemer som faktisk sender under et domene.

## Rapportering som operativ datapipeline

RFC 9990 skiller Aggregate Reporting fra DMARC-kjernespesifikasjonen. En rapport oppsummerer meldinger etter kilde-IP, evaluert policy, disposition, alignment og autentiseringsresultater. Dataformatet er XML; filen skal komprimeres med GZIP og får da filendelsen `.xml.gz`. Enkelte poster inneholder blant annet `source_ip`, `count`, `header_from`, SPF- og DKIM-resultater samt mulige grunner til policy-overstyring ([RFC 9990, avsnitt 3.1 og 3.4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.1)).

En collector er derfor en sikkerhetsrelevant ingest-tjeneste. Den mottar uoppfordret e-post og vedlegg, dekomprimerer fremmede data, parser XML, dedupliserer rapporter og aggregerer volum. Størrelsesgrenser, dekomprimeringsbudsjett, XML-parser uten eksterne entiteter, skadevarebeskyttelse, karantene for feilaktige filer, idempotens og oppbevaring er en del av arkitekturen. RFC 9989 advarer mot bevisst feilaktige rapporter og DoS mot rapporteringsmål; RFC 9990 regulerer duplikater og eksterne mål ([RFC 9989, avsnitt 11.2](https://datatracker.ietf.org/doc/html/rfc9989#section-11.2), [RFC 9990, avsnitt 3.5.4 og 4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.5.4)).

Hvis `rua=` ligger utenfor Organizational Domain, må rapportmottakeren autorisere dette forholdet gjennom en ekstra DNS-TXT-post. Slik kan ikke et domene oversvømme vilkårlige tredjeparter med rapporter ([RFC 9990, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc9990#section-4)). Failure Reports etter RFC 9991 kan avsløre informasjon om enkeltmeldinger. På grunn av personvernrisikoene begrenser mange operatører dem; Aggregate Reports er den anbefalte synlighetskanalen uten sluttbrukerinnhold ([RFC 9991, avsnitt 7](https://datatracker.ietf.org/doc/html/rfc9991#section-7)).

For administratoranalyse er tidsserier viktigere enn én enkelt dagsverdi: volum per kilde-IP og Author Domain, aligned SPF, aligned DKIM, DMARC-fail, `temperror`, `permerror`, ukjente selektorer, policy-overstyringer og rapportørdekning. Rapporter er observasjoner fra enkeltmottakere og ikke et fullstendig utsendelsesregnskap; mottakere er ikke forpliktet til å levere hver ønsket evaluering eller disposition ([RFC 9989, avsnitt 1 og 6](https://datatracker.ietf.org/doc/html/rfc9989#section-6)).

## Stol riktig på Authentication-Results

Den mottakende kontrolltjenesten kan lagre SPF-, DKIM- og DMARC-resultater i headerfeltet `Authentication-Results`. `authserv-id` identifiserer tjenesten eller Administrative Management Domain som har kontrollert. Dette feltet har bare bevisverdi innenfor en definert tillitsgrense. En ekstern avsender kan selv sette inn et overbevisende `Authentication-Results: ... dmarc=pass` ([RFC 8601, avsnitt 1.5.4 til 1.6](https://datatracker.ietf.org/doc/html/rfc8601#section-1.5.4)).

Border-MTA-en må derfor fjerne fremmede forekomster eller bare tillate uttrykkelig pålitelige produsenter. Interne konsumenter trenger en liste over tillatte `authserv-id`-verdier og må ta hensyn til plasseringen i den pålitelige `Received`-kjeden eller en intern metadata-proveniensmodell. RFC 8601 advarer uttrykkelig mot å aktivere headerfeltet for filterbeslutninger uten en verifisert, samsvarende Border-MTA ([RFC 8601, avsnitt 5 og 7.1](https://datatracker.ietf.org/doc/html/rfc8601#section-5)).

ARC etter RFC 8617 kan transportere autentiseringsresultater og påfølgende endringer over formidlere i en signert kjede. ARC er publisert som en eksperimentell protokoll og gir ingen global tillitsrot: Sluttmottakeren avgjør fortsatt hvilke ARC-sealere den stoler på. En gyldig ARC-kjedestatus er derfor kontekst for lokal policy, ikke en erstatning for DMARC og ikke en automatisk akseptbeslutning ([RFC 8617, avsnitt 1 og 5](https://datatracker.ietf.org/doc/html/rfc8617#section-1)).

Frem til nå har det handlet om direkte levering. Videresendinger, lister og gatewayer endrer imidlertid IP-adresse, envelope eller innhold – og dermed nettopp inndataene til de tre kontrollene.

## Indirekte flyter og typiske bruddsteder

Videresendinger, e-postlister, sikkerhetsgatewayer og ticketing-systemer endrer ulike deler av meldingen. En videresending endrer den tilkoblede IP-en og kan bryte SPF. En e-postliste kan endre Subject, List-headere, brødtekst-footer eller MIME-struktur og dermed bryte DKIM. Den kan i tillegg omskrive envelope-avsenderen og det synlige `From:`. RFC 7960 beskriver disse interoperabilitetsproblemene og de respektive bivirkningene; det finnes ingen universell korreksjon som samtidig lar identitet, listefunksjon og eksisterende mottakerpolicy forbli uendret ([RFC 7960, avsnitt 3 og 4](https://datatracker.ietf.org/doc/html/rfc7960#section-3)).

For feilsøking må administratorer derfor sammenligne tilstanden **før og etter hver formidler**: klient-IP, HELO, MailFrom, RFC5322.From, eksisterende DKIM-signaturer, `Authentication-Results`, nye `Received`-felt og innholdsmutasjoner. Et sluttresultat `dmarc=fail` viser ikke alene hvilket hopp som mistet den aligned identiteten.

## Teknisk oppbygning av en produksjonsplattform

E-postautentisering er ikke én enkelt appliance, men et distribuert kontroll- og dataplan:

| Komponent | Teknologistack | Persisten tilstand | Sentralt feilområde |
|---|---|---|---|
| autoritativ DNS | TXT-RRsets, DNSSEC valgfritt, sone- eller API-basert endring | SPF, DKIM Public Keys, DMARC Policy | utdaterte poster, Split-Horizon, TTL, feilaktig delegering |
| Outbound-Signer | MTA-filter, bibliotek eller gateway; RSA/Ed25519 og SHA-256 | Private Keys, selektorkonfigurasjon, signeringspolicy | nøkkeltilgang, feil `d=`, manglende signatur på delstrømmer |
| Inbound-Verifier | Border-MTA/filter, rekursiv resolver, krypto-bibliotek, policy-motor | resolvercache, tillitsgrense, lokale overstyringer | feil klient-IP etter proxy, DNS-timeout, manipulerte Auth-Results |
| DMARC-policy-modul | Alignment, DNS Tree Walk, domene-/policylogikk | cache og policy-versjon | gammel RFC-7489-modell, feilaktig Organizational Domain |
| Report-Generator | MTA-telemetri, aggregator, XML/GZIP, SMTP-sending | tidsvindu, poster, report-ID | datatap, duplikater, ufullstendig rapportørdekning |
| Report-Collector | postboks, dekompressor, XML-parser, database, dashboard | rårapporter, normalisering, tidsserier | parserangrep, feil deduplisering, ukontrollert oppbevaring |

Programmeringsspråk og produkt kan byttes ut, men ikke protokollobjekter og tillitsgrenser. En MTA kan implementere signering og kontroll i C, Rust, Java, Go eller gjennom en separat filterprosess. De samme spørsmålene er avgjørende for drift: Hvor kommer klient-IP-en fra etter lastbalanserere? Hvilken resolver og cache brukes? Hvor ligger den private nøkkelen? Hvilken prosess kan signere? Hvilke headere fjernes før tillitsgrensen? Hvordan korreleres policy-versjon, DNS-svar, Message-ID, Queue-ID og resultat? RFC-ene definerer wire- og evalueringssemantikk, ikke den konkrete prosessmodellen.

### Kontrollert innførings- og endringsprosess

RFC 9989 beskriver en robust rekkefølge for Domain Owners: publiser aligned SPF, konfigurer aligned DKIM, opprett postboks for Aggregate Reports, publiser først DMARC i Monitoring Mode med `p=none`, evaluer rapporter, utbedre mangler og avgjør først deretter enforcement ([RFC 9989, avsnitt 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-5.1)). Dette gir en etterprøvbar prosess for Change Management:

1. Inventariser alle Author Domains, envelope-domener, HELO-navn, utsendelsesprodukter, leietakere, relayer, videresendere og nøkkeleiere.
2. Påvis minst én aligned identitet for hver legitim e-poststrøm; SPF og DKIM sammen reduserer avhengigheten av én enkelt indirekte vei.
3. Gjør rapportmottak og sikker evaluering produksjonsklar før `rua=`.
4. Driv `p=none` som målefase, men ikke forveksle den med en beskyttende effekt.
5. Klassifiser ukjente kilder: legitime og feilkonfigurerte, avviklede, misbrukende eller forvrenget av indirekte flyt.
6. Innfør enforcement per kontrollerbart domene, dokumenter unntak og følg med ved hjelp av rapporter og leveringstelemetri.
7. Øv jevnlig på nøkkelrotasjon, leverandørbytte, DNS-rollback, rapportfeil og kompromittert signatør.

En godkjenning bør ikke bare akseptere en syntaktisk gyldig TXT-post. Den krever testmeldinger via hver utsendelsesvei, headerbevis hos mottakeren, rapportdata fra flere mottakerdomener, måling av SPF-budsjettet, kontroll av selector-TTL-er og en rollback som ikke etterlater Author Domain ubeskyttet eller uleverbar.

Av disse avhengighetene følger en fast diagnoserekkefølge: identifiser først domenene som brukes, kontroller deretter DNS og signatur, og vurder til slutt alignment og DMARC-policy.

## Diagnoseverktøy

Følgende oppslag bruker `example.ch` og eksempelselektoren `s2026a`. Produksjonsnavn og lokalt lagrede meldinger må bare undersøkes i autoriserte miljøer.

### Les SPF, DMARC og DKIM i DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail-Authentifizierungs-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type TXT -DnsOnly
Resolve-DnsName _dmarc.example.ch -Type TXT -DnsOnly
Resolve-DnsName s2026a._domainkey.example.ch -Type TXT -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer example.ch TXT
dig +noall +answer _dmarc.example.ch TXT
dig +noall +answer s2026a._domainkey.example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser TXT-RRsettene, ikke deres fullstendige protokollevaluering. Flere Character Strings i én enkelt TXT-post må settes sammen uten ekstra tegn; flere SPF- eller DMARC-poster på samme navn er derimot en feil ([RFC 7208, avsnitt 3.2 og 4.5](https://datatracker.ietf.org/doc/html/rfc7208#section-3.2), [RFC 9989, avsnitt 4.5](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)). For Split-Horizon- eller DNSSEC-spørsmål må autoritativ og rekursiv visning kontrolleres separat.

### Finn pålitelige resultater i meldingshodet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Header-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\message.eml |
  Select-String -Pattern '^(Authentication-Results|DKIM-Signature|Received):' -Context 0,8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">grep -E -A8 '^(Authentication-Results|DKIM-Signature|Received):' message.eml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) og [`Select-String`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) eller [`grep`](https://www.gnu.org/software/grep/manual/grep.html) hjelper med den første gjennomgangen. På grunn av header folding og flere felt erstatter tekstsøk ikke en RFC-kompatibel parser. Avgjørende er den pålitelige `authserv-id`, plasseringen relativt til egen `Received`-grense, domenene som faktisk ble kontrollert, og å skille mellom råresultat og alignment.

### Kontroller Aggregate Report strukturelt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DMARC-XML-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">[xml]$report = Get-Content .\report.xml -Raw
$report.SelectNodes("//*[local-name()='record']").Count
$report.SelectSingleNode("//*[local-name()='org_name']").InnerText</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">xmllint --noout report.xml
xmllint --xpath 'count(//*[local-name()="record"])' report.xml
xmllint --xpath 'string(//*[local-name()="org_name"])' report.xml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) laster her en allerede utpakket lokal XML-fil; [`xmllint`](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) kontrollerer struktur og XPath-spørringer. Ukjente rapporter bør ikke åpnes interaktivt med privilegerte skrivebordsverktøy. Denne enkeltkontrollen beviser verken skjemasamsvar eller sikker massebehandling, duplikatgjenkjenning eller korrekt aggregering.

### Knytt feil systematisk til en grense

| Observasjon | Sannsynlig årsak | Neste pålitelige bevis |
|---|---|---|
| `spf=none` | feil identitet eller ingen SPF-post | MailFrom/HELO fra SMTP- eller Auth-Result og TXT på det eksakte navnet |
| `spf=permerror` | syntaks, flere poster, rekursjon eller DNS-budsjett | evaluer fullt rekursivt SPF-tre og oppslags-/Void-tellere |
| SPF består, DMARC feiler | SPF-domene ikke aligned | sammenlign RFC5321.MailFrom med RFC5322.From i konfigurert alignment-modus |
| `dkim=fail (body hash did not verify)` | brødtekst endret etter signering | sammenlign signeringshopp, MIME-normalisering, footer/disclaimer og kanonisering |
| `dkim=temperror` | nøkkeloppslag mislyktes midlertidig | selektornavn, resolversvar, timeout og autoritativ DNS-tilgjengelighet |
| DKIM består, DMARC feiler | bare ikke-aligned signatur består | kontroller alle `d=`-domener individuelt mot Author Domain |
| Mottakere rapporterer ulike DMARC-resultater | ulike veier, DNS-cacher, verifikatormodeller eller mutasjoner | korreler samme Message-ID/signatur med én fullstendig header og DNS-tidspunkt per tilfelle |
| ukjent kilde-IP i Aggregate Reports | ny legitim avsender, videresender eller misbruk | fastslå volum, headereksempel, reverse-/leverandørtilordning og intern tjenesteeier |
| `Authentication-Results` motsier hverandre | flere kontrollhopp eller forfalsket eksternt felt | bruk bare resultater innenfor den definerte tillitsgrensen |
| `p=reject`, men meldingen leveres | lokal overstyring hos mottakeren | kontroller `disposition`, overstyringsgrunn og ytterligere filterresultater |

## Teknisk historie

SPF oppsto fra flere forslag til SMTP-avsenderautorisering og ble publisert som eksperimentell RFC 4408 i 2006. RFC 7208 flyttet SPF til Standards Track i 2014, fjernet den separate DNS-RR-typen SPF og presiserte blant annet DNS- og Void-grenser ([RFC 4408](https://datatracker.ietf.org/doc/html/rfc4408), [RFC 7208, vedlegg B](https://datatracker.ietf.org/doc/html/rfc7208#appendix-B)). Arkitekturen forble bevisst orientert mot SMTP-forbindelsen og Reverse Path.

DKIM samlet erfaringer fra DomainKeys og Identified Internet Mail. RFC 4871 standardiserte DKIM i 2007; RFC 6376 erstattet den i 2011 og skjerpet signatur-, nøkkel- og verifikasjonsmodellen. RFC 8301 oppdaterte algoritmer og RSA-nøkkellengder i 2018, RFC 8463 la til Ed25519-SHA256 ([RFC 4871](https://datatracker.ietf.org/doc/html/rfc4871), [RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376), [RFC 8301](https://datatracker.ietf.org/doc/html/rfc8301), [RFC 8463](https://datatracker.ietf.org/doc/html/rfc8463)).

DMARC ble først publisert som et informativt dokument med RFC 7489 i 2015. RFC 9989 erstattet RFC 7489 og PSD-utvidelsen RFC 9091 som Standards Track-spesifikasjon i 2026; Aggregate Reporting og Failure Reporting ble samtidig flyttet ut til RFC 9990 og RFC 9991. Overgangen innførte blant annet DNS Tree Walk, `np`, `psd` og `t`, og fjernet `pct` ([RFC 9989, vedlegg C](https://datatracker.ietf.org/doc/html/rfc9989#appendix-C), [RFC 9990](https://datatracker.ietf.org/doc/html/rfc9990), [RFC 9991](https://datatracker.ietf.org/doc/html/rfc9991)).

Den maskinlesbare overføringen av kontrollresultater utviklet seg fra RFC 5451 via RFC 7001 og RFC 7601 til RFC 8601. ARC ble publisert i 2019 med RFC 8617 som et eksperimentelt forsøk på å videreformidle autentiseringsresultater fra indirekte flyter signert ([RFC 8601, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc8601#section-6), [RFC 8617](https://datatracker.ietf.org/doc/html/rfc8617)). Denne historien forklarer hvorfor reelle plattformer side om side kan vise gamle DMARC-tagger, PSL-baserte Organizational Domains, ulike DKIM-algoritmer og forskjellige tillitsmodeller for Auth-Results.

## Kilder

- [RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208) – SPF-identiteter, postevaluering, resultater og DNS-grenser.
- [RFC 6376 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc6376) – DKIM-signatur-, nøkkel- og verifikasjonsmodell.
- [RFC 9989 – Domain-Based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc9989) – DMARC-kjerneprotokoll, alignment, policy, DNS Tree Walk og drift.
- [RFC 5321, avsnitt 3.3 og 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 5322, avsnitt 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322)
- [RFC 8301 – DKIM Cryptographic Algorithm and Key Usage Update](https://datatracker.ietf.org/doc/html/rfc8301) – SHA-256 og RSA-nøkkellengder.
- [RFC 8463 – Ed25519-SHA256 for DKIM](https://datatracker.ietf.org/doc/html/rfc8463) – ekstra signatur- og nøkkelalgoritme.
- [RFC 9990 – DMARC Aggregate Reporting](https://datatracker.ietf.org/doc/html/rfc9990) – XML-datamodell, transport, duplikater og eksterne rapportmål.
- [RFC 9991 – DMARC Failure Reporting](https://datatracker.ietf.org/doc/html/rfc9991) – per-message Failure Reports og personvern.
- [RFC 8601 – Authentication-Results](https://datatracker.ietf.org/doc/html/rfc8601) – headerformat, `authserv-id` og tillitsgrense.
- [RFC 8617 – Authenticated Received Chain](https://datatracker.ietf.org/doc/html/rfc8617) – eksperimentell ARC-kjede for indirekte flyter.
- [RFC 7960, avsnitt 3 og 4](https://datatracker.ietf.org/doc/html/rfc7960)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-oppslag i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS-oppslag i Unix-systemer.
- [Microsoft Learn – Get-Content](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) – analyse av lokale filer og headere i Windows.
- [Microsoft Learn – Select-String](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) – mønsteranalyse i PowerShell.
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) – tekstsøk i Linux og Unix.
- [libxml2 – xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) – XML- og XPath-kontroll.
- [RFC 4408 – Sender Policy Framework, Experimental](https://datatracker.ietf.org/doc/html/rfc4408) – forgjenger til RFC 7208.
- [RFC 4871 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc4871) – forgjenger til RFC 6376.
