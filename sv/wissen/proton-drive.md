---
title: "Proton Drive: Kryptografisk datamodell och synkroniseringsdrift"
blatt: "proton-drive"
description: "Proton Drive för lagrings-, säkerhets- och plattformsadministratörer: central molnlagring med klientsidig end-to-end-kryptering, volymer, delningar, noder, revisioner och block, OpenPGP-nyckelhierarki och signaturer, Drive SDK, skrivbordssynkronisering och filer på begäran, CLI-automatisering, delning, konflikter, versioner, loggning, återställning och teknisk historia."
fakten:
  - label: Systemroll
    wert: centralt driven molnfillagring med klientsidig end-to-end-kryptering, synkronisering, delning och versionshantering
    href: https://proton.me/support/drive
  - label: Förtroendemodell
    wert: Servern lagrar krypterat innehåll och namn; klartext och upplåst nyckelmaterial uppstår i den auktoriserade klienten
    href: https://proton.me/blog/protondrive-security
  - label: Datamodell
    wert: En volym innehåller delningar; en delning pekar på en trädrot; filer och mappar är noder med länkar, nycklar och revisioner
    href: https://proton.me/blog/protondrive-security
  - label: Filöverföring
    wert: Revisioner består av krypterade block; klienten krypterar, överför, kontrollerar och sätter samman objektet
    href: https://proton.me/blog/drive-cryptography-update
  - label: Kryptografi
    wert: OpenPGP-baserad nyckelhierarki; nya filer använder v6 PKESK, v2 SEIPD och AES-GCM, äldre revisioner förblir schemabundna
    href: https://proton.me/blog/drive-cryptography-update
  - label: Metadata
    wert: Innehåll samt fil- och mappnamn är E2EE; vissa fält, såsom filstorlek, förblir enligt Proton läsbara för servern
    href: https://proton.me/support/drive
  - label: Integritet
    wert: signerad nyckel-, träd- och blockinformation samt OpenPGP-signaturer upptäcker manipulationer
    href: https://proton.me/support/drive-signature-management
  - label: Klientkärna
    wert: Drive SDK enhetliggör filoperationer och kryptografi för webb-, skrivbords-, mobil- och CLI-klienter
    href: https://github.com/ProtonDriveApps/sdk
  - label: Skrivbordssynkronisering
    wert: Windows och macOS integrerar platshållare på begäran, lokala offlinetillstånd och bakgrundssynkronisering i Explorer respektive Finder
    href: https://proton.me/support/proton-drive-windows-on-demand-sync
  - label: CLI
    wert: officiellt fristående program för Windows, macOS och Linux; fil- och delningsoperationer, men ingen permanent synkroniseringsmotor för skrivbordet
    href: https://proton.me/support/drive-cli
  - label: Återställningskärna
    wert: Enbart kontoåtkomst räcker inte efter lösenordsåterställning; den gamla nyckeln låses upp igen med tidigare lösenord, Recovery Phrase eller Recovery File
    href: https://proton.me/support/drive-data-recovery
  - label: Säkerhetskopieringsregel
    wert: Synkronisering och versionshistorik är ingen oberoende säkerhetskopia; kritiska data behöver separat export, kontrollsummor, bevarande och återställningstest
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
werbung:
  - newsletter
ctaThemen:
  - proton-drive
translationSourceHash: 6063c7d85190de63a8973bc7cdb304fff10f81fd617a2044cf40f8371849717c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:49:51.965Z
translationReview: automatic
---

# Proton Drive: Kryptografisk datamodell och synkroniseringsdrift

Proton Drive är en centralt driven molnfillagring vars klienter krypterar innehåll och väsentliga metadata före uppladdning. Tjänsten kombinerar webbåtkomst, skrivbordssynkronisering, mobilappar, delning, versioner och en kommandorad med en end-to-end-krypterad datamodell. [Supportöversikten](https://proton.me/support/drive) nämner åtkomst via webb, Windows, macOS, Android och iOS/iPadOS; den officiella [Drive CLI](https://proton.me/support/drive-cli) kompletterar med skriptbara operationer i Windows, macOS och Linux.

För administratörer är Proton Drive varken en klassisk fildelning eller enbart en uppladdningsportal. En auktoriserad klient hanterar kontosession och nycklar, löser upp ett krypterat träd, övervakar lokala ändringar, skapar revisioner, krypterar block och synkroniserar serverhändelser tillbaka till filsystemet. Servrarna ansvarar fortsatt för tillgänglighet, auktorisering, lagringskvot, versioner och delningar utan att kunna behandla filinnehåll eller krypterade namn i klartext.

Förklaringen följer en fil från klienten via nycklar, krypterade block och API till serverlagringen. Synkronisering, delningar, versionshantering, diagnostik och oberoende säkerhetskopiering bygger på denna dataväg.

Proton Drive lagrar data centralt men flyttar kryptering och signaturkontroll till betrodda klienter. Artikeln följer en fil från den lokala ändringen via nyckelhierarki, block och revision till synkronisering, delning och återställning.

## Arkitektur: central tjänst, betrodd klient

Proton valde central molnlagring i stället för ett peer-to-peer-nätverk. Servrar tillhandahåller API, objektlagring, händelser, medlemskap, kvoter, versioner och tillgänglighet; klienter hanterar klartextbehandling och kryptografi. Proton beskriver motiveringen för denna modell i [klassificeringen av central och decentraliserad lagring](https://proton.me/blog/centalized-vs-p2p-protondrive).

Den tekniska vägen har sex nivåer:

1. **Identitet och session:** Klienten autentiserar ett Proton-konto, läser in krypterade adressnycklar och låser upp dem lokalt.
2. **Volym och delning:** En volym avgränsar lagringsutrymmet. Delningar definierar trädrötter, medlemmar och behörigheter.
3. **Nod och länk:** En fil eller mapp är en nod; länken placerar den i den krypterade hierarkin.
4. **Revision och block:** En fil kan ha flera revisioner. Innehållet delas upp i block och krypteras på klientsidan.
5. **API och objektlagring:** Klienten överför krypterade metadata, nyckelpaket, manifest och block via HTTPS.
6. **Lokalt filsystem:** Skrivbordsklienter projicerar molnträdet som platshållare eller lokala filer, övervakar ändringar och löser konflikter.

[Proton Drives säkerhetsmodell](https://proton.me/blog/protondrive-security) dokumenterar volym, delning, nod, länk och hierarkiska nycklar. [Kryptografiuppdateringen från 2026](https://proton.me/blog/drive-cryptography-update) beskriver dessutom nod, revision, block och innehållsnyckelpaket i den delade klientmotorn.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-proton-drive.svg?v=20260813" title="Interaktive Infografik: Proton Drive von Konto und Share über Node-, Revisions- und Blockverschlüsselung bis zu API, SDK, Desktop-Sync, CLI, Sharing, Versionierung, Recovery und unabhängiger Sicherung" loading="lazy">
  <a href="/images/kb-interaktiv-proton-drive.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Datamodell: volym, delning, nod, länk och revision

En **volym** är ett tilldelat lagringsutrymme. Dess standarddelning pekar på roten av det privata katalogträdet. Ytterligare **delningar** kan peka på filer eller mappar och koppla medlemmar till roller. En delning är därför både åtkomstpunkt och kryptografisk gräns.

En **nod** beskriver en fil eller mapp och bär tekniska attribut samt kryptografiskt material. En **länk** placerar noden i trädet och kopplar den till dess överordnade nod. För filer hanterar **revisioner** på varandra följande innehållstillstånd; en revision refererar till krypterade **block**. Den begreppsmässiga separationen förklarar viktiga driftfenomen:

- Att byta namn ändrar den krypterade trädbenämningen, inte nödvändigtvis filinnehållet.
- Att flytta ändrar föräldrareferensen och därmed kryptografiskt skyddad länkinformation.
- Att ersätta skapar en revision av samma filobjekt, medan ”behåll båda” skapar en andra nod.
- Att dela delegerar åtkomst till en delningsrot i stället för att servern dekrypterar allt underliggande innehåll.
- Att radera flyttar först en post till Proton Drives papperskorg; lokala papperskorgar utgör ett separat tillstånd.

Den officiella [versionshanteringen](https://proton.me/support/version-history) beskriver Replace, Keep both, Skip och återställning av äldre revisioner. [Dokumentationen om radering och återställning](https://proton.me/support/proton-drive-delete-restore-synced-files) förklarar skillnaden mellan molnpapperskorgen och den lokala papperskorgen.

Volym, delning, nod och revision beskriver vilket objekt som lagras. Nyckelhierarkin förklarar därefter vem som kan dekryptera och signera namn, metadata och innehåll.

## Nyckelhierarki och OpenPGP

Nycklar och lösenfraser skapas på klienten och överförs endast krypterat till servern. Säkerhetsmodellen använder flera nivåer:

1. En Proton-adress har ett asymmetriskt nyckelpar.
2. För en delning skapas en slumpmässig lösenfras för delningen och en delningsnyckel.
3. Delningens lösenfras krypteras och signeras för varje behörig medlem med dennes adressnyckel.
4. Varje fil- eller mappnod har nodnycklar och en nodlösenfras.
5. Nodlösenfrasen skyddas med nyckeln för föräldramappen respektive delningsroten.
6. Namn och andra nyttolaster krypteras i denna hierarkiska kontext.
7. Filinnehåll använder en symmetrisk innehållsnyckel vars nyckelpaket skyddas av nodnyckeln.

OpenPGP används som format för meddelanden och nyckelpaket. Asymmetrisk kryptografi skyddar de symmetriska sessionsnycklarna; symmetrisk kryptering behandlar själva data effektivt. Den normativa grunden för OpenPGP är [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580).

Proton beskriver v6-PKESK-paket, v2-SEIPD och AES-GCM för nyligen skapade filer sedan kryptografiomställningen. Schemat för en fil förblir bundet över dess revisioner: en klient måste vid ersättning använda det paketschema som passar det befintliga filobjektet, annars avvisas revisionen ([kryptografiuppdatering](https://proton.me/blog/drive-cryptography-update)). Det är ett interoperabilitetsskäl att uppdatera klienter kontrollerat.

## Krypterat, signerat och läsbart för servern

”End-to-end-krypterat” betyder inte att servern inte ser några driftdata alls. Proton anger i [Drive-supportöversikten](https://proton.me/support/drive) filinnehåll, filnamn och mappnamn som E2EE, men påpekar samtidigt att vissa fält, såsom filstorlek, förblir läsbara för servern för att funktioner som sortering ska vara möjliga.

För en hotmodell separeras dataklasser:

| Dataklass | Skyddsväg | Administrativ relevans |
|---|---|---|
| Filinnehåll | klientsidigt krypterade block | Servern kan inte indexera eller reparera innehållet |
| Fil-/mappnamn | krypterad nyttolast i trädet | serverbaserad namnsökning är begränsad |
| Nyckelpaket | krypterade och signerade | Återställning är beroende av privata nycklar |
| Signaturer/manifest | integritets- och upphovsbevis | Varningar får inte ignoreras slentrianmässigt |
| Filstorlek | serverläsbart fält | Volym- och trafikmönster förblir synliga |
| Konto-, sessions- och delningsrelation | metadata i kontrollplanet | Auktorisering och granskning förblir serverbaserade |
| IP, tidpunkt, klientversion | drift-/säkerhetsmetadata | Del av leverantörens och endpointens hotmodell |

Digitala signaturer bekräftar uppladdaren och oförändrade nyttolaster. [Signaturhantering](https://proton.me/support/drive-signature-management) förklarar OpenPGP-signaturkontroll och varningar. En signatur bevisar inte att innehållet är ofarligt eller sakligt korrekt; den styrker ursprung respektive oförändradhet i den kontrollerade nyckelkontexten.

## Block, revisioner och commitgräns

Filer delas upp i block på klientsidan. Klienten skapar eller övertar innehållsnyckeln, krypterar blocken, överför dem och publicerar revisionen med tillhörande kryptografisk information. Servern kan lagra och leverera block utan att känna till klartexten.

Uppladdningsförloppet är ännu ingen verksamhetsmässig commit. En administratör skiljer mellan:

- lokal fil fullständigt läst,
- alla block krypterade,
- alla block överförda utan fel,
- revision accepterad av servern,
- händelsemarkör uppdaterad,
- filen kan lösas upp på en andra klient,
- signatur och innehåll verifierade vid nedladdning.

Vid stora nedladdningar dekrypterar och kontrollerar klienten data lokalt. Webbklienten använder en strömningsväg med Service Worker för omfattande data, så att hela arkivet inte måste rymmas i webbläsarminnet; Proton beskriver flödet i [webbnedladdning och streaming](https://proton.me/blog/proton-drive-web-encryption-technique).

## Drive SDK och klientarkitektur

Klientlandskapet växte ursprungligen fram ur plattformsspecifika implementationer. Det gemensamma **Drive SDK** enhetliggör konto-, API-, cache-, nod-, delnings-, uppladdnings-, nedladdnings- och kryptografivägar. Källkoden finns i det officiella [Proton Drive SDK Repository](https://github.com/ProtonDriveApps/sdk); JavaScript-/TypeScript-paketkontraktet publiceras som [`@protontech/drive-sdk`](https://www.npmjs.com/package/@protontech/drive-sdk).

Den officiella GitHub-organisationen visar de väsentliga teknikstackarna ([ProtonDriveApps](https://github.com/ProtonDriveApps)):

| Klient/komponent | Huvudstack | Driftroll |
|---|---|---|
| Webb | TypeScript/JavaScript och webbläsarkryptografi | interaktiv åtkomst, streaming, delning |
| Drive SDK / CLI | TypeScript; CLI som ett Bun-paketerat fristående program | gemensamma filoperationer och automatisering |
| [Windows](https://github.com/ProtonDriveApps/windows-drive) | C#/.NET plus gemensamma SDK-/Crypto-komponenter | bakgrundssynkronisering, Explorer och Cloud Files |
| [macOS](https://github.com/ProtonDriveApps/mac-drive) | Swift plus gemensamma SDK-komponenter | bakgrundssynkronisering, Finder/File Provider |
| Android | Kotlin | mobila filer, offlinetillstånd och foton |
| iOS/iPadOS | Swift | mobila filer, offlinetillstånd och foton |

SDK-metoden minskar skillnader mellan kryptografi- och filimplementationer, men undanröjer inte plattformsgränser. Windows namnregler, macOS File Provider-beteende, begränsningar för mobil bakgrundskörning och Linux-skrivbordsintegration förblir uppgifter för respektive klient.

Det kryptografiska API:t är ännu ingen filsynkroniseringstjänst. Skrivbordsklienten övervakar lokala och fjärrändringar, kopplar dem till varandra och avgör om uppladdning, nedladdning eller konflikt krävs.

## Skrivbordssynkronisering som tillståndsmaskin

Skrivbordssynkronisering håller inte en enkel bytekopia utan reconcilerar två föränderliga namnrymder. En abstraherad cykel lyder:

1. Läs in lokalt filsystemtillstånd och molnhändelsemarkör.
2. Identifiera lokala ändringar, serverhändelser och väntande överföringar.
3. Normalisera namn och plattformsregler eller markera konflikter.
4. Lös upp nödvändig nyckelhierarki och noder.
5. Planera uppladdning, nedladdning, flytt, radering eller hydrering.
6. Överför innehåll i block och committa revisionen.
7. Uppdatera lokala platshållare, attribut, cache och markör atomärt.
8. Försök igen med stabil objektreferens vid fel eller eskalera synligt.

Synkroniseringsstatusen ”synkroniserad” är endast relevant för den aktuella klienten och markören. En andra klient, en delning eller en separat säkerhetskopia ger ett starkare end-to-end-bevis.

## Filer på begäran och lokala tillstånd

Windows och macOS visar molnträdet i filhanteraren utan att spara allt innehåll permanent lokalt. [Windows-dokumentationen om filer på begäran](https://proton.me/support/proton-drive-windows-on-demand-sync) skiljer mellan **Available when online**, **Available on this device** och **Always available on this device**. [macOS-guiden](https://proton.me/support/drive-macos-guide) har Online Only, tillfälligt lokalt och Make Available Offline.

Ett synligt filnamn bevisar därför inte att det finns en lokal kopia. För säkerhetskopiering, antivirus, indexerare, byggsystem eller bilagearbetsflöde måste det klargöras om filen:

- endast finns som platshållare,
- hydreras när den öppnas,
- mellanlagras lokalt tillfälligt,
- uttryckligen har fästs för offlineanvändning,
- har ändrats lokalt men ännu inte laddats upp,
- på grund av ett fel endast finns på denna enhet.

Att kapsla in två molnklienter med filer på begäran i samma träd är riskabelt. [Windows-felguiden](https://proton.me/support/proton-drive-windows-common-issues) nämner uttryckligen filer från annan molnlagring som ett problem om de endast finns online. Ett säkerhetskopieringsjobb fäster indatamängden lokalt eller använder den officiella CLI:n för explicita överföringar.

## Konflikter, namn och drift med flera enheter

Konflikter uppstår när klient och moln ändrar samma logiska nod oberoende av varandra eller när målplattformen inte kan representera ett namn. Typiska klasser är:

- parallell innehållsändring av samma revision,
- Delete mot Modify,
- Rename mot Move,
- versaler/gemener på plattformar med annan skiftlägessemantik,
- tecken eller reserverade namn som inte är tillåtna i Windows,
- uppladdningar med samma namn med Replace, Keep both eller Skip,
- offlineändring efter lång händelseeftersläpning,
- lokal fil låst av en applikation under överföringen.

Proton dokumenterar att en konflikt kan skapa en omdöpt fil eller ny version ([Shared with me i Windows](https://proton.me/support/shared-with-me-windows)). Windows-felguiden beskriver öppna filer och otillåtna namn ([vanliga synkroniseringsproblem](https://proton.me/support/proton-drive-windows-common-issues)). Administratörer söker inte bara efter ”fel”, utan även efter konfliktkopior, väntande uppladdningar och filer som aldrig nådde molnet på grund av namnregler.

## Klientinventering, binärfil och loggar

En hanterad endpoint inventerar plattform, klientpaket, signatur, autostart, konto, synkroniseringsrot, lokal tillgänglighet, ledigt utrymme och loggsökväg. Versionsnummer hör hemma i driftinventeringen och inte som statiska artikelfakta.

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

[`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) kontrollerar Windows-processen; [`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) kodsignaturen; [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventerar loggar. I Unix identifierar [`file`](https://man7.org/linux/man-pages/man1/file.1.html) binärfilen, [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) dess hash och [`ldd`](https://man7.org/linux/man-pages/man1/ldd.1.html) dynamiska beroenden. Åtkomst till stödda loggar för Windows och macOS finns i [Proton Drive Logs](https://proton.me/support/proton-drive-logs).

## CLI: explicit operation i stället för bakgrundssynkronisering

Proton Drive CLI är en SDK-baserad klient för Windows, macOS och Linux. Inloggning sker via webbläsarflöde; enligt [CLI-introduktionen](https://proton.me/blog/proton-drive-cli) lagras sessioner i Windows Credential Manager, macOS Keychain respektive libsecret. CLI:n listar, laddar upp och ner, hanterar filer, papperskorg, delningar och inbjudningar. Den körs som en enskild operation och avslutas därefter.

Den viktigaste driftgränsen är uttryckligen dokumenterad: CLI:n ersätter ingen synkroniseringsmotor för skrivbordet. En schemalagd uppladdning identifierar inte automatiskt raderingar, namnbyten, dubbelriktade konflikter eller framgång på en andra enhet. Den som använder CLI:n som säkerhetskopieringstransport implementerar själv källinventering, riktning, raderingsregel, upprepning, exitkod, målverifiering och bevarande.

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

Alla CLI-kommandon och hjälptexter dokumenteras i [Using Proton Drive CLI](https://proton.me/support/drive-cli). [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) och [`jq`](https://jqlang.org/manual/) bearbetar maskinläsbar utdata. Automatiseringar förlitar sig på JSON-fält och exitkod, inte på lokaliserad skärmtext.

## Uppladdning, konfliktstrategi och verifiering

En repeterbar uppladdning anger lokal källa, målrot och konfliktstrategi. Jokertecken expanderas av det anropande skalet och kan ge olika mängder i Windows och Unix. Kritiska jobb skapar därför i förväg ett manifest med sökväg, storlek och hash.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv), [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) och [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html) skapar den lokala målmängden. Leverantörslistan bevisar dock ännu ingen innehållslikhet. För kritiska data följer nedladdning till ett separat mål och hashjämförelse.

## Nedladdning och end-to-end-godkännande

Ett nedladdningstest använder en ny målsökväg så att ingen befintlig lokal fil döljer resultatet. Därefter kontrolleras den mot källmanifestet. Vid flera revisioner måste testet fastställa vilken revision som förväntas.

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

[`New-Item`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item) och [`mkdir`](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html) skapar det isolerade målet; [`diff`](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html) jämför manifest. Absoluta sökvägar eller sökvägar med olika prefix normaliseras före jämförelsen.

## Delning till konton och via offentliga länkar

Proton Drive har minst två delningsmodeller:

- **Medlemskap/inbjudan:** En person blir medlem i en delning med rollen Viewer eller Editor. Delningens lösenfras paketeras för medlemmens adressnyckel.
- **Offentlig länk:** En länk ger åtkomst utan Proton-konto. Ett lösenord skyddar delningens lösenfras; länk och lösenord utgör tillsammans åtkomsthemligheten.

[Åtkomsthantering](https://proton.me/support/drive-manage-access-shared-files) dokumenterar roller, delegering till Editor, återkallande och separata inställningar för e-postinbjudan och Public Link. [Lösenord och utgångsdatum](https://proton.me/support/password-protect-files-proton-drive) begränsar länkåtkomst. Om ett slumpmässigt lösenord skickas med som URL-fragment ser webbläsare och mottagare den fullständiga länken; fragmentet skickas visserligen inte som HTTP-sökväg till servern, men kan läcka via urklipp, webbläsarhistorik, skärmbilder eller endpointtelemetri.

Administratörer inventerar ägare, delningsrot, medlemmar, roll, vidarebefordringsrätt, Public Link-status, utgångsdatum och senaste kontroll. En Editor kan beroende på inställning själv dela och ändra behörigheter. Offboarding återkallar därför inte bara kontot, utan granskar alla delningar och externa länkar.

## Versioner, papperskorg och säkerhetskopiering

Versionshistorik lagrar tidigare revisioner av samma filobjekt. En återställd revision raderar inte automatiskt nyare versioner. Papperskorgen behåller raderade noder till slutlig radering. Båda funktionerna är värdefulla, men ligger fortsatt inom samma administrativa och kryptografiska felområde.

De skyddar inte tillförlitligt mot:

- förlust av eller omöjlighet att återställa kontonyckeln,
- felaktiga behörigheter eller illvillig slutlig radering,
- kvot-, avtals- och leverantörshändelser,
- fel som uppstod före den första lyckade molnrevisionen,
- oupptäckta synkroniseringsutelämnanden,
- organisationsomfattande kompromettering av identitet eller endpoint.

En oberoende säkerhetskopia läser ut en definierad molnmängd med en auktoriserad klient, skriver den till ett separat administrerat mål, förser den med manifest, bevarande och oföränderlig historik samt utför återställningstester. NIST SP 800-34 behandlar säkerhetskopiering, alternativ lagring och återställningstester som sammanhängande kontroller för beredskapsplanering ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)). Grunder finns under [Backup och Disaster Recovery](/kb/backup-dr) och [Storage](/kb/storage).

## Identitet, session och endpointförtroende

End-to-end-kryptering flyttar förtroendet från leverantören till klient, konto och endpoint. Den som kontrollerar den upplåsta enheten, webbläsarsessionen eller kontot kan dekryptera och signera ändringar i användarens namn. Därför hör MFA, hårdvarusäkerhetsnycklar, enhetsskydd, sessionsåterkallande, klientursprung och skydd av lokal cache ihop.

[Hotmodellen](https://proton.me/blog/proton-drive-threat-model) nämner bland annat komprometterade enheter, nätfiske och officiella klienters äkthet som gränser. Öppen källkod möjliggör kodgranskning, men ersätter inte verifiering av den binärfil som faktiskt körs och dess uppdateringsväg.

Malwarekontroll är också begränsad: den valfria [upptäckten av skadliga filer](https://proton.me/support/proton-drive-malware-protection) jämför på begäran en hash med kända malwarehashar. Den dekrypterar inte hela innehållet på serversidan, identifierar inte okänt innehåll och ersätter inte en endpointsensor.

## Lösenordsåterställning och dataåterställning

Kontoåterställning och dataåterställning är olika processer. En lösenordsåterställning kan återställa åtkomsten till kontot, men göra tidigare privata nycklar otillgängliga till en början. Proton Drive visar då de gamla data som låsta.

Den officiella [Drive-dataåterställningen](https://proton.me/support/drive-data-recovery) anger tidigare lösenord, Recovery Phrase eller Recovery File som upplåsningsvägar. Nya uppladdningar efter återställningen kan skapas under nya nycklar medan äldre innehåll förblir låst; foton och album kan behöva ett ytterligare sammanfogningssteg.

Ett organisationsrunbook dokumenterar därför:

- vem som får initiera en lösenordsåterställning,
- vilka återställningsmetoder som är konfigurerade och testade i förväg,
- var Recovery Phrase eller File förvaras offline och åtkomstskyddat,
- hur gamla och nya nyckeltillstånd identifieras,
- hur delningar och medlemmar kontrolleras efter återställning,
- när en låst Drive under inga omständigheter får raderas.

”Unlock later” bevarar möjligheten till senare återställning. ”Delete locked drive” är en oåterkallelig dataradering och hör inte hemma i en obekräftad standardprocedur.

När identitet, nycklar och lokalt tillstånd är plausibla följer nätverksvägen. Proxy, TLS, API-slutpunkter och blockerad överföring måste granskas separat från den kryptografiska datamodellen.

## Nätverksväg och transportdiagnostik

HTTPS skyddar transporten mellan klienten och Proton-API:t respektive lagringsslutpunkterna. Det ersätter inte end-to-end-kryptering, men skyddar sessions- och driftmetadata, endpointidentitet och tillgänglighet. Proxy, DNS-filter, TLS-inspektion, VPN, MTU eller instabila anslutningar kan påverka kontrollplansförfrågningar och blocköverföringar olika.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) och [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) kontrollerar namnupplösning; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) TCP 443; [`curl`](https://curl.se/docs/manpage.html) HTTP och omdirigeringar; [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) respektive [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) aktiva anslutningar; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) TLS och SNI. Testet mot webbdomänen bevisar inte att varje API-/blockslutpunkt kan nås; klientloggarna visar den faktiskt felande mål- och förfrågningsklassen.

## Observerbarhet och diagnoskedja

En meningsfull diagnostik börjar med verksamhetsobjektet:

1. Bestäm konto, delning, logisk sökväg och berörd klient.
2. Skilj lokalt tillstånd: platshållare, hydrerad, fäst, ändrad, låst eller felaktig.
3. Kontrollera aktivitetsvy och loggar för nod-/revisions-/överföringsreferens.
4. Kontrollera ledigt lokalt utrymme och molnkvot.
5. Kontrollera namn- och sökvägskompatibilitet.
6. Kontrollera DNS, TCP, TLS och proxy/VPN mot den faktiska slutpunkten.
7. Hantera signatur-, dekrypterings- och nyckelvarningar separat från transportfel.
8. Lista eller ladda ned filen med den officiella CLI:n eller en andra klient.
9. Jämför nedladdningshashen och den förväntade revisionen.
10. Överväg först därefter paus/återupptagning, återautentisering, uppdatering eller cacheåtgärd.

[Windows-synkroniseringsfel](https://proton.me/support/proton-drive-windows-common-issues) nämner förlopp, öppna filer, ytterligare molnklienter, paus/återupptagning, namnregler och nätverk. Dessa användarhjälper kompletteras i administrativ drift med objektinventering och end-to-end-verifiering. Allmän metodik finns under [Troubleshooting](/kb/troubleshooting).

## Prestanda- och kapacitetsmodell

Kostnaderna uppstår inte enbart genom överförda byte. Många små filer skapar arbete per nod för nycklar, metadata, API och signaturer. Djupa träd kräver upprepad hierarkiupplösning. Ett stort filinnehåll kan däremot bearbetas effektivare i parallella block. Proton påpekar i Windows-felguiden att många små eller djupt kapslade filer kan synkroniseras långsammare än en enskild fil av samma storlek.

Kapacitetsplanering omfattar:

- molnkvot inklusive versioner och papperskorg,
- lokala offline- och cachetillstånd,
- temporärt utrymme för uppladdning, nedladdning och konfliktkopior,
- CPU för kryptografi och hashning,
- filantal och träddjup,
- händelseeftersläpning efter offlineperioder,
- Fair Use-/strypningsbeteende vid automatiserade överföringar.

CLI-introduktionen varnar för att upprepade gånger ladda upp oförändrade data helt och hänvisar till Fair Use ([Proton Drive CLI](https://proton.me/blog/proton-drive-cli)). Ett schemalagt jobb underhåller därför ett eget källmanifest eller kontrollerar måltillståndet i stället för att behandla varje körning som en fullständig uppladdning.

## Drift med Windows, macOS och Linux

Windows och macOS erbjuder filhanterarintegration och bakgrundssynkronisering. Mobilappar hanterar uppladdning, nedladdning, offlineobjekt och foton, men är inte en identisk ersättning för en filsystemtjänst på skrivbordet. Den officiella CLI:n gör fil- och delningsoperationer skriptbara i alla tre operativsystemen för skrivbordet.

I juni 2026 publicerade Proton CLI:n och beskrev en fullvärdig Linux-skrivbordsapplikation med bakgrundssynkronisering som ett separat, fortfarande pågående projekt ([CLI-meddelande](https://proton.me/blog/proton-drive-cli)). Denna historiska avgränsning är fortsatt viktig: även om plattformssortimentet utvecklas vidare likställs inte **CLI-operation**, **montering** och **dubbelriktad synkroniseringsmotor**. För respektive utrullning kontrolleras [sidan med release notes](https://proton.me/support/drive-release-notes) som en tidsberoende källa.

I serverjobb körs webbläsarautentisering kontrollerat och den resulterande sessionstoken lagras i operativsystemets hemlighetslager för ett dedikerat tjänstkonto. Ett headless-jobb får inget personligt hemkatalog från en administratör. Exitkod, JSON-resultat, källmanifest och målverifiering bevaras; hemligheter och delningslänkar tas bort ur loggar.

Versioner och papperskorg skyddar mot enskilda handhavandefel, men ersätter ingen oberoende återställning. Runbooken måste behandla konto, nyckelåtkomst, lokala data och verifierad export gemensamt.

## Runbook för säkerhetskopiering och återställning

Proton Drive kan vara mål för en säkerhetskopia, men ersätter inte en återställningsdesign. En robust runbook omfattar:

1. Definiera källmängd och Source of Truth.
2. Hydrera lokala data på begäran fullständigt före filbaserad säkerhetskopiering eller exportera direkt med CLI.
3. Skapa ett filmanifest med sökväg, storlek och hash.
4. Utför uppladdning med uttrycklig konflikt- och raderingsregel.
5. Arkivera CLI-exitkod och mållistning.
6. Utför stickprov eller fullständig nedladdning till en isolerad sökväg.
7. Jämför hashar och förväntade revisioner.
8. Dokumentera delningar, versionshantering, papperskorg och bevarande.
9. Testa Proton-konto- och dataåterställning separat.
10. Behåll en oberoende andra kopia utanför samma konto- och leverantörsdomän.

RPO avser den senast bevisligen committade och verifierade revisionen i Proton, inte tidpunkten då den lokala filen skrevs. RTO omfattar kontoåtkomst, nyckelupplåsning, listning, nedladdning, dekryptering, bandbredd och målåterställning.

## Teknisk historia

Proton utvecklade Drive som ett komplement till det krypterade Proton-ekosystemet. Säkerhetsmodellen som publicerades 2020 etablerade central lagring med klientsidig OpenPGP-nyckelhierarki, krypterade namn, nodträd, delningar, signerade block och länkd elning. Webb- och mobilklienter kom före skrivbordsklienterna för synkronisering; Version History infördes 2023 som en egen revisionsväg ([introduktion av versionshistorik](https://proton.me/blog/drive-version-history)).

Windows och macOS kompletterade bakgrundssynkronisering och filer på begäran med plattformsspecifik Explorer-/Finderintegration. De separata implementationerna ledde stegvis till det gemensamma Drive SDK. Proton beskrev i början av 2026 SDK-utrullningen över webb, Windows, macOS, Android och iOS ([SDK-utveckling](https://proton.me/blog/drive-sdk-january-2026)).

I juni 2026 följde två tekniska steg: en gemensam kryptografiomställning till v6 PKESK, v2 SEIPD och AES-GCM för nya filobjekt samt den SDK-baserade CLI:n för Windows, macOS och Linux. Därmed blev officiellt skriptbar åtkomst möjlig utan att återskapa den kryptografiska klientlogiken. Arkitekturen förblir dock klientcentrerad: servrar lagrar och samordnar, auktoriserade klienter dekrypterar, signerar och reconcilerar.

## Checklista för administratörer

För klientsidigt krypterad lagring måste datamodell, nycklar, synkronisering och oberoende säkerhetskopiering förstås tillsammans. Checklistan kondenserar dessa punkter för driftgodkännande.

- **Datamodell:** Håll isär volym, delning, nod, länk, revision och block.
- **Kryptografi:** Dokumentera adress-, delnings-, nod- och innehållsnyckelkontext samt paketschema.
- **Metadata:** Separera E2EE-fält och serverläsbara driftfält i hotmodellen.
- **Klienter:** Inventera plattform, teknik, SDK, filsystemintegration, binärsignatur och loggsökväg.
- **Synkronisering:** Övervaka lokalt tillstånd, händelsemarkör, konfliktkopior, namn och väntande överföringar.
- **CLI:** Behandla inte en explicit filoperation som en permanent synkroniserings- eller säkerhetskopieringsmotor.
- **Delning:** Kontrollera ägare, medlemmar, roller, delegering till Editor, Public Link, lösenord och utgångsdatum.
- **Återställning:** Testa kontolösenord, dataupplåsning, Recovery Phrase/File och delningsåtkomst separat.
- **Säkerhetskopiering:** Ha en oberoende kopia, manifest, versionshantering och återställningsnedladdning.
- **Godkännande:** Dekryptera filen på en andra klient eller via isolerad nedladdning och öppna den efter signatur- och hashkontroll.

## Källor

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [Proton Drive – Supportöversikt](https://proton.me/support/drive)
- [Proton Drive – CLI](https://proton.me/support/drive-cli)
- [Proton – central och decentraliserad lagring](https://proton.me/blog/centalized-vs-p2p-protondrive)
- [Proton Drive – säkerhetsmodell](https://proton.me/blog/protondrive-security)
- [Proton Drive – kryptografiuppdatering](https://proton.me/blog/drive-cryptography-update)
- [Proton Drive – versionshantering](https://proton.me/support/version-history)
- [Proton Drive – radera och återställa synkroniserade filer](https://proton.me/support/proton-drive-delete-restore-synced-files)
- [RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [Proton Drive – signaturhantering](https://proton.me/support/drive-signature-management)
- [Proton Drive – webbnedladdning och streaming](https://proton.me/blog/proton-drive-web-encryption-technique)
- [Proton Drive – SDK Repository](https://github.com/ProtonDriveApps/sdk)
- [npm – @protontech/drive-sdk](https://www.npmjs.com/package/@protontech/drive-sdk)
- [GitHub – ProtonDriveApps](https://github.com/ProtonDriveApps)
- [Windows](https://github.com/ProtonDriveApps/windows-drive)
- [macOS](https://github.com/ProtonDriveApps/mac-drive)
- [Proton Drive – Windows On-Demand Sync](https://proton.me/support/proton-drive-windows-on-demand-sync)
- [Proton Drive – macOS-guide](https://proton.me/support/drive-macos-guide)
- [Proton Drive – vanliga Windows-synkroniseringsproblem](https://proton.me/support/proton-drive-windows-common-issues)
- [Proton Drive – Shared with me i Windows](https://proton.me/support/shared-with-me-windows)
- [Microsoft – Get-Process](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html)
- [Proton Drive – Logs](https://proton.me/support/proton-drive-logs)
- [Proton – introduktion till Drive CLI](https://proton.me/blog/proton-drive-cli)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [jq – handbok](https://jqlang.org/manual/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [Microsoft – New-Item](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)
- [GNU Coreutils – mkdir](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html)
- [GNU Diffutils – diff](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html)
- [Proton Drive – åtkomsthantering](https://proton.me/support/drive-manage-access-shared-files)
- [Proton Drive – lösenordsskydd och utgångsdatum](https://proton.me/support/password-protect-files-proton-drive)
- [Proton Drive – hotmodell](https://proton.me/blog/proton-drive-threat-model)
- [Proton Drive – upptäckt av skadliga filer](https://proton.me/support/proton-drive-malware-protection)
- [Proton Drive – dataåterställning](https://proton.me/support/drive-data-recovery)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [curl – handbok](https://curl.se/docs/manpage.html)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Proton Drive – Release Notes](https://proton.me/support/drive-release-notes)
- [Proton Drive – introduktion av versionshistorik](https://proton.me/blog/drive-version-history)
- [Proton Drive – SDK-utveckling 2026](https://proton.me/blog/drive-sdk-january-2026)
