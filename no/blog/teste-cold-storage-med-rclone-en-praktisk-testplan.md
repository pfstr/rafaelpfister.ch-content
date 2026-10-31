---
title: "Teste Cold Storage med Rclone: en praktisk testplan"
navTitle: "Test Rclone"
description: "Før en tjeneste leser filene sine fra skyen via en Rclone-mount, bør du kontrollere mer enn bare katalogtilgang. Denne testplanen dekker kalde lesinger, varme lesinger, skriveoperasjoner, cache-atferd, filintegritet og feil."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min lesetid"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
slug: "teste-cold-storage-med-rclone-en-praktisk-testplan"
translationOf: "cloud-mount-testen-dummy-pdfs"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:29:04.369Z
translationReview: required
translationSourceHash: 820965b014837c0d0b5b5bf629a02842f2ebbbb26f62d343d8f1630278cd3bec
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/teste-cold-storage-med-rclone-en-praktisk-testplan
---

En Rclone-mount er rask å sette opp. Remote-en vises som en katalog, `ls` viser filer, og den første funksjonstesten er bestått. For produksjonsdrift sier det likevel lite.

Så snart en tjeneste får tilgang til mounten, oppstår flere spørsmål: Hvor lang tid tar første tilgang til en fil? Hvilke tilganger håndteres av den lokale cachen? Hva skjer med en fil som ennå ikke er lastet opp hvis Rclone krasjer? Ser en container som kjører mounten igjen når den er bygget opp på nytt? Og hvordan reagerer tjenesten når skyen midlertidig ikke er tilgjengelig?

Denne artikkelen gir en generell testplan for dette. Du kan bruke den til et dokumentarkiv, en medieserver, en bildeadministrasjon eller enhver annen tjeneste som henter sjelden brukte filer via Rclone fra Cold Storage.

## De viktigste alternativene i rclone

Først, for orientering, Rclone-alternativene som forekommer i denne testplanen, fritt oversatt fra dokumentasjonen:

<details class="options-details">
<summary>Oversikt over alternativer</summary>

| Alternativ | Betydning |
|---|---|
| `--seed n` | Startverdi for tilfeldighetsgeneratoren ved `rclone test makefiles`; samme seed gir samme filtre |
| `--files n` | Antall testfiler som skal opprettes |
| `--files-per-directory n` | Gjennomsnittlig antall filer per katalog |
| `--min-file-size grösse` | Minste genererte filstørrelse (suffikser som K, M, G er tillatt) |
| `--max-file-size grösse` | Største genererte filstørrelse |
| `--progress` | Løpende fremdriftsvisning under overføringen |
| `--stats dauer` | Intervall for utskrift av overføringsstatistikk |
| `--log-file datei` | Skriver loggen til den angitte filen |
| `--log-level stufe` | Detaljnivå for loggen: DEBUG, INFO, NOTICE eller ERROR |
| `--one-way` | Kontrollerer ved `rclone check` bare at alle kildefiler finnes i målet og er identiske; ekstra filer i målet regnes ikke som feil |
| `--download` | Laster faktisk ned dataene ved sammenligning, i stedet for bare å sammenligne hasher |
| `--vfs-cache-mode modus` | Cache-strategi for VFS-laget; `full` mellomlagrer lese- og skrivetilganger lokalt |
| `--cache-dir verzeichnis` | Katalog for lokal cache |
| `--vfs-cache-max-size grösse` | Myk grense for total størrelse på VFS-cachen |
| `--vfs-cache-poll-interval dauer` | Intervall der Rclone kontrollerer og rydder cachen |
| `--vfs-write-back dauer` | Ventetid etter at en fil lukkes før opplasting til remote starter |
| `--vfs-read-ahead grösse` | Ekstra datamengde som forhåndsleses ved `full` utover den forespurte posisjonen |
| `--poll-interval dauer` | Intervall der Rclone spør remote-en om endringer (bare for backender med støtte for polling) |
| `--dir-cache-time dauer` | Gyldighetstid for mellomlagrede kataloglister |
| `--allow-other` | Tillater andre brukere enn den som monterer, å få tilgang til FUSE-mounten |

</details>

De komplette listene finnes i Rclone-dokumentasjonen, særlig under [rclone mount](https://rclone.org/commands/rclone_mount/) og i oversikten over [globale flagg](https://rclone.org/flags/).

## Definer først hva du vil oppnå

Cold Storage betyr ikke automatisk det samme for alle applikasjoner. En medieserver leser som regel store filer sekvensielt. En bildeadministrasjon laster mange små forhåndsvisningsdata og hopper til ulike posisjoner. Et dokumentarkiv åpner relativt små filer, men ofte bare én gang.

Noter de viktigste egenskapene ved din reelle bestand før testen:

- typisk filstørrelse samt største forekommende fil
- antall filer per katalog
- fullstendig lesing eller tilfeldige tilganger til enkeltområder
- forholdet mellom lese- og skrivetilganger
- antall samtidige brukere eller prosesser
- endringer som skjer direkte i remote-en utenfor mounten
- akseptabel ventetid for en kald lesing
- maksimalt tilgjengelig plass for lokal cache

Først da får du meningsfulle suksesskriterier. Å åpne én enkelt fil på 1,2 sekunder kan være helt greit for et arkiv og ubrukelig for en interaktiv applikasjon.

## Opprett en reproduserbar testbestand

Rclone har allerede et passende verktøy for dette. `rclone test makefiles` oppretter samme filtre hver gang med en fast seed:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Målkatalog der testtreet opprettes |
| `--seed 42` | Fast startverdi for tilfeldighetsgeneratoren; hver kjøring oppretter samme bestand |
| `--files 250` | Totalt 250 testfiler |
| `--files-per-directory 25` | I gjennomsnitt 25 filer per katalog |
| `--min-file-size 16K` | Minste fil på 16 KiB |
| `--max-file-size 32M` | Største fil på 32 MiB |

</details>

Tilpass antall og størrelser til din reelle bestand. Ikke test bare gjennomsnittsfiler. Noen svært små filer viser hvor kostbare metadata-tilganger er; noen store filer synliggjør gjennomstrømning, Read-Ahead og cache-atferd.

Legg også til filnavn som kan skape problemer i praksis:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `mkdir -p` | Oppretter også manglende overordnede kataloger og rapporterer ikke feil dersom katalogen finnes |
| `printf '…\n' > datei` | Skriver den angitte teksten som filinnhold; omdirigeringen oppretter filen med det problematiske navnet |

</details>

Den siste testen er særlig viktig dersom lokalt filsystem og skybackend behandler store og små bokstaver ulikt.

Hvis tjenesten din bare godtar bestemte formater, er ikke vilkårlige binærfiler nok. Opprett da også syntetiske filer i nettopp disse formatene. For Paperless-ngx var dette PDF-er med ekte tekstlag, slik at testen ikke utilsiktet måler OCR-ytelse i stedet for lagringsbanen. For en bildeadministrasjon bør bestanden inneholde ulike bildestørrelser og formater, og for en medieserver korte filer med ulike kodeker.

## En referansemåling uten mount

Før FUSE og VFS-cache kommer inn i bildet, bør du måle backenden direkte. Kopier bestanden til test-remote-en med Rclone og lagre en detaljert logg:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Kilde for kopien: den lokalt opprettede testbestanden |
| `remote:cold-storage-test` | Mål: sti i konfigurert remote |
| `--progress` | Løpende fremdriftsvisning i terminalen |
| `--stats 5s` | Overføringsstatistikk hvert 5. sekund i stedet for med standardintervallet |
| `--log-file rclone-copy.log` | Fullstendig logg i en fil for senere analyse |
| `--log-level INFO` | Logger overføringer, nye forsøk og feil, uten DEBUG-omfanget |

</details>

Kontroller deretter om kilde og mål stemmer overens:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Referanse: den lokale originalbestanden |
| `remote:cold-storage-test` | Testobjekt: den nettopp opplastede bestanden i remote-en |
| `--one-way` | Kontrollerer bare at alle kildefiler finnes i målet og er identiske; ekstra filer i målet påpekes ikke |
| `--download` | Laster ned dataene og sammenligner innholdet, i stedet for å stole på hasher |

</details>

`--download` er avgjørende her, fordi enkelte backender ikke tilbyr passende hasher. Sammenligningen tar lengre tid, men gir et brukbart utgangspunkt for den senere integritetstesten.

Noter opplastingstid, overføringshastighet, antall nye forsøk og API-feil. Hvis direkte tilgang allerede er ustabil, kan mounten ikke reparere det.

## Skill test-mounten fra produksjonscachen

Bruk et eget mountpunkt og en egen cache-katalog for målingen:

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `remote:cold-storage-test` | Remote-en med sti som skal monteres |
| `/mnt/rclone-test` | Mountpunkt på testsystemet |
| `--vfs-cache-mode full` | Mellomlagrer lese- og skrivetilganger fullstendig i lokal cache |
| `--cache-dir /var/cache/rclone-test` | Egen cache-katalog, skilt fra produksjonscachen |
| `--vfs-cache-max-size 10G` | Myk grense på 10 GiB for VFS-cachen |
| `--vfs-cache-poll-interval 1m` | Cache-kontroll og opprydding hvert minutt |
| `--allow-other` | Også andre brukere enn den som monterer, får tilgang; nødvendig for tjenester og containere |
| `--log-file /var/log/rclone-test.log` | Logg i en fil for å spore tilganger under testene |
| `--log-level INFO` | Middels detaljnivå i loggen |

</details>

Verdiene er et eksempel og ingen generell anbefaling. Det avgjørende er separasjonen: En tom test-cache gjør kalde lesinger reproduserbare, uten at du må slette filer fra en aktiv produksjonscache.

`--vfs-cache-mode full` er vanligvis den mest opplysende testmodusen for applikasjoner. Rclone mellomlagrer lese- og skrivetilganger lokalt og kan bedre gjengi filtilganger som ikke ville vært mulig med ren objektlagring. Den ekstra kompatibiliteten koster lokal lagringsplass.

## Kontroller alltid fra den faktiske tjenestens perspektiv

En mount kan fungere for brukeren din og likevel være ubrukelig for tjenesten. Vanlige årsaker er en annen bruker-ID, manglende `--allow-other`, containergrenser eller feil mount-propagation.

Utfør derfor minst én fullstendig lesetilgang med samme identitet som applikasjonen senere skal kjøre under:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-u <service-user>` | Kjører kommandoen som den angitte brukeren, ikke som root |
| `/mnt/rclone-test/pfad/zur/datei` | Fil som skal leses; `sha256sum` tvinger fram fullstendig lesetilgang |

</details>

Kjører tjenesten i Docker, hører testen hjemme i containeren:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `--user <uid>:<gid>` | Kjører kommandoen i containeren med denne bruker- og gruppe-ID-en, uavhengig av imagets standardbruker |
| `<app-container>` | Navn eller ID på den kjørende applikasjonscontaineren |
| `sha256sum /pfad/im/container/datei` | Kommando som skal kjøres; stien er mounten sett fra containeren |

</details>

Enda bedre er en reell applikasjonstest. Åpne filen via tjenestens webgrensesnitt eller API. Bare slik oppdager du om applikasjonen for eksempel starter flere parallelle lesinger, hopper til slutten av filen eller forventer ytterligere metadata.

## Mål kalde lesinger og varme lesinger separat

Ved `--vfs-cache-mode full` finnes det tre lag mellom applikasjonen og skyen:

| Lag | Hva som ligger der |
|---|---|
| Remote | hele filen i skytjenesten |
| VFS-cache | lokalt lagrede områder av filer som allerede er lest |
| Linux-sidecache | nylig brukte data i RAM |

For en kald lesing velger du en fil hvis innhold aldri tidligere er lest via test-mounten. Ved den direkte påfølgende varme lesingen ligger den i VFS-cachen og som regel også i RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `date +%s%3N` | Tidsstempel i millisekunder: Unix-sekunder, direkte etterfulgt av de tre første sifrene i nanosekundandelen (GNU date) |
| `cat "$file" > /dev/null` | Leser filen fullstendig og forkaster utdataene; bare lesetiden måles |
| `"$1"`, `"$2"` | Argumenter til shell-funksjonen: filsti og etikett for målelinjen |

</details>

Ikke mål bare én fil. Bruk minst ti tidligere uleste filer i ulike størrelser, og noter median, tregeste verdi og filstørrelse. Én enkelt beste verdi er ikke et grunnlag for beslutning.

En varm lesing er ikke en ren disktest, fordi kjernen kan holde deler av filen i RAM. For de fleste Cold Storage-scenarier er dette ikke et problem. Det avgjørende er hva brukeren opplever ved første og gjentatt åpning. Hvis du vil vurdere RAM og lokal disk separat, må du i tillegg kontrollere og dokumenterbart tømme sidecachen.

## Test ikke bare fullstendige lesetilganger

`cat` leser en fil fra begynnelse til slutt. Mange applikasjoner oppfører seg annerledes:

- En videospiller leser først header og indeks, hopper senere til en annen posisjon og fortsetter deretter sekvensielt.
- En bildeadministrasjon leser metadata og oppretter deretter et forhåndsvisningsbilde.
- Et arkivprogram kan først lese slutten av filen.
- Flere arbeidere kan få tilgang til ulike filer samtidig.

Test disse forløpene med den faktiske applikasjonen. Observer Rclone-loggen og cachen parallelt. For store filer er det interessant hvor mye Rclone faktisk lagrer lokalt, og om `--vfs-read-ahead` passer til tilgangsmønsteret.

En Rclone-mount er dessuten ikke et fornuftig lagringssted for databaser eller andre filer som trenger pålitelig låsing og hyppige endringer i samme fil. VFS-laget utjevner forskjeller mellom filsystem og objektlagring, men gjør ikke backenden om til et lokalt filsystem.

## Godkjenn skrivebanen separat

Hvis tjenesten din bare leser, monter remote-en skrivebeskyttet der det er mulig. Må den skrive, test oppretting, overskriving, omdøping og sletting hver for seg.

En skrevet fil vises ikke nødvendigvis umiddelbart i remote-en. Med aktiv VFS-cache starter opplastingen først etter at filen er lukket og `--vfs-write-back` har utløpt. Kontroller derfor begge tilstandene:

1. Applikasjonen har lukket filen uten feil.
2. Filen kan deretter leses i remote-en via direkte Rclone-tilgang.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Etter at --vfs-write-back har utløpt:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Målfiler i mounten; omdirigeringen skriver via VFS-cachen |
| `remote:cold-storage-test/writeback-test.txt` | Direkte tilgang utenom mounten: `rclone cat` leser filen fra remote-en og skriver den til stdout |

</details>

Gjenta testen med en stor fil og stopp Rclone mens opplastingen fortsatt pågår. Start deretter på nytt med samme cache-katalog og kontroller om opplastingen fortsetter. Nettopp dette tidsvinduet avgjør hvor mye data som er utsatt ved serverfeil.

Test også omdøping og sletting. Mange skybackender håndterer disse operasjonene annerledes enn et lokalt filsystem. Det relevante er ikke bare om kommandoen avsluttes uten feil, men når endringen blir synlig ved direkte tilgang til remote-en og for andre klienter.

## Test endringer utenfor mounten

Filer kan endres via leverandørens webgrensesnitt, en annen Rclone-prosess eller en annen server. Mounten ser ikke alltid slike endringer umiddelbart fordi kataloginformasjon mellomlagres.

Opprett derfor en fil direkte i remote-en med et nytt Rclone-kall:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `external-change.txt` | Kilde: den lokalt opprettede filen |
| `remote:cold-storage-test/external-change.txt` | Mål med nøyaktig filnavn; `copyto` kopierer én enkelt fil under nøyaktig dette navnet, i stedet for som `copy` til en katalog |

</details>

Mål når filen vises i mounten. Gjenta testen for endring og sletting. Resultatet avhenger av backenden, dens polling-støtte samt `--poll-interval` og `--dir-cache-time`. Hvis applikasjonen må se aktuelle endringer umiddelbart, må denne atferden uttrykkelig inngå i godkjenningskriteriene.

Med aktivert Remote Control-grensesnitt kan du forkaste katalogcachen målrettet:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `vfs/forget` | Remote Control-kommando som skal kjøres: forkaster det mellomlagrede katalogtreet i VFS, slik at neste tilgang spør remote-en på nytt |

</details>

Dette er nyttig for en manuell test, men erstatter ikke en passende driftsstrategi.

## Sett cachen under press

En nesten tom cache er det enkleste tilfellet. Sett `--vfs-cache-max-size` bevisst lavt i en andre testrunde, og les mer data enn den rommer.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `du -s` | Sammenfatter plassforbruket i én totalrad, i stedet for å liste hver underkatalog |
| `du -h` | Utdata i lesbare enheter (K, M, G) |
| `du --apparent-size` | Viser logisk filstørrelse i stedet for faktisk brukt lagringsplass |
| `find … -type f` | Tar bare hensyn til vanlige filer, ikke kataloger |
| `wc -l` | Teller linjene i utdataene, altså antall cache-filer her |

</details>

De to størrelsene kan avvike sterkt fra hverandre. I Full-modus bruker Rclone sparse files: En fil viser full logisk størrelse selv om bare de leste områdene opptar lokal plass.

Cache-grensen er dessuten myk. Rclone kontrollerer den i intervallet for `--vfs-cache-poll-interval`, og åpne filer kan ikke fjernes. Cachen kan derfor kortvarig overskride grensen. Etter at filene er lukket og neste opprydding er utført, bør den imidlertid synke igjen.

Loggfør toppverdien, verdien etter opprydding og tiden dette tar. Slik kan nødvendig lokal lagringsplass dimensjoneres fornuftig.

## Simuler to ulike feil

En utilgjengelig sky og en krasjet Rclone-prosess er to ulike feil:

| Feil | Hva du kontrollerer med den |
|---|---|
| Backend eller nettverk utilgjengelig, Rclone fortsetter å kjøre | Atferd ved nye forsøk, tidsavbrudd og allerede mellomlagrede filer |
| Rclone-prosess avsluttet | Atferd for FUSE-mounten og gjenoppretting av mountpunktet |

Simuler begge bare i testmiljøet. Du kan hardt avslutte en Rclone-container for det andre tilfellet:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `--signal KILL` | Sender SIGKILL (ved `docker kill` er dette allerede standarden, angitt her for tydelighetens skyld); prosessen får ingen mulighet til å rydde opp |
| `<rclone-container>` | Navn eller ID på Rclone-containeren |

</details>

Kontroller applikasjonen under feilen, ikke bare mountpunktet:

- Hvilke funksjoner er fortsatt tilgjengelige?
- Hvor lenge venter en tilgang før en feil vises?
- Er allerede fullstendig mellomlagrede filer fortsatt tilgjengelige?
- Stopper applikasjonen nye skriveoperasjoner?
- Oppstår det en forståelig feilmelding eller bare en hengende prosess?
- Utløses overvåkingen?

En skrivetjeneste må ikke skrive umerket til den underliggende lokale katalogen når mounten mangler. Etter at mounten kommer tilbake, vil disse filene bli skjult. En enkel beskyttelse før hver skrivejobb er:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-q` | Ingen utdata; resultatet finnes bare i exit-koden |
| `/mnt/rclone-test` | Sti som skal kontrolleres; exit-kode 0 bare dersom en mount faktisk er aktiv der |
| `\|\| exit 1` | Avbryter skriptet dersom stien ikke er et mountpunkt |

</details>

Etter at Rclone er startet på nytt, kontrollerer du mounten på verten og fra hver container som bruker den. En mount som er bygget opp på nytt, når en allerede kjørende container bare med riktig mount-propagation. For Docker er vanligvis `rslave` nødvendig på den konsumerende siden. Detaljene finnes i artikkelen [Drift av Rclone-mounter i Docker pålitelig](/blog/rclone-mount-in-docker-container).

## Et konkret eksempel fra Paperless-ngx

Til Paperless-testen min opprettet jeg 40 PDF-er med totalt 13,9 MB. Et dokument som ikke tidligere var åpnet, tok rundt 1,8 sekunder, mens en direkte gjentatt tilgang tok 19 til 24 millisekunder. En VFS-cache begrenset til 4 MB steg kortvarig til 12,7 MiB og ble ryddet igjen ved neste kjøring.

Mens remote-en ikke var tilgjengelig, fungerte dokumentliste, fulltekstsøk og forhåndsvisningsbilder fortsatt, fordi disse dataene lå lokalt. Bare originalen kunne ikke åpnes. Etter at mounten var bygget opp igjen, kunne Paperless-containeren som fortsatt kjørte, igjen få tilgang til filene uten å bli startet på nytt selv.

Disse tallene er ingen benchmark for Rclone eller Proton Drive. Atferden er interessant: Hot Storage forble tilgjengelig lokalt, kalde lesinger var tregere, men forutsigbare, og tjenesten kom seg etter feilen.

## Hva testprotokollen bør inneholde

Et resultat som senere kan etterprøves, inneholder minst:

- Rclone-versjon og brukt backend
- operativsystem, FUSE-variant og filsystemet for cache-katalogen
- fullstendig mount-kommando uten påloggingsopplysninger
- antall, størrelsesfordeling og struktur for testfilene
- verdier for kald og varm lesing av flere filer
- skrivetid frem til synlighet i remote-en
- cache-toppverdi og varighet for opprydding
- resultat av `rclone check --download`
- atferd ved backend-feil og avsluttet Rclone-prosess
- gjenopprettingstid sett fra applikasjonen
- nye forsøk, tidsavbrudd, begrensninger og autentiseringsfeil fra loggen

Definer en grenseverdi på forhånd for hvert punkt. Da avsluttes testen med en beslutning, ikke bare med en samling interessante tall.

## Når oppsettet er klart

En Cold Storage-mount er klar til bruk når du kan svare ja på disse spørsmålene:

- Er kalde lesinger raske nok for den tiltenkte tjenesten?
- Akselererer cachen gjentatte tilganger som forventet?
- Forblir lokalt plassbehov kontrollerbart også under belastning?
- Stemmer alle filer etter en fullstendig nedlasting?
- Fungerer alle nødvendige filoperasjoner med valgt backend?
- Oppfører applikasjonen seg kontrollert ved skyfeil?
- Stoppes skriveoperasjoner sikkert når mounten mangler?
- Når en mount bygges opp på nytt, når den alle kjørende forbrukere?
- Viser overvåkingen feilen før en bruker rapporterer den?

Hvis et svar mangler, vet du i det minste nøyaktig hva du må jobbe videre med. Det er langt mer nyttig enn en mount som så bra ut ved første `ls`, men først viser begrensningene sine i drift.

## Endringslogg

- **25. oktober 2026:** Korrigert: Ved docker kill er SIGKILL allerede standardsignalet; alternativet --signal KILL brukes bare for tydelighetens skyld.

## Kilder

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproduserbare testfiler og katalogstrukturer med konfigurerbare størrelser.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachemoduser, writeback, sparse files, cache-grenser og katalogcache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): sammenligning av kilde og mål, inkludert fullstendig kontroll med `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): målrettet forkasting av VFS-katalogcachen med `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): fullstendig referanse for globale alternativer, blant annet logging, statistikk og VFS-parametrene.
