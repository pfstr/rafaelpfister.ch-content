---
title: "Teste Cold Storage med Rclone: en praktisk testplan"
navTitle: "Teste Rclone"
description: "Før en tjeneste leser filene sine fra skyen via en Rclone-mount, bør du kontrollere mer enn bare katalogtilgang. Denne testplanen dekker kalde lesinger, varme lesinger, skriveoperasjoner, cacheatferd, filintegritet og feil."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min lesetid"
themen:
  - "rclone"
related:
  - "rclone-mount-in-docker-container"
  - "paperless-dokumente-clouddienst-auslagern"
slug: "teste-cold-storage-med-rclone-en-praktisk-testplan"
translationOf: "cloud-mount-testen-dummy-pdfs"
url: "https://rafaelpfister.ch/no/blog/teste-cold-storage-med-rclone-en-praktisk-testplan"
---

En Rclone-mount er rask å sette opp. Remote-en vises som en katalog, `ls` viser filer, og den første funksjonstesten er bestått. Dette sier likevel lite om produksjonsdrift.

Så snart en tjeneste får tilgang til mounten, dukker det opp flere spørsmål: Hvor lang tid tar første tilgang til en fil? Hvilke tilganger håndteres av den lokale cachen? Hva skjer med en fil som ennå ikke er lastet opp, dersom Rclone krasjer? Ser en kjørende container mounten på nytt når den bygges opp igjen? Og hvordan reagerer tjenesten dersom skyen midlertidig ikke er tilgjengelig?

Denne artikkelen gir en generell testplan for dette. Du kan bruke den for et dokumentarkiv, en medieserver, en bildeadministrasjon eller enhver annen tjeneste som henter sjelden brukte filer via Rclone fra Cold Storage.

## Fastsett først hva du vil oppnå

Cold Storage betyr ikke automatisk det samme for alle applikasjoner. En medieserver leser som regel store filer sekvensielt. En bildeadministrasjon laster mange små forhåndsvisningsdata og hopper til ulike posisjoner. Et dokumentarkiv åpner forholdsvis små filer, men ofte bare én gang.

Noter de viktigste egenskapene ved ditt faktiske datasett før testen:

- typisk filstørrelse samt største forekommende fil
- antall filer per katalog
- fullstendig lesing eller tilfeldige tilganger til enkelte områder
- forholdet mellom lese- og skrivetilganger
- antall samtidige brukere eller prosesser
- endringer som skjer direkte i remote-en utenfor mounten
- akseptabel ventetid for en kald lesing
- maksimalt tilgjengelig plass for den lokale cachen

Først da får du meningsfulle suksesskriterier. Å åpne én enkelt fil på 1,2 sekunder kan være helt greit for et arkiv og ubrukelig for en interaktiv applikasjon.

## Opprett et reproduserbart testdatasett

Rclone har allerede et egnet verktøy for dette. `rclone test makefiles` oppretter det samme filtreet hver gang med en fast seed:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Tilpass antall og størrelser til ditt faktiske datasett. Ikke test bare gjennomsnittsfiler. Noen svært små filer viser hvor kostbare metadata-tilganger er; noen store filer synliggjør gjennomstrømning, Read-Ahead og cacheatferd.

Legg også til filnavn som kan skape problemer i praksis:

```bash
mkdir -p "testdata/Spesialtilfeller/Underkatalog"
printf 'Mellomrom\n' > "testdata/Spesialtilfeller/Fil med mellomrom.txt"
printf 'Spesialtegn\n' > "testdata/Spesialtilfeller/Størrelse og endring.txt"
printf 'Store bokstaver\n' > "testdata/Spesialtilfeller/Test.txt"
printf 'Små bokstaver\n' > "testdata/Spesialtilfeller/test.txt"
```

Den siste testen er særlig viktig dersom lokalt filsystem og skybackend behandler store og små bokstaver ulikt.

Dersom tjenesten din bare godtar bestemte formater, holder det ikke med vilkårlige binærfiler. Opprett da også syntetiske filer i nettopp disse formatene. For Paperless-ngx var dette PDF-er med et ekte tekstlag, slik at testen ikke utilsiktet måler OCR-ytelsen i stedet for lagringsstien. En bildeadministrasjon bør ha ulike bildestørrelser og formater i datasettet, mens en medieserver bør ha korte filer med forskjellige kodeker.

## En referansemåling uten mount

Før FUSE og VFS-cache kommer inn i bildet, bør du måle backend-en direkte. Kopier datasettet med Rclone til test-remote-en og lagre en detaljert logg:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Kontroller deretter at kilde og mål stemmer overens:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

Med `--download` leser Rclone faktisk dataene og sammenligner dem, selv om backend-en ikke tilbyr passende hasher. Dette tar lengre tid, men gir et brukbart utgangspunkt for den senere integritetstesten.

Registrer opplastingstid, overføringshastighet, antall forsøk på nytt og API-feil. Dersom den direkte tilgangen allerede er ustabil, kan mounten ikke reparere dette.

## Skill test-mounten fra produksjonscachen

Bruk et eget mountpunkt og en egen cachekatalog for målingen:

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

Verdiene er et eksempel og ikke en generell anbefaling. Det avgjørende er separasjonen: En tom testcache gjør kalde lesinger reproduserbare uten at du må slette filer fra en aktiv produksjonscache.

`--vfs-cache-mode full` er vanligvis den mest informative testmodusen for applikasjoner. Rclone mellomlagrer lese- og skrivetilganger lokalt og kan bedre gjengi filtilganger som ikke ville vært mulig med ren objektlagring. Den ekstra kompatibiliteten krever lokal lagringsplass.

## Kontroller alltid fra den faktiske tjenestens perspektiv

En mount kan fungere for brukeren din, men likevel være ubrukelig for tjenesten. Vanlige årsaker er en annen bruker-ID, manglende `--allow-other`, containergrenser eller feil mount-propagation.

Utfør derfor minst én fullstendig lesetilgang med samme identitet som applikasjonen senere skal kjøre under:

```bash
sudo -u <tjenestebruker> sha256sum /mnt/rclone-test/sti/til/fil
```

Kjører tjenesten i Docker, skal testen utføres i containeren:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /sti/i/container/fil
```

Enda bedre er en reell applikasjonstest. Åpne filen via tjenestens webgrensesnitt eller API. Bare slik oppdager du om applikasjonen for eksempel starter flere parallelle lesinger, hopper til slutten av filen eller forventer ekstra metadata.

## Mål kalde og varme lesinger separat

Med `--vfs-cache-mode full` finnes det tre nivåer mellom applikasjonen og skyen:

| Nivå | Hva som ligger der |
|---|---|
| Remote | hele filen i skytjenesten |
| VFS-cache | lokalt lagrede områder av allerede leste filer |
| Linux-sidecache | nylig brukte data i RAM |

For en kald lesing velger du en fil hvis innhold aldri tidligere har blitt lest via test-mounten. Ved den direkte påfølgende varme lesingen ligger filen i VFS-cachen og som regel også i RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/stor-fil.bin" "Kald lesing"
measure_read "/mnt/rclone-test/stor-fil.bin" "Varm lesing"
```

Ikke mål bare én fil. Bruk minst ti hittil uleste filer med forskjellig størrelse, og noter median, tregeste verdi og filstørrelse. Én enkelt toppverdi er ikke et beslutningsgrunnlag.

En varm lesing er ikke en ren disktest, fordi kjernen kan beholde deler av filen i RAM. For de fleste Cold Storage-scenarioer er dette ikke et problem. Det avgjørende er hva en bruker opplever ved første og gjentatt åpning. Dersom du vil vurdere RAM og lokal disk separat, må du i tillegg kontrollere og dokumenterbart tømme sidecachen.

## Ikke test bare fullstendige lesetilganger

`cat` leser en fil fra begynnelse til slutt. Mange applikasjoner oppfører seg annerledes:

- En videospiller leser først headere og indeks, hopper senere til en annen posisjon og fortsetter deretter sekvensielt.
- En bildeadministrasjon leser metadata og oppretter deretter et forhåndsvisningsbilde.
- Et arkivprogram kan først lese slutten av filen.
- Flere workere kan få tilgang til forskjellige filer samtidig.

Test disse arbeidsflytene med den faktiske applikasjonen. Observer Rclone-loggen og cachen parallelt. For store filer er det interessant hvor mye Rclone faktisk lagrer lokalt, og om `--vfs-read-ahead` passer til tilgangsmønsteret.

En Rclone-mount er dessuten ikke et fornuftig lagringssted for databaser eller andre filer som krever pålitelig låsing og hyppige endringer i samme fil. VFS-laget utjevner forskjeller mellom filsystem og objektlagring, men gjør ikke backend-en om til et lokalt filsystem.

## Test skrivebanen separat

Dersom tjenesten din bare leser, bør du montere remote-en skrivebeskyttet når det er mulig. Må den skrive, test oppretting, overskriving, omdøping og sletting enkeltvis.

En skrevet fil vises ikke nødvendigvis umiddelbart i remote-en. Med aktiv VFS-cache starter opplastingen først etter at filen er lukket og `--vfs-write-back` har utløpt. Kontroller derfor begge tilstander:

1. Applikasjonen har lukket filen uten feil.
2. Filen kan deretter leses i remote-en via direkte Rclone-tilgang.

```bash
printf 'tilbakeskriving-test\n' > /mnt/rclone-test/tilbakeskriving-test.txt

# Etter at --vfs-write-back har utløpt:
rclone cat remote:cold-storage-test/tilbakeskriving-test.txt
```

Gjenta testen med en stor fil og avslutt Rclone mens opplastingen fortsatt pågår. Start deretter på nytt med samme cachekatalog og kontroller om opplastingen fortsetter. Nettopp dette tidsvinduet avgjør hvor mye data som er utsatt ved serverutfall.

Test også omdøping og sletting. Mange skybackender representerer disse operasjonene annerledes enn et lokalt filsystem. Det relevante er ikke bare om kommandoen avsluttes vellykket, men når endringen blir synlig ved direkte tilgang til remote-en og for andre klienter.

## Test endringer utenfor mounten

Filer kan endres via leverandørens webgrensesnitt, en annen Rclone-prosess eller en annen server. Mounten ser ikke alltid slike endringer umiddelbart, fordi kataloginformasjon mellomlagres.

Opprett derfor en fil direkte i remote-en med et andre Rclone-kall:

```bash
printf 'ekstern-endring\n' > ekstern-endring.txt
rclone copyto ekstern-endring.txt \
  remote:cold-storage-test/ekstern-endring.txt
```

Mål når filen vises i mounten. Gjenta testen for endring og sletting. Resultatet avhenger av backend-en, dens støtte for polling samt `--poll-interval` og `--dir-cache-time`. Dersom applikasjonen må se aktuelle endringer umiddelbart, må denne atferden uttrykkelig inngå i akseptansekriteriene.

Med aktivert Remote Control-grensesnitt kan du forkaste katalogcachen målrettet:

```bash
rclone rc vfs/forget
```

Dette er nyttig for en manuell test, men erstatter ikke en egnet driftsstrategi.

## Sett cachen under press

En nesten tom cache er det enkleste tilfellet. Sett `--vfs-cache-max-size` bevisst lavt i en andre testrunde, og les mer data enn det er plass til.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

De to størrelsene kan avvike mye fra hverandre. I full-modus bruker Rclone sparse filer: En fil viser hele sin logiske størrelse, selv om bare de leste områdene bruker lokal plass.

Cachegrensen er dessuten myk. Rclone kontrollerer den i intervallet `--vfs-cache-poll-interval`, og åpne filer kan ikke fjernes. Cachen kan derfor kortvarig overskride grensen. Etter at filene er lukket og neste oppryddingsrunde er kjørt, bør den likevel reduseres igjen.

Loggfør toppverdi, verdi etter opprydding og tiden dette tar. Slik kan nødvendig lokal lagringsplass dimensjoneres fornuftig.

## Simuler to forskjellige feil

En utilgjengelig sky og en krasjet Rclone-prosess er to forskjellige feil:

| Feil | Hva du kontrollerer |
|---|---|
| Backend eller nettverk er utilgjengelig, Rclone fortsetter å kjøre | Atferd ved nye forsøk, tidsavbrudd og allerede mellomlagrede filer |
| Rclone-prosessen avsluttes | Atferd for FUSE-mounten og gjenoppretting av mountpunktet |

Simuler begge kun i testmiljøet. Du kan hardt avslutte en Rclone-container for det andre tilfellet:

```bash
docker kill --signal KILL <rclone-container>
```

Kontroller applikasjonen under feilen, ikke bare mountpunktet:

- Hvilke funksjoner er fortsatt tilgjengelige?
- Hvor lenge venter en tilgang før det vises en feil?
- Er allerede fullstendig mellomlagrede filer fortsatt tilgjengelige?
- Stopper applikasjonen nye skriveoperasjoner?
- Oppstår det en forståelig feilmelding eller bare en hengende prosess?
- Utløses overvåkingen din?

En skrivetjeneste må ikke ubemerket skrive til den underliggende lokale katalogen når mounten mangler. Etter at mounten kommer tilbake, vil disse filene bli skjult. En enkel beskyttelse før hver skrivejobb er:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

Etter omstart av Rclone kontrollerer du mounten på verten og fra hver konsumerende container. En gjenoppbygd mount når en allerede kjørende container bare med riktig mount-propagation. For Docker kreves vanligvis `rslave` på den konsumerende siden. Detaljene står i artikkelen [Drift Rclone-mounter pålitelig i Docker](/blog/rclone-mount-in-docker-container).

## Et konkret eksempel fra Paperless-ngx

For min Paperless-test opprettet jeg 40 PDF-er på til sammen 13,9 MB. Et dokument som ikke tidligere var åpnet, tok rundt 1,8 sekunder, mens en umiddelbart gjentatt tilgang tok 19 til 24 millisekunder. En VFS-cache begrenset til 4 MB steg kortvarig til 12,7 MiB og ble ryddet igjen ved neste kjøring.

Mens remote-en ikke var tilgjengelig, fungerte dokumentlisten, fulltekstsøk og forhåndsvisningsbilder fortsatt fordi disse dataene lå lokalt. Bare originalen kunne ikke åpnes. Etter at mounten var bygget opp igjen, kunne den kjørende Paperless-containeren få tilgang til filene på nytt uten å måtte startes på nytt.

Disse tallene er ingen benchmark for Rclone eller Proton Drive. Atferden er det interessante: Hot Storage forble lokalt tilgjengelig, kalde lesinger var langsommere, men forutsigbare, og tjenesten gjenopprettet seg etter feilen.

## Hva testprotokollen bør inneholde

Et resultat som senere kan etterprøves, inneholder minst:

- Rclone-versjon og brukt backend
- operativsystem, FUSE-variant og filsystem for cachekatalogen
- fullstendig mount-kommando uten tilgangsdata
- antall, størrelsesfordeling og struktur for testfilene
- verdier for kalde og varme lesinger for flere filer
- skrivetid frem til synlighet i remote-en
- toppverdi for cache og varighet på oppryddingen
- resultat av `rclone check --download`
- atferd ved backend-feil og avsluttet Rclone-prosess
- gjenopprettingstid fra applikasjonens perspektiv
- nye forsøk, tidsavbrudd, begrensninger og autentiseringsfeil fra loggen

Definer en grenseverdi for hvert punkt på forhånd. Da ender testen med en beslutning og ikke bare med en samling interessante tall.

## Når oppsettet er klart

En Cold Storage-mount er klar til bruk når du kan svare ja på disse spørsmålene:

- Er kalde lesinger raske nok for den tiltenkte tjenesten?
- Akselererer cachen gjentatte tilganger som forventet?
- Forblir det lokale plassbehovet kontrollerbart også under belastning?
- Stemmer alle filer etter en fullstendig nedlasting?
- Fungerer alle nødvendige filoperasjoner med den valgte backend-en?
- Oppfører applikasjonen seg kontrollert ved skyutfall?
- Stanses skriveoperasjoner trygt når mounten mangler?
- Når en gjenoppbygd mount alle kjørende konsumenter?
- Viser overvåkingen feilen før en bruker rapporterer den?

Dersom ett svar mangler, vet du i det minste nøyaktig hva du må jobbe videre med. Det er langt mer nyttig enn en mount som så bra ut ved første `ls` og først viser begrensningene sine i drift.

## Kilder

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproduserbare testfiler og katalogstrukturer med konfigurerbare størrelser.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachemoduser, tilbakeskriving, sparse filer, cachegrenser og katalogcache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): sammenligning av kilde og mål, inkludert fullstendig kontroll med `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): målrettet forkasting av VFS-katalogcachen med `vfs/forget`.
