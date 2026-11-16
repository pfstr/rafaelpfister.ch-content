---
title: "Migrering: målarkitektur, sameksistens og kontrollert cutover"
blatt: "migration"
description: "Teknisk migrering for meldings- og infrastrukturadministratorer: kilde- og målarkitektur, avhengigheter, identitets- og datakartlegging, rehost til refactor, sameksistens, seed- og deltasykronisering, pilot, cutover, ansvarsgrenser, rollback, godkjenning og avvikling."
fakten:
  - label: Utgangspunkt
    wert: forretningsmessig transaksjon + kildearkitektur + tilstander + eiere
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Mål
    wert: målbar målarkitektur med grensesnitt, datastyring, tillits- og feilgrenser
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Strategier
    wert: Retain · Retire · Rehost/Relocate · Repurchase · Replatform · Refactor
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Mønstre
    wert: Cutover · Pilot/bølger · sameksistens · gradvis utskifting
    href: https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html
  - label: Identitet
    wert: kartlegg kilde-ID, mål-ID, pålogging, SMTP-adresser, grupper, roller og livssyklus separat
    href: https://datatracker.ietf.org/doc/html/rfc7643
  - label: Datastiforløp
    wert: Seed → deltasykronisering → Freeze/Final Delta → avstemming
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration
  - label: Sameksistens
    wert: adressebok · e-postruting · Discovery · Auth · tillatelser · kalender · support
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: E-postansvar
    wert: 250 etter End-of-Data betyr aksept fra det svarende SMTP-hoppet
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: DNS-cutover
    wert: TTL begrenser cachelevetid; negative cacher og lokale overstyringer må kontrolleres separat
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Rollbackgrense
    wert: fastsett eksplisitt før den første eksklusive skriveoperasjonen i målet
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Godkjenning
    wert: fullstendighet · integritet · tillatelse · transaksjon · ytelse · overvåking
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Avvikling
    wert: behandle ruting, identiteter, nøkler, sikkerhetskopier, logger, avtaler og datamedier kontrollert
    href: https://csrc.nist.gov/pubs/sp/800/88/r1/final
werbung:
  - stargate
  - newsletter
ctaThemen:
  - migration
  - messaging
  - exchange-hybrid
translationSourceHash: bdb2fea86e1f811e78f2f210afd26b082f808325322fb675a8ec446e47b38ac3
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:23:04.204Z
translationReview: automatic
---

# Migrering: målarkitektur, sameksistens og kontrollert cutover

En teknisk migrering overfører forretningsprosesser, grensesnitt og tilstander fra en kildearkitektur til en definert målarkitektur. Det synlige produktbyttet er bare én del av dette. Identiteter, adresser, tillatelser, køer, data, nøkler, DNS, sertifikater, klienter, automatiseringer, overvåking, sikkerhetskopiering og supportprosesser må fortsatt oppfylle de samme faglige løftene – eller bevisst overføres til en ny avtale.

For meldingssystemer er det viktigste løftet kontrollert ansvar for hver melding. Under en migrering kan gamle og nye plattformer samtidig kjenne adresser, akseptere forbindelser og endre data. Uten eksplisitt dataeierskap oppstår løkker, Split-Brain-adressebøker, dobbelt levering, tapte tillatelser og en returvei som bare finnes i planleggingen. En robust migrering begynner derfor med arkitektur og tilstander, ikke med omkoblingsdatoen.

Forklaringen starter med dokumentert nåtilstand og går via målbilde, avhengigheter og dataoverføring til sameksistens. Pilot, cutover, returvei og godkjenning følger først når det er klart hvilke data og funksjoner som flyttes.

En migrering er ikke en kopieringsprosess, men det kontrollerte byttet fra en fungerende kildearkitektur til en verifisert målarkitektur. Først registreres tilstander og avhengigheter, deretter planlegges sameksistens, dataflytting, cutover og returvei.

## Fra kilde- til målarkitektur

Før produktvalget dokumenteres den eksisterende tjenesten som en graf. Hver node har en rolle; hver kant har initiator, mål, protokoll, identitet, dataklasse, feilatferd og eier. En CMDB-liste over servernavn er ikke tilstrekkelig dersom det er ukjent hvilken skanner som kaller et SMTP-hopp, hvilken tjeneste som utvider grupper eller hvilken nøkkel som dekrypterer gamle meldinger.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-migration.svg?v=20260813" title="Interaktive Infografik: Migration von Ausgangs- zu Zielarchitektur mit Strategie, Zustandsklassen, Koexistenz, Seed und Delta, Pilotwellen, Cutover, Rollbackgrenze, Abnahme und Stilllegung" loading="lazy">
  <a href="/images/kb-interaktiv-migration.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

Kildedokumentasjonen omfatter minst:

| Domene | Nåtilstand som skal registreres | Typisk skjult avtale |
|---|---|---|
| E-postruting | MX, relay-/submission-lyttere, koblinger, smarthoster, Accepted Domains, kilde-IP, TLS-policy | Partner tillater en gammel IP eller forventer et sertifikatnavn |
| Identitet | katalogkilde, objektanker, pålogging, grupper, roller, tjenestekontoer, klargjøring | Applikasjonen bruker e-postadresse som uforanderlig ID |
| Data | postkasse, arkiver, journaler, kø, karantene, blob, indeks, konfigurasjon, revisjon | Søkeindeksen kan rekonstrueres, mens journalen må oppbevares |
| Kryptografi | TLS-sertifikater, S/MIME-/OpenPGP-nøkler, HSM/KMS, passfraser, gjenoppretting | gammel kryptert melding trenger en historisk privat nøkkel |
| Klienter og API-er | Outlook/IMAP/SMTP/API, agenter, mobil, webhooks, skript | hardkodet URL, chiffer, port, header eller API-felt |
| Drift | overvåking, sikkerhetskopiering, gjenoppretting, patching, logging, support, lisenser | Alarm kontrollerer bare prosessstatusen til det gamle systemet |
| Organisasjon | eier, vaktordning, godkjenning, personvern, oppbevaring, avtale | eksternt team kontrollerer DNS eller brannmur med lang ledetid |

Målarkitekturen beskriver de samme feltene som **målavtale**. I tillegg angis tillitsgrenser, dataeierskap, feilområder, gjenopprettingsmål, skalering og planlagt sluttilstand for det gamle systemet. NIST SP 800-34 knytter systemutvikling til beredskapsplanlegging og krever at gjenopprettingsstrategier og tester utledes av forretningskrav. For migreringer følger det av dette: Målarkitektur og tilbakefallsdrift utformes sammen ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)).

## Migreringsstrategi: Hva endres faktisk?

De «7 R-ene» dokumentert av AWS skiller mellom Retire, Retain, Rehost, Relocate, Repurchase, Replatform og Refactor/Re-architect. De er en taksonomi for skymigrering, men kan brukes som et presist spørsmål ved enhver plattformmigrering: Består funksjonen, avvikles den, flyttes den uendret, erstattes den av et produkt, tilpasses den en ny plattform eller bygges den faglig på nytt? AWS påpeker at refaktorering i store migreringer er den mest komplekse strategien, og at separat modernisering kan redusere risiko ([AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)).

| Strategi | Hva forblir likt? | Hva endres? | Eksempel for meldinger |
|---|---|---|---|
| Retain | Plattform og ansvar | bare dokumentert unntak/frister | Legacy-arkivet forblir lesbart til oppbevaringstiden utløper |
| Retire | ingenting produktivt | funksjon og tilganger fjernes | ubrukt POP3-endepunkt slås av |
| Rehost / Relocate | applikasjon og datamodell i stor grad | compute-/virtualiseringssted | MTA-VM flyttes til annen infrastruktur |
| Repurchase | forretningskrav | produkt, datamodell, drift og avtale | appliance-gateway erstattes av SaaS-gateway |
| Replatform | fagfunksjon og deler av arkitekturen | runtime, database eller driftsmodell | e-posttjeneste bruker ny database-/containerplattform |
| Refactor | bare forretningsintensjon | grensesnitt, komponenter og tilstander | monolittisk e-postpipeline bygges på nytt hendelsesorientert |

Strategien velges **per komponent og dataklasse**. Et prosjekt kan replatforme e-postruting, beholde arkivdata, avvikle et rapporteringsverktøy og repurchase brukergrensesnittet. Den som kaller alt «Lift and Shift», skjuler de faktisk ulike risikoene.

## Cutover, bølger, sameksistens og gradvis utskifting

Fire omstillingsmønstre er teknisk forskjellige og må utledes av dokumenterte produktegenskaper:

- **Engangs-cutover:** Alle skrivebaner byttes i ett vindu. Egnet når sameksistens ikke er teknisk mulig, tilstanden er liten og avbruddet er akseptabelt. Returveien må være klar før den første målskriveoperasjonen.
- **Pilot og bølger:** En representativ populasjon bytter først; flere batcher følger etter godkjenning. Microsofts migreringsbatcher er et konkret eksempel på trinnvis overføring av postkasser ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).
- **Sameksistens:** Gammelt og nytt deler midlertidig navneområde, adressebok, ruting eller brukerpopulasjon. Exchange Hybrid dokumenterer blant annet sikker e-postflyt, felles domenenavnerom, adressebok og kalenderdeling som sameksistensfunksjoner ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).
- **Gradvis utskifting:** Et rute- eller integrasjonslag fanger opp funksjoner og flytter dem gradvis til målet. Martin Fowler beskrev dette mønsteret i 2004 som «Strangler Fig Application» for å redusere risikoen ved én enkelt stor rewrite-cutover ([Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Ingen av disse mønstrene er generelt sikre. Sameksistens reduserer cutover-omfanget, men skaper flere tilstander, rutingregler og supporttilfeller. Dual Write kan øke tilgjengeligheten, men skaper problemer med konfliktløsning og rekkefølge. En engangs-cutover er enklere å forstå, men øker den samtidige skaderadiusen. Beslutningen angir derfor produktbelegg, tillatt varighet, dataeierskap og exitkriterier.

Etter at strategien er valgt, må det for hver dataklasse være fastlagt hvilket system som kan skrive i hver fase. To ukoordinerte sannhetskilder skaper konflikter som et kopieringsverktøy ikke kan løse.

## Tilstandsklasser og dataeierskap

Før hver synkronisering fastsettes hvilke data som er **autoritativ kilde**, **replisert kopi**, **avledet projeksjon** eller **kun historisk arkiv**:

| Tilstandsklasse | Autoritet under sameksistens | Konflikt-/gjenopprettingsspørsmål |
|---|---|---|
| Identitetsgrunndata | nøyaktig én ledende katalog eller HR-system | hvilken objektanker overlever navneendring og tenantbytte? |
| SMTP-adresser og rutingobjekter | én godkjent adressekilde | hvordan hindres duplikater og Catch-all-/aliasregler? |
| Postkasseinnhold | entydig kilde eller mål per migrerte postkasse | kan det fortsatt skrives i kildepostkassen etter cutover? |
| Kø | den aktuelle aksepterende MTA-en har ansvaret | hvordan tømmes gamle køer uten dobbel reinjisering? |
| Policy og konfigurasjon | deklarativ måltilstand eller angitt master | hvilke regler er semantisk ekvivalente, ikke bare syntaktisk kopiert? |
| Nøkler og sertifikater | KMS/HSM eller definert nøkkelansvarlig | hvilke historiske private nøkler må fortsatt være lesbare? |
| Revisjon, journal og oppbevaring | uforanderlig oppbevaringskilde | er tid, fullstendighet, Legal Hold og eksport dokumenterbare? |
| Indeks/cache | rekonstruerbar projeksjon | hvor lang tid tar gjenoppbygging, og når anses søket som fullstendig? |

«Toveis synkronisert» er ikke en datamodell uten konfliktregel. For hvert felt dokumenteres retning, frekvens, transformasjon, feilplassering og atferd ved samtidig endring. Dersom verktøyet ikke kan slå sammen endringer uten tap, gjøres én side skrivebeskyttet i den aktuelle fasen eller endres kun gjennom en sentral skrivebane.

## Identitet, adresser og tillatelser

Brukernavn, e-postadresse og teknisk objekt-ID er forskjellige ting. RFC 7643 definerer blant annet `id` for SCIM som en uforanderlig identifikator tildelt av tjenesteleverandøren og entydig innenfor tenanten; `externalId` kan bære en korrelasjon administrert av klienten. Dette illustrerer en sentral migreringsregel: En endringsbar adresse er ikke en sikker objektanker ([RFC 7643](https://datatracker.ietf.org/doc/html/rfc7643)).

En identitetskartlegging inneholder minst kilde-ID, mål-ID, `externalId` eller en annen korrelasjonsanker, pålogging/UPN, primær SMTP-adresse, aliaser, grupper, roller, leder-/organisasjonsattributter, tjenestekontoer, autentiseringsmetode og klargjøringsstatus. RFC 7644 beskriver SCIM-operasjoner for oppretting, endring, søk og bulkbehandling; den konkrete målplattformen kan likevel ha avvikende obligatoriske felt, entydighetsregler og rate limits ([RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644)).

Tillatelser migreres som relasjoner, ikke som tekstlister. Delegasjon, Send As, Send on Behalf, Folder ACL, Shared Mailbox, ressursbestilling, Application Access og administratorrolle kan referere til ulike objekttyper. Hver relasjon trenger en oppløst målanker; foreldreløse eller sykliske referanser identifiseres før bølgen. Microsoft dokumenterer for eksempel at migrering av postkasser på tvers av tenanter forutsetter mål-`MailUser`-objekter og bestemte attributter og bare overfører definert innhold/tillatelser ([Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)).

## Databane: Seed, Delta, Freeze og avstemming

Store tilstander forhåndslastes vanligvis før cutover. Den robuste prosessen er:

1. **Seed:** komplett utgangsbestand med objekttelling, bytevolum, feilliste og tidsområde.
2. **Deltasykronisering:** overfør gjentatte ganger endringer siden seed; observer forsinkelse og feilrate.
3. **Freeze eller skriveomdirigering:** stans nøyaktig definerte kildeskrivebaner eller led dem til målet.
4. **Final Delta:** hent inn gjenværende endringer og sikre et reproduserbart vannmerke.
5. **Cutover:** bytt ruting, oppdagelse og brukertilgang til målet.
6. **Avstemming:** sammenlign antall, størrelse, nøkkelattributter, tillatelser, stikkprøver og faglige transaksjoner.

Microsofts trinnvise e-postmigrering dokumenterer batcher, synkroniseringsstatus og kravet om å vente på minst én ytterligere synkronisering etter ruteomleggingen før en batch slettes. Dette er produktspesifikt belegg for seed-/delta-/fullføringsfaser, ikke en universell kommando for andre systemer ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).

Sjekksummer dokumenterer byteidentitet bare ved identisk serialisering. Eksport/import kan normalisere meldinger, konvertere metadata eller strukturere containere annerledes. Godkjenningen kombinerer derfor tekniske summer med faglige invarianter: objektantall per klasse, eldste/nyeste tidsstempel, total størrelse, manglende/større objekter, tillatelsesrelasjoner, tilfeldig stikkprøve og målrettet vanskelige tilfeller som store MIME-meldinger, kalendergjentakelser, Unicode, kryptering og Legal Hold.

Når identitet og dataeierskap er definert, kan løpende meldingstrafikk føres mellom gammelt og målsystemet. Sameksistens er da en midlertidig produksjonsarkitektur med egne rute- og feiltilfeller.

## Sameksistensarkitektur for meldinger

Et felles SMTP-domene på to plattformer krever en entydig beslutning om mottaker og ruting. Microsoft beskriver for hybriddrift et felles domenenavnerom, sikker e-postflyt, felles adressebok og kalenderfunksjoner; for sameksistens med tredjepart fremheves beslutninger om identitet, synkronisering og autentisering uttrykkelig ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid), [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)).

Den lokale sameksistensmatrisen besvarer:

- Hvilken plattform kjenner hver mottaker og med hvilken type?
- Hvem har ansvar for ukjente mottakere, og hvordan forhindres løkker?
- Hvilket hopp signerer DKIM, bruker DLP/arkiv/disclaimer eller krypterer?
- Hvordan gjenkjenner gammelt og nytt hverandre som internt, autentisert eller eksternt hopp?
- Hvor ligger Free/Busy, delegasjon, gruppeutvidelse og Autodiscover/Service Discovery?
- Hvilken plattform skriver hvilke identitets- og adresseattributter?
- Hvordan korreleres bouncer, karantene, journaler og Message Trace på begge sider?
- Hvilket supportpunkt har ende-til-ende-oversikt?

Hver overgang gjøres observerbar med [SMTP](/kb/smtp)-svar, kø-ID og Trace-/Message-ID. RFC 5321 fastsetter at den mottakende serveren overtar ansvaret etter vellykket avslutning av DATA. En rollback etter et `250`-svar må ikke glemme denne aksepterte meldingen eller reinjisere den ukontrollert ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

## DNS, sertifikater og eksterne avtaler

DNS er en distribuert utvalgsmekanisme, ikke en global bryter. RFC 1034 beskriver caching som en grunnleggende del og overlater til datakilden å avveie TTL mellom aktualitet og innsats. Før en omstilling reduseres relevante TTL-er derfor i tide; etter cutover kontrolleres gamle baner minst så lenge resolver, negative cacher, lokale hosts-filer, proxyer og eksterne gatewayer kan bruke gamle data ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).

MX, A/AAAA, CNAME, SRV og TXT samt interne/eksterne views må kontrolleres. SPF må autorisere nye sendende IP-er uten å fjerne gamle kilder for tidlig; DKIM-selector og DMARC-justering valideres med ekte meldinger. Sertifikater trenger navn, kjede, trust, privat nøkkel, Extended Key Usage og eksport-/HSM-funksjonalitet. Historiske S/MIME-/OpenPGP-dekrypteringsnøkler følger sin egen livssyklus. NIST SP 800-57 omhandler nøkkeltilstander, beskyttelse og arkivering; et plattformbytte opphever ikke disse kravene ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Eksterne avtaler omfatter partner-allowlisting, Smarthost-credentials, API-redirect-URI-er, webhooks, SIEM-kilder, backupagenter, overvåkingsprober, brannmurer, NAT, Reverse DNS og supporttilganger. Hver avtale har en teknisk eier og en test før cutover.

## Pilot, bølger og beslutningsporter

En pilot er representativ, ikke bare velvillig utvalgt. Den omfatter store og små postkasser, delegeringer, Shared/Resource Mailboxes, mobile og eldre støttede klienter, eksterne partnere, kryptering, arkiver, Power Users, ulike steder og minst ett kontrollert negativt tilfelle. Pilotpopulasjonen må samtidig ikke inneholde en irreversibel særfunksjon.

Hver bølge har porter:

- **Ready:** Kartlegging komplett, målobjekter finnes, seed innenfor lag-målet, klient/nettverk/policy er klart, Restore og support er bemannet.
- **Go:** Siste delta- og integritetskontroll er vellykket, ingen åpen blokkeringsklasse, avhengigheter og tidsvindu er bekreftet.
- **Hold:** Feilrate, lag, kø, replikering eller supportbelastning utenfor terskelen; ingen flere objekter startes.
- **Abort:** Dataintegritet, sikkerhet, ukontrollerbart meldingstap eller andre forhåndsdefinerte røde kriterier.
- **Accept:** Faglig godkjenning og observasjonsperiode er oppfylt; objektet tilhører nå entydig målet.

Bølger deles slik at en feil forblir begrenset: for eksempel etter lokasjon, tenant, domene, database, klienttype, funksjonsgruppe eller avhengighet. Tilfeldige mengder kan være representative, men er operativt vanskelige å reversere. Hver bølge får en uforanderlig objektliste og et resultatdatasett.

Pilot og bølger leverer bevisene for den egentlige cutoveren. Returen forblir bare realistisk så lenge tilstandsendringer fortsatt kan føres fullstendig tilbake til det gamle systemet.

## Cutover og rollbackgrense

Cutover-runbooken inneholder ikke bare trinn, men **forutsetninger, eier, forventet funn, maksimal varighet, avbruddskriterium og returvei**. En mulig rekkefølge er:

1. Frys endringer, åpne incident-/kommunikasjonskanal og sikre utgangsmålinger.
2. Kontroller Final Delta og kartleggingsfeil; sperr kildeskrivebaner i samsvar med modellen.
3. Test mållistenere, sertifikater, identitet, policy og nedstrømsavhengigheter på nytt.
4. Bytt interne rute-/discoverykontroller, deretter eksterne DNS-/partnerbaner.
5. Utfør kontrollerte transaksjoner for inbound, outbound, submission, access, policy og kryptering.
6. Overvåk kø, SMTP-koder, autentisering, replikering, indeks, klientfeil og overvåking.
7. Beslutt Go/Hold/Abort og dokumenter tidspunktet.

**Rollback Point** ligger før tidspunktet da målet overtar det eneste, ikke-tapsfritt reverserbare skriveansvaret. Etter dette er blind omkobling til kilden ikke en rollback, men en andre migrering. Nye målmeldinger, kalenderendringer, tillatelser, nøkler og køtilstander må da tilbakesynkroniseres, eksporteres eller bevares gjennom roll-forward. [Modellen for sikkerhetskopiering og DR](/kb/backup-dr) definerer konsistenspunkt, RPO/RTO og gjenopprettingsrekkefølge.

## Administratorverktøy for før-/etterbevis

Eksemplene kontrollerer enkeltavtaler. De kjøres fra den faktisk berørte opprinnelsen og med godkjente testdata. Påloggingsdata, personopplysninger og private nøkler skal ikke ligge ubeskyttet i saker eller shell-historikk.

### Sammenlign DNS før og etter cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName autodiscover.example.ch -Type CNAME -DnsOnly -Server 192.0.2.53
Resolve-DnsName example.ch -Type TXT -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer example.ch MX
dig @192.0.2.53 +noall +answer autodiscover.example.ch CNAME
dig @192.0.2.53 +noall +answer example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) gjør resolver og posttype eksplisitte. Interne og eksterne resolvere kontrolleres separat; et riktig svar fra en offentlig resolver beviser ikke at produksjonstjenesten bruker samme view.

### Nå nye endepunkter fra den reelle opprinnelsen

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection new-mx.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection new-api.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 new-mx.example.ch 25
nc -vz -w 5 new-api.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) dokumenterer bare DNS-/TCP-tilgjengelighet fra den valgte opprinnelsen. Brannmur, NAT, kilde-IP, TLS, autentisering og fagprotokoll kontrolleres deretter separat.

### Kontroller TLS-identitet og protokollsvar

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) og [`curl`](https://curl.se/docs/manpage.html) kontrollerer TLS og en kontrollert dialog. Begge bruker sin egen trust-/proxy-/buildkonfigurasjon; produksjonsklienten må i tillegg testes med sin faktiske policy.

### Eksporter og sammenlign identitetskartlegging

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Identitätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -Filter * -Properties ObjectGUID,UserPrincipalName,mail,proxyAddresses,Enabled |
  Select-Object ObjectGUID,SamAccountName,UserPrincipalName,mail,proxyAddresses,Enabled |
  Export-Csv .\\identity-source.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'dc=example,dc=ch' '(objectClass=person)' \
  entryUUID uid userPrincipalName mail proxyAddresses</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) og [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) leverer identitetsattributter fra de respektive katalogene. Eksport og spørring må ta hensyn til scope, paging, slettede/deaktiverte objekter, flerverdiattributter og tilgangsrettigheter. Målavstemmingen bruker den avtalte objektankeren, ikke bare visningsnavnet.

### Verifiser eksportartefakter bytepresist

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Hashprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\export\\mailbox-001.pst -Algorithm SHA256
Get-FileHash .\\export\\mapping.csv -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./export/mailbox-001.mbox
sha256sum ./export/mapping.csv</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) oppdager endringer i identiske filer. De dokumenterer ikke at kilde- og målpostkassen er semantisk like; dette krever objekttelling, attributter, feillister og faglige stikkprøver.

### Sikre logger i cutover-tidsvinduet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T20:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 20:00:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) og [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leverer tidsrefererte systemhendelser. I tillegg kommer produkt-, migreringsbatch-, SMTP-, katalog-, database- og API-logger med kø-/objekt-/trace-ID. Flyktige bevis sikres før omstart eller rotasjon.

## Godkjenning og etterløp

Godkjenningen kontrollerer ikke bare «brukeren kan logge inn», men seks dimensjoner:

1. **Fullstendighet:** forventede objekter, tidsområder, mapper, attributter, regler, nøkler og tillatelser finnes.
2. **Integritet:** ingen uakseptabel endring, duplikat, tapt relasjon eller inkonsistent dataeierskap.
3. **Tillatelse:** brukere, tjenester og administratorer kan utføre nøyaktig de planlagte operasjonene – inkludert negative tester.
4. **Transaksjon:** inbound/outbound, submission/access, ruting, policy, kryptering, karantene, arkiv og API fungerer ende til ende.
5. **Drift:** ytelse, kø, replikering, indeks, sikkerhetskopiering, gjenoppretting, patching, support og kapasitet oppfyller måltallene.
6. **Observability:** logger, målinger, traces, Message Tracking, varsling og runbooks ser målbanen.

Etterløpsfasen varer gjennom relevante TTL-, cache-, retry-, delta-, retention- og forretningsbelastningssykluser. Gamle plattformer forblir bare aktive i den forhåndsdefinerte rollen: for eksempel å tømme køen eller lese historisk, ikke umerket fortsette å akseptere nye data. Midlertidige forwardere og sameksistensdomener får en utløpsdato og overvåking.

Først etter faglig godkjenning og etterløp begynner avviklingen. Logger, nøkler, DNS-oppføringer, tjenestekontoer og skjulte integrasjoner må fjernes eller arkiveres kontrollert.

## Avvikling uten tapte avhengigheter

Avvikling er en egen kontrollert tilstandsendring. Før avslåing kontrolleres siste tilganger, gjenværende køer, aktive økter, jobber, DNS-/brannmurtreff, tjenestekontoer, sikkerhetskopier, gjenopprettingsplikter, Legal Holds, lisenser og supportavtaler. Deretter fjernes ruting og discovery, tilganger sperres, secrets og sertifikater tilbakekalles eller arkiveres, overvåking tilpasses og CMDB oppdateres.

Datamedier og skyressurser slettes eller gjøres kryptografisk ubrukelige i henhold til dataklasse og avtale. NIST SP 800-88 beskriver Clear, Purge og Destroy som sanitiseringsmetoder og krever en beslutning tilpasset medier og risiko ([NIST SP 800-88 Rev. 1](https://csrc.nist.gov/pubs/sp/800/88/r1/final)). Sikkerhetskopier og nøkler kan leve lenger enn produktinstansen; deres oppbevaring, lesbarhet og endelige destruksjon forblir dokumentert.

## Teknisk utvikling av migreringer

Tidlige infrastrukturmigreringer var ofte en engangsutskifting av maskinvare eller programvare med et stort omkoblingsvindu. Virtualisering og imagekopier forenklet Rehost og Relocate; standardiserte katalog-, API- og klargjøringsprotokoller muliggjorde mer finmasket sameksistens. Sky- og SaaS-plattformer gjorde tenant-, identitets- og avtalegrenser til like viktige migreringsobjekter som server og database. 7-R-taksonomien beskriver denne ulike inngripensdybden; Strangler-Fig-mønsteret beskriver gradvis funksjonsutskifting ([AWS – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html), [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Den tekniske utviklingen fjerner ikke cutoveren. Den flytter den til mindre enheter: enkeltidentiteter, domener, API-er, dataklasser eller rutebeslutninger. Dermed reduseres den samtidige skaderadiusen, mens sameksistens og dataeierskap blir mer krevende. Moderne migreringsarkitektur optimaliserer derfor ikke bare kopieringshastighet, men **tydelighet i ansvar, observerbarhet for hver overgang og en reell tilbake- eller fremovervei**.

## Kilder

- [NIST – SP 800-34 Rev. 1, beredskapsplanlegging](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)
- [Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)
- [Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)
- [IETF RFC 7643 – SCIM Core Schema](https://datatracker.ietf.org/doc/html/rfc7643)
- [IETF RFC 7644 – SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
- [Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)
- [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [NIST – SP 800-57 Part 1 Rev. 5, nøkkelhåndtering](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – kommandolinjemanual](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2-verktøy](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [NIST – SP 800-88 Rev. 1, mediesanitering](https://csrc.nist.gov/pubs/sp/800/88/r1/final)
