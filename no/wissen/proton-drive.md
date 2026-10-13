---
title: "Proton Drive: Kryptografisk datamodell og synkroniseringsdrift"
blatt: "proton-drive"
description: "Proton Drive for lagrings-, sikkerhets- og plattformadministratorer: sentral skylagring med klientside ende-til-ende-kryptering, volumer, delinger, noder, revisjoner og blokker, OpenPGP-nøkkelhierarki og signaturer, Drive SDK, skrivebordssynkronisering og filer på forespørsel, CLI-automatisering, deling, konflikter, versjoner, logging, gjenoppretting og teknisk historie."
fakten:
  - label: Systemrolle
    wert: sentralt drevet skylagring for filer med klientside ende-til-ende-kryptering, synkronisering, deling og versjonering
    href: https://proton.me/support/drive
  - label: Tillitsmodell
    wert: Serveren lagrer kryptert innhold og navn; klartekst og opplåst nøkkelmateriale oppstår i den autoriserte klienten
    href: https://proton.me/blog/protondrive-security
  - label: Datamodell
    wert: Volume inneholder Shares; Share peker på en trerot; filer og mapper er Nodes med Links, nøkler og revisjoner
    href: https://proton.me/blog/protondrive-security
  - label: Filoverføring
    wert: Revisjoner består av krypterte blokker; klienten krypterer, overfører, kontrollerer og setter sammen objektet
    href: https://proton.me/blog/drive-cryptography-update
  - label: Kryptografi
    wert: OpenPGP-basert nøkkelhierarki; nye filer bruker v6 PKESK, v2 SEIPD og AES-GCM, mens eldre revisjoner forblir skjemabundet
    href: https://proton.me/blog/drive-cryptography-update
  - label: Metadata
    wert: Innhold samt fil- og mappenavn er E2EE; enkelte felt, som filstørrelse, forblir ifølge Proton lesbare for serveren
    href: https://proton.me/support/drive
  - label: Integritet
    wert: signert nøkkel-, tre- og blokkinformasjon samt OpenPGP-signaturer oppdager manipulering
    href: https://proton.me/support/drive-signature-management
  - label: Klientkjerne
    wert: Drive SDK standardiserer filoperasjoner og kryptografi på tvers av web-, skrivebords-, mobil- og CLI-klienter
    href: https://github.com/ProtonDriveApps/sdk
  - label: Skrivebordssynkronisering
    wert: Windows og macOS integrerer plassholdere på forespørsel, lokale frakoblet-tilstander og bakgrunnssynkronisering i henholdsvis Explorer og Finder
    href: https://proton.me/support/proton-drive-windows-on-demand-sync
  - label: CLI
    wert: offisielt enkeltprogram for Windows, macOS og Linux; fil- og delingsoperasjoner, men ingen permanent synkroniseringsmotor for skrivebord
    href: https://proton.me/support/drive-cli
  - label: Gjenopprettingskjerne
    wert: Kontotilgang alene er ikke nok etter tilbakestilling av passord; gammel nøkkel låses opp igjen med tidligere passord, Recovery Phrase eller Recovery File
    href: https://proton.me/support/drive-data-recovery
  - label: Regel for sikkerhetskopiering
    wert: Synkronisering og versjonshistorikk er ingen uavhengig sikkerhetskopi; kritiske data krever separat eksport, kontrollsummer, oppbevaring og gjenopprettingstest
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
werbung:
  - newsletter
ctaThemen:
  - proton-drive
translationSourceHash: 6063c7d85190de63a8973bc7cdb304fff10f81fd617a2044cf40f8371849717c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:51:11.032Z
translationReview: automatic
---

# Proton Drive: Kryptografisk datamodell og synkroniseringsdrift

Proton Drive er en sentralt drevet skyfillagring der klientene krypterer innhold og vesentlige metadata før opplasting. Tjenesten kombinerer nettilgang, skrivebordssynkronisering, mobilapper, delinger, versjoner og en kommandolinje med en ende-til-ende-kryptert datamodell. [Supportoversikten](https://proton.me/support/drive) nevner tilgang via web, Windows, macOS, Android og iOS/iPadOS; den offisielle [Drive CLI](https://proton.me/support/drive-cli) supplerer med skriptbare operasjoner på Windows, macOS og Linux.

For administratorer er Proton Drive verken en klassisk fildeling eller bare en opplastingsportal. En autorisert klient forvalter kontoøkt og nøkler, løser opp et kryptert tre, overvåker lokale endringer, oppretter revisjoner, krypterer blokker og synkroniserer hendelser på serversiden tilbake til filsystemet. Serverne forblir ansvarlige for tilgjengelighet, autorisering, lagringskvote, versjoner og delinger, uten å kunne behandle filinnhold eller krypterte navn i klartekst.

Forklaringen følger en fil fra klienten via nøkler, krypterte blokker og API til lagring på serversiden. Synkronisering, delinger, versjonering, diagnostikk og uavhengig sikkerhetskopiering bygger på denne dataveien.

Proton Drive lagrer data sentralt, men flytter kryptering og signaturkontroll til pålitelige klienter. Artikkelen følger en fil fra lokal endring via nøkkelhierarki, blokker og revisjon til synkronisering, deling og gjenoppretting.

## Arkitektur: sentral tjeneste, pålitelig klient

Proton valgte sentral skylagring i stedet for et peer-to-peer-nettverk. Servere leverer API, objektlagring, hendelser, medlemskap, kvoter, versjoner og tilgjengelighet; klientene håndterer klartekstbehandling og kryptografi. Proton forklarer begrunnelsen for denne modellen i [sammenligningen av sentral og desentral lagring](https://proton.me/blog/centalized-vs-p2p-protondrive).

Den tekniske banen har seks nivåer:

1. **Identitet og økt:** Klienten autentiserer en Proton-konto, laster krypterte adressenøkler og låser dem opp lokalt.
2. **Volume og Share:** Et Volume begrenser lagringsområdet. Shares definerer trerøtter, medlemmer og rettigheter.
3. **Node og Link:** En fil eller mappe er en Node; en Link plasserer den i det krypterte hierarkiet.
4. **Revisjon og blokk:** En fil kan ha flere revisjoner. Innhold deles opp i blokker og krypteres på klientsiden.
5. **API og objektlagring:** Klienten overfører krypterte metadata, nøkkelpakker, manifester og blokker via HTTPS.
6. **Lokalt filsystem:** Skrivebordsklienter projiserer skytreet som plassholdere eller lokale filer, overvåker endringer og løser konflikter.

[Proton Drive-sikkerhetsmodellen](https://proton.me/blog/protondrive-security) dokumenterer Volume, Share, Node, Link og hierarkiske nøkler. [Kryptografioppdateringen fra 2026](https://proton.me/blog/drive-cryptography-update) beskriver i tillegg Node, Revision, Block og Content Key Package i den delte klientmotoren.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-proton-drive.svg?v=20260813" title="Interaktive Infografik: Proton Drive von Konto und Share über Node-, Revisions- und Blockverschlüsselung bis zu API, SDK, Desktop-Sync, CLI, Sharing, Versionierung, Recovery und unabhängiger Sicherung" loading="lazy">
  <a href="/images/kb-interaktiv-proton-drive.svg?v=20260813">Åpne den interaktive grafikken direkte</a>.
</iframe>

## Datamodell: Volume, Share, Node, Link og Revision

Et **Volume** er et tildelt lagringsområde. Standarddelingen peker på roten av det private katalogtreet. Flere **Shares** kan peke på filer eller mapper og knytte medlemmer til roller. En Share er derfor både et tilgangspunkt og en kryptografisk grense.

En **Node** beskriver en fil eller mappe og inneholder tekniske attributter og kryptografisk materiale. En **Link** plasserer Noden i treet og kobler den til foreldrenoden. For filer forvalter **Revisions** påfølgende innholdsversjoner; en revisjon refererer til krypterte **Blocks**. Det begrepsmessige skillet forklarer viktige driftsfenomener:

- Omdøping endrer den krypterte trebetegnelsen, ikke nødvendigvis filinnholdet.
- Flytting endrer foreldrereferansen og dermed kryptografisk beskyttet Link-informasjon.
- Erstatting oppretter en revisjon av samme filobjekt, mens «behold begge» oppretter en annen Node.
- Deling delegerer tilgang til en Share-rot, i stedet for å dekryptere alt underliggende innhold på serversiden.
- Sletting flytter først en oppføring til Proton Drive-papirkurven; lokale papirkurver utgjør en separat tilstand.

Den offisielle [versjonshåndteringen](https://proton.me/support/version-history) beskriver Replace, Keep both, Skip og gjenoppretting av eldre revisjoner. [Dokumentasjonen om sletting og gjenoppretting](https://proton.me/support/proton-drive-delete-restore-synced-files) forklarer forskjellen mellom skyens papirkurv og lokal papirkurv.

Volume, Share, Node og Revision beskriver hvilket objekt som lagres. Nøkkelhierarkiet forklarer deretter hvem som kan dekryptere og signere navn, metadata og innhold.

## Nøkkelhierarki og OpenPGP

Nøkler og passfraser opprettes på klienten og overføres bare kryptert til serveren. Sikkerhetsmodellen bruker flere nivåer:

1. En Proton-adresse har et asymmetrisk nøkkelpar.
2. For en Share opprettes en tilfeldig Share-passfrase og en Share-nøkkel.
3. Share-passfrasen krypteres og signeres for hvert autoriserte medlem med medlemmets adressenøkkel.
4. Hver fil- eller mappe-Node har Node-nøkler og Node-passfrase.
5. Node-passfrasen beskyttes med nøkkelen til overordnet mappe eller Share-roten.
6. Navn og andre payloads krypteres i denne hierarkiske konteksten.
7. Filinnhold bruker en symmetrisk Content Key, der nøkkelpakken beskyttes av Node-nøkkelen.

OpenPGP fungerer her som format for meldinger og nøkkelpakker. Asymmetrisk kryptografi beskytter de symmetriske sesjonsnøklene; symmetrisk kryptering behandler de faktiske dataene effektivt. Det normative grunnlaget for OpenPGP er [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580).

For nylig opprettede filer beskriver Proton siden kryptografiomleggingen v6-PKESK-pakker, v2-SEIPD og AES-GCM. Oppsettet for en fil forblir bundet gjennom revisjonene: Ved erstatting må en klient bruke pakkeskjemaet som passer til det eksisterende filobjektet, ellers avvises revisjonen ([kryptografioppdatering](https://proton.me/blog/drive-cryptography-update)). Dette er en interoperabilitetsgrunn til å oppdatere klienter kontrollert.

## Kryptert, signert og lesbar for serveren

«Ende-til-ende-kryptert» betyr ikke at serveren ikke ser noen driftsdata. I [Drive-supportoversikten](https://proton.me/support/drive) oppgir Proton filinnhold, filnavn og mappenavn som E2EE, men påpeker samtidig at enkelte felt, som filstørrelse, forblir lesbare for serveren slik at funksjoner som sortering er mulig.

For en trusselmodell skilles dataklasser:

| Dataklasse | Beskyttelsesbane | Administratorrelevans |
|---|---|---|
| Filinnhold | klientsidekrypterte blokker | Serveren kan ikke indeksere eller reparere innholdet |
| Fil-/mappenavn | kryptert payload i treet | navnesøk på serversiden er begrenset |
| Nøkkelpakker | kryptert og signert | Gjenoppretting avhenger av private nøkler |
| Signaturer/manifester | integritets- og opphavsbevis | Advarsler må ikke ignoreres generelt |
| Filstørrelse | felt lesbart for serveren | Volum- og trafikkmønstre forblir synlige |
| Konto-, økt- og Share-referanse | Control Plane-metadata | Autorisering og revisjon forblir på serversiden |
| IP, tidspunkt, klientversjon | drifts-/sikkerhetsmetadata | Del av trusselmodellen for leverandør og endepunkt |

Digitale signaturer bekrefter opplaster og uendrede payloads. [Signaturhåndtering](https://proton.me/support/drive-signature-management) forklarer OpenPGP-signaturkontroll og advarsler. En signatur beviser ikke at innholdet er ufarlig eller faglig riktig; den dokumenterer opphav eller uendret tilstand i den kontrollerte nøkkelkonteksten.

## Blokker, revisjoner og commit-grense

Filer deles opp i blokker på klientsiden. Klienten oppretter eller overtar Content Key, krypterer blokkene, overfører dem og publiserer revisjonen med tilhørende kryptografisk informasjon. Serveren kan lagre og levere blokker uten å kjenne klarteksten.

Opplastingsfremdrift er ennå ikke en faglig commit. En administrator skiller mellom:

- lokal fil er lest fullstendig,
- alle blokker er kryptert,
- alle blokker er overført uten feil,
- revisjonen er godtatt på serversiden,
- hendelsesmarkøren er oppdatert,
- filen kan løses opp i en annen klient,
- signatur og innhold er verifisert ved nedlasting.

Ved store nedlastinger dekrypterer og kontrollerer klienten data lokalt. Nettklienten bruker en strømmesti med Service Worker for omfattende data, slik at ikke hele arkivet må ligge i nettleserens minne; Proton beskriver prosessen i [webnedlasting og strømming](https://proton.me/blog/proton-drive-web-encryption-technique).

## Drive SDK og klientarkitektur

Klientlandskapet oppsto opprinnelig fra plattformspesifikke implementeringer. Det felles **Drive SDK** standardiserer konto-, API-, cache-, Node-, Share-, opplastings-, nedlastings- og kryptografibaner. Kildekoden ligger i det offisielle [Proton Drive SDK Repository](https://github.com/ProtonDriveApps/sdk); pakkeavtalen for JavaScript/TypeScript er publisert som [`@protontech/drive-sdk`](https://www.npmjs.com/package/@protontech/drive-sdk).

Den offisielle GitHub-organisasjonen viser de viktigste teknologistakkene ([ProtonDriveApps](https://github.com/ProtonDriveApps)):

| Klient/komponent | Hovedstack | Driftsrolle |
|---|---|---|
| Web | TypeScript/JavaScript og nettleserkryptografi | interaktiv tilgang, strømming, deling |
| Drive SDK / CLI | TypeScript; CLI som Bun-pakket enkeltprogram | felles filoperasjoner og automatisering |
| [Windows](https://github.com/ProtonDriveApps/windows-drive) | C#/.NET pluss felles SDK-/Crypto-komponenter | bakgrunnssynkronisering, Explorer og Cloud Files |
| [macOS](https://github.com/ProtonDriveApps/mac-drive) | Swift pluss felles SDK-komponenter | bakgrunnssynkronisering, Finder/File Provider |
| Android | Kotlin | mobile filer, frakoblet tilstand og bilder |
| iOS/iPadOS | Swift | mobile filer, frakoblet tilstand og bilder |

SDK-tilnærmingen reduserer ulike kryptografi- og filimplementeringer, men opphever ikke plattformgrenser. Navneregler i Windows, File Provider-atferd i macOS, bakgrunnsgrenser på mobil og Linux-skrivebordsintegrasjon forblir oppgaver for den enkelte klienten.

Det kryptografiske API-et er ennå ingen filsynkroniseringstjeneste. Skrivebordsklienten overvåker lokale og eksterne endringer, knytter dem til hverandre og avgjør om det skal lastes opp, lastes ned eller oppstå konflikt.

## Skrivebordssynkronisering som tilstandsmaskin

Skrivebordssynkronisering holder ikke en enkel bytekopi, men avstemmer to foranderlige navnerom. En abstrahert syklus er:

1. Last inn lokal filsystemtilstand og skyens hendelsesmarkør.
2. Fastslå lokale endringer, hendelser på serversiden og ventende overføringer.
3. Normaliser navn og plattformregler, eller marker konflikter.
4. Løs opp nødvendig nøkkelhierarki og Nodes.
5. Planlegg opplasting, nedlasting, flytting, sletting eller hydrering.
6. Overfør innhold i blokker og committ revisjonen.
7. Oppdater lokale plassholdere, attributter, cache og markør atomisk.
8. Prøv på nytt ved feil med stabil objektreferanse, eller eskaler synlig.

Synkroniseringsstatusen «synkronisert» er bare meningsfull for den aktuelle klienten og markøren. En annen klient, en deling eller en separat sikkerhetskopi gir et sterkere ende-til-ende-bevis.

## Filer på forespørsel og lokale tilstander

Windows og macOS viser skytreet i filbehandleren uten å lagre alt innhold permanent lokalt. [Windows-dokumentasjonen for On-Demand](https://proton.me/support/proton-drive-windows-on-demand-sync) skiller mellom **Available when online**, **Available on this device** og **Always available on this device**. [macOS-veiledningen](https://proton.me/support/drive-macos-guide) bruker Online Only, midlertidig lokal og Make Available Offline.

Et synlig filnavn beviser derfor ingen lokal kopi. For sikkerhetskopiering, antivirus, indeksering, byggesystem eller vedleggsflyt må det avklares om filen:

- bare finnes som plassholder,
- hydreres ved åpning,
- er mellomlagret lokalt midlertidig,
- er eksplisitt festet for frakoblet bruk,
- er endret lokalt, men ennå ikke lastet opp,
- bare ligger på denne enheten på grunn av en feil.

Å bygge inn to skyklienter på forespørsel i samme tre er risikabelt. [Windows-feilhjelpen](https://proton.me/support/proton-drive-windows-common-issues) nevner uttrykkelig filer fra annen skylagring som et problem når de bare er tilgjengelige på nettet. En sikkerhetskopieringsjobb fester inndatamengden lokalt eller bruker den offisielle CLI-en for eksplisitte overføringer.

## Konflikter, navn og drift med flere enheter

Konflikter oppstår når klient og sky endrer samme logiske Node uavhengig, eller når målplattformen ikke kan representere et navn. Typiske klasser er:

- parallelle innholdsendringer av samme revisjon,
- sletting mot endring,
- omdøping mot flytting,
- store/små bokstaver på plattformer med ulik case-semantikk,
- tegn eller reserverte navn som ikke er tillatt i Windows,
- opplastinger med samme navn med Replace, Keep both eller Skip,
- endring frakoblet etter lang hendelsesforsinkelse,
- lokal fil låst av en applikasjon under overføring.

Proton dokumenterer at en konflikt kan opprette en omdøpt fil eller ny versjon ([Shared with me i Windows](https://proton.me/support/shared-with-me-windows)). Windows-feilhjelpen beskriver åpne filer og navn som ikke er tillatt ([vanlige synkroniseringsproblemer](https://proton.me/support/proton-drive-windows-common-issues)). Administratorer ser ikke bare etter «feil», men også etter konfliktkopier, ventende opplastinger og filer som aldri kom til skyen på grunn av navneregler.

## Klientinventar, binærfil og logger

Et administrert endepunkt inventariserer plattform, klientpakke, signatur, autostart, konto, synkroniseringsrot, lokal tilgjengelighet, ledig plass og loggsti. Versjonsnumre hører hjemme i driftsinventaret og ikke som statiske artikkelfakta.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Clientinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process ProtonDrive -ErrorAction SilentlyContinue
Get-AuthenticodeSignature "$env:LOCALAPPDATA\Proton\Drive\ProtonDrive.exe"
Get-ChildItem "$env:LOCALAPPDATA\Proton\ProtonDrive\Logs"
proton-drive version</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive version
file ./proton-drive
sha256sum ./proton-drive
ldd ./proton-drive</code></pre>
  </div>
</div>

[`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) kontrollerer Windows-prosessen; [`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) kontrollerer kodesignaturen; [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventariserer logger. Under Unix identifiserer [`file`](https://man7.org/linux/man-pages/man1/file.1.html) binærfilen, [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) hashen og [`ldd`](https://man7.org/linux/man-pages/man1/ldd.1.html) dynamiske avhengigheter. Støttet loggtilgang for Windows og macOS finnes i [Proton Drive Logs](https://proton.me/support/proton-drive-logs).

## CLI: eksplisitt operasjon i stedet for bakgrunnssynkronisering

Proton Drive CLI er en SDK-basert klient for Windows, macOS og Linux. Pålogging skjer via nettleserflyt; økter lagres ifølge [CLI-introduksjonen](https://proton.me/blog/proton-drive-cli) i Windows Credential Manager, macOS Keychain eller libsecret. CLI-en lister, laster opp og ned, og administrerer filer, papirkurv, delinger og invitasjoner. Den kjører som enkeltoperasjon og avsluttes deretter.

Den viktigste driftsgrensen er uttrykkelig dokumentert: CLI-en erstatter ikke en synkroniseringsmotor for skrivebord. En planlagt opplasting oppdager ikke automatisk slettinger, omdøpinger, toveis konflikter eller vellykket resultat på en annen enhet. Den som bruker CLI-en som transport for sikkerhetskopiering, implementerer selv kildeinventar, retning, sletteregel, gjentakelse, exitkode, målverifisering og oppbevaring.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Anmeldung und Proton-Drive-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | ConvertFrom-Json
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | jq .
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
</div>

Alle CLI-kommandoer og hjelpetekster er dokumentert i [Using Proton Drive CLI](https://proton.me/support/drive-cli). [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) og [`jq`](https://jqlang.org/manual/) behandler maskinlesbar utdata. Automatiseringer baserer seg på JSON-felt og exitkode, ikke på lokaliserte skjermtekster.

## Opplasting, konfliktstrategi og verifikasjon

En repeterbar opplasting angir lokal kilde, målrot og konfliktstrategi. Jokertegn utvides av skallet som kaller dem og kan gi ulike mengder i Windows og Unix. Kritiske jobber oppretter derfor på forhånd et manifest med bane, størrelse og hash.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Upload und Manifest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\reports -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\reports-manifest.csv -NoTypeInformation
proton-drive filesystem upload .\reports\* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json | Out-File .\drive-result.json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./reports -type f -print0 | sort -z | xargs -0 sha256sum > reports-manifest.sha256
proton-drive filesystem upload ./reports/* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json > drive-result.json
jq . drive-result.json</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv), [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) og [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html) oppretter den lokale ønskemengden. Leverandørlisten beviser imidlertid ennå ikke likt innhold. For kritiske data følger en nedlasting til et separat mål og en hashsammenligning.

## Nedlasting og ende-til-ende-godkjenning

En nedlastingstest bruker en ny målbane, slik at ingen eksisterende lokal fil skjuler resultatet. Deretter kontrolleres det mot kildemanifestet. Ved flere revisjoner må testen angi hvilken revisjon som forventes.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Downloadprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">New-Item -ItemType Directory .\restore-test
proton-drive filesystem download /my-files/Reports .\restore-test
Get-ChildItem .\restore-test -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\restore-manifest.csv -NoTypeInformation</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mkdir -p ./restore-test
proton-drive filesystem download /my-files/Reports ./restore-test
find ./restore-test -type f -print0 | sort -z | xargs -0 sha256sum > restore-manifest.sha256
diff -u reports-manifest.sha256 restore-manifest.sha256</code></pre>
  </div>
</div>

[`New-Item`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item) og [`mkdir`](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html) oppretter det isolerte målet; [`diff`](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html) sammenligner manifester. Absolutte eller ulikt prefiksede baner normaliseres før sammenligning.

## Deling til kontoer og via offentlige lenker

Proton Drive har minst to delingsmodeller:

- **Medlemskap/invitasjon:** En person blir medlem av en Share med Viewer- eller Editor-rolle. Share-passfrasen pakkes for medlemmets adressenøkkel.
- **Offentlig lenke:** En lenke gir tilgang uten Proton-konto. Et passord beskytter Share-passfrasen; lenke og passord utgjør sammen tilgangshemmeligheten.

[Tilgangshåndtering](https://proton.me/support/drive-manage-access-shared-files) dokumenterer roller, Editor-delegering, tilbakekalling og separate innstillinger for e-postinvitasjon og Public Link. [Passord og utløpsdato](https://proton.me/support/password-protect-files-proton-drive) begrenser lenketilgang. Dersom et tilfeldig passord sendes med som URL-fragment, ser nettleser og mottaker den fullstendige lenken; fragmentet sendes riktignok ikke som HTTP-bane til serveren, men kan lekke via utklippstavle, nettleserhistorikk, skjermbilder eller endepunkttelemetri.

Administratorer inventariserer eier, Share-rot, medlemmer, rolle, videredelingsrett, Public Link-status, utløpsdato og siste kontroll. En Editor kan, avhengig av innstillingen, selv dele og endre rettigheter. Offboarding tilbakekaller derfor ikke bare kontoen, men kontrollerer alle Shares og eksterne lenker.

## Versjoner, papirkurv og sikkerhetskopiering

Versjonshistorikk lagrer tidligere revisjoner av samme filobjekt. En gjenopprettet revisjon sletter ikke automatisk nyere versjoner. Papirkurven beholder slettede Nodes frem til endelig sletting. Begge funksjonene er verdifulle, men forblir innenfor samme administrative og kryptografiske feilområde.

De beskytter ikke pålitelig mot:

- tap eller manglende gjenopprettbarhet av kontonøkkelen,
- feilaktige rettigheter eller ondsinnet endelig sletting,
- kvote-, kontrakts- og leverandørhendelser,
- feil som oppstod før den første vellykkede skyrevisjonen,
- uoppdagede synkroniseringsutelatelser,
- organisasjonsomfattende kompromittering av identitet eller endepunkt.

En uavhengig sikkerhetskopi leser en definert skymengde med en autorisert klient, skriver den til et separat administrert mål, utstyrer den med manifest, oppbevaring og uforanderlig historikk og gjennomfører gjenopprettingstester. NIST SP 800-34 behandler sikkerhetskopiering, alternativ lagring og gjenopprettingstester som sammenhengende kontroller for beredskapsplanlegging ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)). Grunnleggende informasjon finnes under [Backup og Disaster Recovery](/kb/backup-dr) og [Storage](/kb/storage).

## Identitet, økt og tillit til endepunktet

Ende-til-ende-kryptering flytter tillit fra leverandøren til klient, konto og endepunkt. Den som kontrollerer den opplåste enheten, nettleserøkten eller kontoen, kan dekryptere og signere endringer på brukerens vegne. Derfor hører MFA, maskinvaresikkerhetsnøkler, enhetsbeskyttelse, tilbakekalling av økter, klientopprinnelse og beskyttelse av lokal cache sammen.

[Trusselmodellen](https://proton.me/blog/proton-drive-threat-model) nevner blant annet kompromitterte enheter, phishing og ektheten til offisielle klienter som grenser. Åpen kildekode muliggjør kodekontroll, men erstatter ikke verifisering av binærfilen som faktisk kjøres, og dens oppdateringsbane.

Malwarekontroll er også begrenset: Den valgfrie [oppdagelsen av skadelige filer](https://proton.me/support/proton-drive-malware-protection) sammenligner på forespørsel en hash med kjente skadevarehasher. Den dekrypterer ikke hele beholdningen på serversiden, oppdager ikke ukjent innhold og erstatter ingen endepunktssensor.

## Tilbakestilling av passord og datagjenoppretting

Kontogjenoppretting og datagjenoppretting er ulike prosesser. En tilbakestilling av passord kan gjenopprette tilgangen til kontoen, men gjøre de tidligere private nøklene utilgjengelige i første omgang. Proton Drive viser da de gamle dataene som låst.

Den offisielle [Drive-datagjenopprettingen](https://proton.me/support/drive-data-recovery) oppgir tidligere passord, Recovery Phrase eller Recovery File som opplåsingsmetoder. Nye opplastinger etter tilbakestillingen kan oppstå under nye nøkler, mens gammel beholdning forblir låst; bilder og album kan kreve et ekstra sammenslåingstrinn.

Et organisatorisk runbook dokumenterer derfor:

- hvem som kan utløse en tilbakestilling av passord,
- hvilke gjenopprettingsmetoder som er konfigurert og testet på forhånd,
- hvor Recovery Phrase eller File lagres frakoblet og tilgangsbeskyttet,
- hvordan gamle og nye nøkkeltilstander oppdages,
- hvordan Shares og medlemmer kontrolleres etter gjenoppretting,
- når en låst Drive aldri må slettes.

«Unlock later» bevarer muligheten for senere gjenoppretting. «Delete locked drive» er irreversibel datasletting og hører ikke hjemme i en ubekreftet standardprosedyre.

Når identitet, nøkler og lokal tilstand er plausible, følger nettverksbanen. Proxy, TLS, API-endepunkter og blokkert overføring må kontrolleres separat fra den kryptografiske datamodellen.

## Nettverksbane og transportdiagnostikk

HTTPS beskytter transporten mellom klienten og Proton API eller lagringsendepunktene. Det erstatter ikke ende-til-ende-kryptering, men beskytter økt- og driftsmetadata, endepunktidentitet og tilgjengelighet. Proxy, DNS-filter, TLS-inspeksjon, VPN, MTU eller ustabile forbindelser kan påvirke Control Plane-forespørsler og blokkoverføringer ulikt.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Netzwerkdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName drive.proton.me
Test-NetConnection drive.proton.me -Port 443
curl.exe -sS -D - -o NUL https://drive.proton.me/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">getent ahosts drive.proton.me
curl -sS -D - -o /dev/null https://drive.proton.me/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect drive.proton.me:443 -servername drive.proton.me -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) og [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) kontrollerer navneoppløsning; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) TCP 443; [`curl`](https://curl.se/docs/manpage.html) HTTP og omdirigeringer; [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) eller [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) aktive forbindelser; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) TLS og SNI. Testen mot nettdomenet beviser ikke at hvert API-/blokkendepunkt er tilgjengelig; klientloggene viser faktisk mislykket mål- og forespørselsklasse.

## Observerbarhet og diagnosekjede

En fornuftig diagnose begynner ved fagobjektet:

1. Fastslå konto, Share, logisk bane og berørt klient.
2. Skill lokal tilstand: plassholder, hydrert, festet, endret, låst eller feilaktig.
3. Kontroller aktivitetsvisning og logger for Node-/Revision-/Transfer-referanse.
4. Kontroller ledig lokal plass og skykvote.
5. Kontroller navn- og banekompatibilitet.
6. Kontroller DNS, TCP, TLS og proxy/VPN mot det faktiske endepunktet.
7. Behandle signatur-, dekrypterings- og nøkkeladvarsler separat fra transportfeil.
8. List eller last ned filen med offisiell CLI eller en annen klient.
9. Sammenlign nedlastingshash og forventet revisjon.
10. Først deretter vurder pause/resume, reautentisering, oppdatering eller cacheinngrep.

[Windows-synkroniseringsfeilene](https://proton.me/support/proton-drive-windows-common-issues) nevner fremdrift, åpne filer, andre skyklienter, pause/resume, navneregler og nettverk. Disse brukerstøttene suppleres i administrasjonsdrift med objektinventar og ende-til-ende-verifikasjon. Generell metodikk finnes under [Troubleshooting](/kb/troubleshooting).

## Ytelses- og kapasitetsmodell

Kostnadene oppstår ikke bare av overførte byte. Mange små filer skaper arbeid per Node med nøkler, metadata, API og signaturer. Dype trær krever gjentatt hierarkioppløsning. Et stort filinnhold kan derimot behandles mer effektivt i parallelle blokker. Proton påpeker i Windows-feilhjelpen at mange små eller dypt nestede filer kan synkroniseres langsommere enn én enkelt fil av samme størrelse.

Kapasitetsplanlegging tar hensyn til:

- skykvote inkludert versjoner og papirkurv,
- lokale frakoblet- og cachetilstander,
- midlertidig plass for opplasting, nedlasting og konfliktkopier,
- CPU for kryptografi og hashing,
- filantall og tredybde,
- hendelsesforsinkelse etter frakoblet tid,
- Fair Use-/begrensningsatferd ved automatiserte overføringer.

CLI-introduksjonen advarer mot gjentatt full opplasting av uendrede data og viser til Fair Use ([Proton Drive CLI](https://proton.me/blog/proton-drive-cli)). En planlagt jobb vedlikeholder derfor eget kildemanifest eller kontrollerer måltilstand, i stedet for å behandle hver kjøring som full opplasting.

## Drift med Windows, macOS og Linux

Windows og macOS tilbyr filbehandlerintegrasjon og bakgrunnssynkronisering. Mobilapper håndterer opplasting, nedlasting, frakoblet-objekter og bilder, men er ingen identisk erstatning for en filsystemtjeneste på skrivebordet. Den offisielle CLI-en gjør fil- og delingsoperasjoner skriptbare på alle tre skrivebordsoperativsystemene.

I juni 2026 publiserte Proton CLI-en og beskrev en fullverdig Linux-skrivebordsapplikasjon med bakgrunnssynkronisering som et separat prosjekt under arbeid ([CLI-kunngjøring](https://proton.me/blog/proton-drive-cli)). Denne historiske avgrensningen forblir viktig: Selv om plattformutvalget utvikler seg videre, likestilles ikke **CLI-operasjon**, **Mount** og **toveis synkroniseringsmotor**. For den aktuelle utrullingen kontrolleres [Release Notes-siden](https://proton.me/support/drive-release-notes) som tidsavhengig kilde.

I serverjobber kjøres nettleserautentisering kontrollert, og det resulterende økttokenet lagres i operativsystemets hemmelighetslager for en dedikert tjenestekonto. En headless-jobb får ikke en administrators personlige hjemmekatalog. Exitkode, JSON-resultat, kildemanifest og målverifisering beholdes; hemmeligheter og delingslenker fjernes fra logger.

Versjoner og papirkurv beskytter mot enkeltstående brukerfeil, men erstatter ikke en uavhengig gjenoppretting. Runbook må behandle konto, nøkkeltilgang, lokale data og verifisert eksport samlet.

## Runbook for sikkerhetskopiering og gjenoppretting

Proton Drive kan være mål for en sikkerhetskopi, men er fortsatt ingen erstatning for et gjenopprettingsdesign. Et robust runbook omfatter:

1. Definer kildemengde og Source of Truth.
2. Hydrer lokale On-Demand-data fullstendig før filbasert sikkerhetskopiering, eller eksporter direkte med CLI.
3. Opprett filmanifest med bane, størrelse og hash.
4. Utfør opplasting med eksplisitt konflikt- og sletteregel.
5. Arkiver CLI-exitkode og målliste.
6. Gjennomfør stikkprøve eller full nedlasting til en isolert bane.
7. Sammenlign hasher og forventede revisjoner.
8. Dokumenter delinger, versjonering, papirkurv og oppbevaring.
9. Test Proton-konto- og datagjenoppretting separat.
10. Behold en uavhengig andre kopi utenfor samme konto- og leverandørdomene.

RPO viser til den siste revisjonen som beviselig er committet og verifisert i Proton, ikke tidspunktet for lokal filskriving. RTO omfatter kontotilgang, nøkkelopplåsing, listing, nedlasting, dekryptering, båndbredde og gjenoppretting av mål.

## Teknisk historie

Proton utviklet Drive som et supplement til det krypterte Proton-økosystemet. Sikkerhetsmodellen publisert i 2020 etablerte sentral lagring med klientside OpenPGP-nøkkelhierarki, krypterte navn, Node-trær, Shares, signerte blokker og lenkedeling. Web- og mobilklienter kom før skrivebordsklientene for synkronisering; Version History ble innført i 2023 som en egen revisjonsbane ([innføring av versjonshistorikk](https://proton.me/blog/drive-version-history)).

Windows og macOS la til bakgrunnssynkronisering og filer på forespørsel med plattformspesifikk Explorer-/Finder-integrasjon. De separate implementeringene førte gradvis til felles Drive SDK. Proton beskrev tidlig i 2026 SDK-utrullingen på tvers av web, Windows, macOS, Android og iOS ([SDK-utvikling](https://proton.me/blog/drive-sdk-january-2026)).

I juni 2026 fulgte to tekniske steg: en felles kryptografiomlegging til v6 PKESK, v2 SEIPD og AES-GCM for nye filobjekter, samt den SDK-baserte CLI-en for Windows, macOS og Linux. Dette muliggjorde offisielt skriptbar tilgang uten å gjenskape den kryptografiske klientlogikken. Arkitekturen forblir likevel klientsentrert: Servere lagrer og koordinerer, autoriserte klienter dekrypterer, signerer og avstemmer.

## Sjekkliste for administratorer

Ved klientsidekryptert lagring må datamodell, nøkler, synkronisering og uavhengig sikkerhetskopiering forstås samlet. Sjekklisten sammenfatter disse punktene for driftsgodkjenning.

- **Datamodell:** Skill mellom Volume, Share, Node, Link, Revision og Block.
- **Kryptografi:** Dokumenter kontekst for adresse-, Share-, Node- og Content Key samt pakkeskjema.
- **Metadata:** Skill E2EE-felt og serverlesbare driftsfelt i trusselmodellen.
- **Klienter:** Inventariser plattform, teknologi, SDK, filsystemintegrasjon, binærsignatur og loggsti.
- **Synkronisering:** Overvåk lokal tilstand, hendelsesmarkør, konfliktkopier, navn og ventende overføringer.
- **CLI:** Ikke behandle eksplisitt filoperasjon som permanent synkroniserings- eller sikkerhetskopieringsmotor.
- **Deling:** Kontroller eier, medlemmer, roller, Editor-delegering, Public Link, passord og utløp.
- **Gjenoppretting:** Test kontopassord, dataopplåsing, Recovery Phrase/File og Share-tilgang separat.
- **Sikkerhetskopiering:** Ha uavhengig kopi, manifest, versjonering og gjenopprettingsnedlasting tilgjengelig.
- **Godkjenning:** Dekrypter fil på en annen klient eller i isolert nedlasting, og åpne den med kontroll av signatur og hash.

## Kilder

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [Proton Drive – Supportoversikt](https://proton.me/support/drive)
- [Proton Drive – CLI](https://proton.me/support/drive-cli)
- [Proton – sentral og desentral lagring](https://proton.me/blog/centalized-vs-p2p-protondrive)
- [Proton Drive – sikkerhetsmodell](https://proton.me/blog/protondrive-security)
- [Proton Drive – kryptografioppdatering](https://proton.me/blog/drive-cryptography-update)
- [Proton Drive – versjonshåndtering](https://proton.me/support/version-history)
- [Proton Drive – sletting og gjenoppretting av synkroniserte filer](https://proton.me/support/proton-drive-delete-restore-synced-files)
- [RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [Proton Drive – signaturhåndtering](https://proton.me/support/drive-signature-management)
- [Proton Drive – webnedlasting og strømming](https://proton.me/blog/proton-drive-web-encryption-technique)
- [Proton Drive – SDK Repository](https://github.com/ProtonDriveApps/sdk)
- [npm – @protontech/drive-sdk](https://www.npmjs.com/package/@protontech/drive-sdk)
- [GitHub – ProtonDriveApps](https://github.com/ProtonDriveApps)
- [Windows](https://github.com/ProtonDriveApps/windows-drive)
- [macOS](https://github.com/ProtonDriveApps/mac-drive)
- [Proton Drive – Windows On-Demand Sync](https://proton.me/support/proton-drive-windows-on-demand-sync)
- [Proton Drive – macOS-veiledning](https://proton.me/support/drive-macos-guide)
- [Proton Drive – vanlige synkroniseringsproblemer i Windows](https://proton.me/support/proton-drive-windows-common-issues)
- [Proton Drive – Shared with me i Windows](https://proton.me/support/shared-with-me-windows)
- [Microsoft – Get-Process](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html)
- [Proton Drive – logger](https://proton.me/support/proton-drive-logs)
- [Proton – introduksjon til Drive CLI](https://proton.me/blog/proton-drive-cli)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [jq – håndbok](https://jqlang.org/manual/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [Microsoft – New-Item](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)
- [GNU Coreutils – mkdir](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html)
- [GNU Diffutils – diff](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html)
- [Proton Drive – tilgangshåndtering](https://proton.me/support/drive-manage-access-shared-files)
- [Proton Drive – passordbeskyttelse og utløpsdato](https://proton.me/support/password-protect-files-proton-drive)
- [Proton Drive – trusselmodell](https://proton.me/blog/proton-drive-threat-model)
- [Proton Drive – oppdagelse av skadelige filer](https://proton.me/support/proton-drive-malware-protection)
- [Proton Drive – datagjenoppretting](https://proton.me/support/drive-data-recovery)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [curl – håndbok](https://curl.se/docs/manpage.html)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Proton Drive – Release Notes](https://proton.me/support/drive-release-notes)
- [Proton Drive – innføring av versjonshistorikk](https://proton.me/blog/drive-version-history)
- [Proton Drive – SDK-utvikling 2026](https://proton.me/blog/drive-sdk-january-2026)
